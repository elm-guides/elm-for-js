# Mailboxes, Messages, and Addresses

*Note: This essay assumes basic familiarity with Elm.*

In Elm, signals always have a data source associated with them. `Window.dimensions` is exactly what you think it is, and you can't send your own events on it. You can derive your own signals from these primitives using `map`, `filter`, and `merge`, but the timing of events is beyond your control.

This becomes a problem when you try to add UI elements. We want to be able to add checkboxes and dropdown menus, and to receive the current state of these elements as a signal. So how do we do that?

## The Bad Old Days

An earlier version of Elm offered this approach:

```elm
checkbox : Bool -> (Element, Signal Bool)
```

Given an initial state, this function created the checkbox DOM element and the signal of boolean state. The element needed to be placed into the page manually. (It wasn't a signal of elements because it kept the same size whether checked or unchecked.) Then the programmer would use the signal of booleans somewhere in the program. If you knew how many checkboxes you need, and it's more than one, you'd have to do tedious mapping and merging to get a single signal that told you _which_ checkbox got checked.

Do you see the problem yet? What happens if you have a dynamic number of checkboxes? Maybe you're designing a music player, and you have one checkbox per song but an unknown or variable number of songs? If we try to use `Signal.map` on the old checkbox function, we get a value of type `Signal (Element, Signal Bool)`. This is gross enough already, but the signal of booleans becomes unusable because Elm forbids signals of signals (for complex but good technical reasons).

The root of the problem is that we're creating one signal for each element. The key idea behind a mailbox is that it's one signal that can receive updates for many elements. That means that a mailbox's signal is _not_ constrained to only one source of data provided by the runtime, the way `Window.dimensions` is. But, creating an event on a signal is an effect, and Elm functions don't have effects (other than their return value). So how do we make this work?

## Enter Mailboxes

Here's the definition of a mailbox:

```elm
type alias Mailbox a =
    { signal : Signal a
    , address : Address a
    }
```

So a mailbox is a record with a perfectly ordinary signal, and this new Address thing, which we'll come to in a moment. But first, how do you make a mailbox? Using a function in the `Signal` library:

```elm
mailbox : a -> Mailbox a
```

Okay, so this just takes the initial value of the new signal and returns the new mailbox. A few things to note: first, don't use `Signal.map` with this function. That would give you a signal of a record containing a signal, which puts us back into the forbidden world of signals of signals.

Second, you'll often see type annotations like `myMailbox : Signal.Mailbox MyType`. The period is critically important: it's not a signal of mailboxes, it's just the `Mailbox` type from the `Signal` module. You can get around this with `import Signal exposing (Mailbox)`.

Third, `mailbox` is one of the few impure functions in Elm. By "pure", we mean that if you call a function with the same arguments, you get the same result. That's true of almost every function in Elm (and _not_ true of functions in other languages that look at global mutable state.) But here, we're cheating: if you call `mailbox myThing` twice, you get two different mailboxes.

What this means in practice is that you have a specific and constant number of mailboxes in your program. You don't make more dynamically and you typically don't make one inside a function. If you set things up like I'm going to show you, you'll only have _one_ mailbox in your program, or at least only one per module.

## Things to do with an Address

Okay, back to this address thing that we get when we make a mailbox. If you want to know what an address _is_, well, it's an opaque handle that identifies which signal a value should be sent to. There are three ways to use an address to send a value on its signal.

The first is to make a message. A message is like an envelope, sealed up with a letter and an address written on the front, but not yet sent. You can make a message with this function (again in the `Signal` library):

```elm
message : Address a -> a -> Message
```

So yeah, an address, the event to send, and you get a message. (Interestingly, it's just a `Message`, not a `Message a`. We know that every message has a match between the address and the event, and we don't need to know that type to actually send it.) The neat thing about a message is that since it only _describes_ sending a value on a signal, rather than actually sending it, you can send a message as many times as you like. The reason you'd make a message however, is to use it with some other API. Like this function from `Graphics.Input`:

```elm
button : Message -> String -> Element
```

We provide the button text and a message to send whenever the button is clicked. The actual sending of the message, the effect, is handled by the runtime.

It's pretty common to pass along not a message but a function to create one. For example, here's how we make checkboxes in Elm today:

```elm
checkbox : (Bool -> Message) -> Bool -> Element
```

The simplest way to get a `Bool -> Message` function is to have a mailbox of booleans, and then do `Signal.message myMailbox.address`. By partially applying `Signal.message`, we're left with a function of exactly the type we want. Usually you'll want your mailbox to track more than just one boolean, and we'll come back to that.

The second way you can use an address is just pass it along, as with these functions from [elm-html](http://package.elm-lang.org/packages/evancz/elm-html/4.0.1/Html-Events):

```elm
onSubmit : Address a -> a -> Attribute
onKeyUp : Address a -> (Int -> a) -> Attribute
```

If you look at these types, and then the type of `Signal.message`, you'll see that they're very similar. Essentially, the library is calling `message` for you, and you provide it the address and the value (or function to make a value).

## Taken to Task

The third and final way you can use an address is to turn it into a task:

```elm
send : Address a -> a -> Task x ()
```

Notice that this function takes the same arguments as `message` but instead returns a [Task](http://package.elm-lang.org/packages/elm-lang/core/latest/Task). A task is a much more general way to manage effects. A message can only send a value to a signal, but a task can send HTTP requests, interact with local storage, access the browser history, and more. The reason we have messages is to guarantee that clicking a button won't fire off an HTTP request (at least, not directly).

Like a message, a task doesn't do anything until run. Unlike a message, you can explicitly run a task yourself by sending it out a port:

```elm
myMailbox = Signal.mailbox 0
myTask = Signal.send myMailbox.address 1

port example : Task x ()
port example = myTask
```

This is a signal that starts as `0` and then quickly becomes `1`. A more realistic example would send an HTTP request as soon as the program starts, [and then](http://package.elm-lang.org/packages/elm-lang/core/2.1.0/Task#andThen) use the returned value to make another task with `Signal.send` to pipe it back into the program.

You can also send signals of tasks out of ports, for example, sending a chat room message to the server (which can happen any number of times).

## Talk to This Guy
Now that we understand the basics of mailboxes, there's one more function to cover. Here it is (with helpful type variables instead of `a` and `b`):

```elm
forwardTo : Address general -> (specific -> general) -> Address specific
```

Okay, looks like a typical mapping function, except–wait a moment–the types are flipped! We can go from specific to general, but turn a general address into a specific one? _What the WHAT??_

What's going on here is that the new address doesn't have a signal associated with it. Instead, whenever anybody tries to use it like a "real" address, it applies the function to turn the specific value into a general value, and then sends that along to the general address.

If that doesn't make sense yet, it's probably best to see an example.

## Taking Action
We're going to be writing a very simple spaceship simulator. So simple, in fact, that all we can do is decide whether to direct power to the forward or rear deflector shields, using checkboxes. (Blame budget cuts to NASA.)

Using the classic [Elm Architecture](https://github.com/evancz/elm-architecture-tutorial), we define our action and model types, as well as the initial model (I use the convention `model0` for this, like the subscript zero in physics equations):

```elm
type Side = Forward | Rear
type Action = NoOp | Shield Side Bool

type alias Model = {forward : Bool, rear : Bool}

fwd0 = True
rear0 = False
model0 : Model
model0 = Model fwd0 rear0
```

In case you haven't seen it before, the last line uses record constructor syntax, which basically makes a `Model` by passing in the values of the record, in order. Moving on to the mailbox:

```elm
myMailbox = Signal.mailbox NoOp

cbForward =
  checkbox
  (\b -> Signal.message myMailbox.address (Shield Forward b))
  fwd0

cbRear =
  checkbox
  (Signal.message (Signal.forwardTo myMailbox.address (Shield Rear)))
  rear0
```

We define the mailbox using the `NoOp` (no operation) tag, and then our two checkboxes. Remember that the first argument to `checkbox` is `Bool -> Message`. In the first example, I use an anonymous function to convert to our action type directly. In the second example, I use `forwardTo` to create a new address that accepts a boolean. `Shield Rear` is a partial application of the `Shield` tag; it's expecting a Boolean and produces an `Action`, which is exactly the specific-to-general function we want. Anyway, both approaches are viable in this case, but `forwardTo` becomes more helpful as you grow larger and more modular applications.

Finally we define our step function and tie everything together with a `foldp` to keep the model updated.

```elm
step : Action -> Model -> Model
step a m =
  case a of
    NoOp -> m
    Shield Forward b -> {m| forward <- b}
    Shield Rear b    -> {m| rear <- b}

model : Signal Model
model = Signal.foldp step model0 myMailbox.signal
```

Both of these are fairly obvious (provided you're familiar with the [record update syntax](http://elm-lang.org/docs/syntax#records)), and there seems to be a lot of clunky boilerplate throughout. But it gets the job done.

Actually putting the checkboxes onscreen is fairly simple and you can find the entire program in Appendix I.

## Transformations
Action types are well and good, but there's another way to construct your program. It's not always better, and it's a little harder to understand, but it also can be more concise and explicit. The basic idea is that rather than having a mailbox of action types, you have a mailbox _of functions_. Specifically, of transformations:

```elm
type alias Model = {forward : Bool, rear : Bool}  -- same as before
type alias Transformation = Model -> Model

myMailbox : Signal.Mailbox Transformation
myMailbox = Signal.mailbox identity
```

The first piece of good news is that we no longer need a `NoOp` tag, or even an explicit action type at all. We just use the `identity` function to initialize our mailbox.

Creating the initial model is the same as before, but the checkboxes are defined very differently:

```elm
cbForward =
  checkbox
  (\b -> Signal.message myMailbox.address (\m -> {m| forward <- b}))
  fwd0

cbRear =
  checkbox
  (\b -> Signal.message myMailbox.address (\m -> {m| rear <- b}))
  rear0
```

It's like we squished the step function into the message! Each checkbox says how it wants to manipulate the model directly. This is less modular than an explicit action type, but it's still reasonable because it's a record update. You can add more fields to `Model` without breaking this code.

Finally, all that's left of the step function is just function application.

```elm
model : Signal Model
model = Signal.foldp (<|) model0 myMailbox.signal
```

So, we've sacrificed a little bit of modularity and intuition for a dramatic reduction in boilerplate. We also put the UI controls right next to the change to the model that they enact. The full code is in Appendix II.

## Handling Dynamic Data

Earlier, I mentioned that if you were managing many songs, there was no way for the old system to give you one checkbox per song. To hint at how this would work with mailboxes, I'll define an action type and sketch out the model:

```elm
type Action = Check SongID Bool | NoOp | ...
myMailbox = Signal.mailbox NoOp

-- implementation is left for the reader
getSongs : Model -> List SongID
getStatus : Model -> SongID -> Bool
model : Signal Model
```

Then I'll write a function that makes a checkbox for a given song and current checked status. It's this function's responsibility to wrangle `Graphics.Input` into submission.

```elm
mkCheckbox : SongID -> Bool -> Element
mkCheckbox id currentStatus =
  checkbox
  (\b -> Signal.message myMailbox.address (Check id b))
  currentStatus
```

Next, we extract that data we need from the model and make one checkbox per list item. Then we map that function over the signal of the model.

```elm
mkCheckboxes : Model -> List Element
mkCheckboxes model =
  let songs = getSongs model
      statuses = List.map (getStatus model) songs
  in List.map2 mkCheckbox songs statuses

checkboxes : Signal (List Element)
checkboxes = Signal.map mkCheckboxes model
```

This gives me a list of checkboxes that I can put into the view, based on a signal of a list of songs, which varies in both space and time.

The takeaway of this example is that every time the model updates, `checkboxes` creates a bunch of new checkboxes that will send messages to the same address that has always been there. The old checkboxes are taken offscreen and eventually garbage collected. As the program runs, more and more elements are created that can send events to the mailbox, and that's perfectly fine.

When implementing, you need to watch for two things. First, you have to feed the checkboxes their own state from the model (that's the purpose of `statuses`). That's because every time you check a box, the model is updated, which causes a new set of checkboxes to be made, hopefully with the newly updated state. Second, you need to ensure that you can uniquely identify which datum the checkbox refers to (the `SongID`). If you have many actions per datum that are controlled by checkboxes, you need to track that too (either as a parameter of `Check` or another `Action`).

## The Future?

So far, I've tried to remain objective in how mailboxes work and can be used. In this section, I'm going to talk briefly about a [proposal I made](https://gist.github.com/mgold/f3527359996fdf295843) which would slightly alter the library to further reduce boilerplate and (hopefully) make it more intuitive. Let me stress that the proposal has not been accepted and might not ever be incorporated into Elm.

The big change is that `myMailbox.address` would be replaced with a `dispatch : a -> Message` field. It's like `Signal.message` is called on the address for you. How would this change our checkboxes?

```elm
-- action type style
cbForward =
  checkbox
  ((Shield Forward) >> myMailbox.dispatch)
  fwd0

-- transformations style
cbForward =
  checkbox
  (\b -> myMailbox.dispatch (\m -> {m| forward <- b}))
  fwd0
```

The action type style benefits tremendously; we replace `forwardTo` with standard function composition. Surprisingly, the transformation style doesn't change much, though it does get a little shorter.

Even if the proposal is never implemented, I hope this example helps show what messages and addresses truly are underneath the types and functions.

Hopefully mailboxes make more sense to you now, but frequently even the best explanation can only do so much. If you're still struggling, study and extend the fully-working examples below. Thanks for reading.

--------------

## Appendix I
This is the code for the traditional "action type" architecture.

```elm
import Graphics.Input exposing (checkbox)
import Graphics.Element exposing (flow, down, right, show)

type Side = Forward | Rear
type Action = NoOp | Shield Side Bool

type alias Model = {forward : Bool, rear : Bool}

fwd0 = True
rear0 = False
model0 : Model
model0 = Model fwd0 rear0

myMailbox = Signal.mailbox NoOp

cbForward =
  checkbox
  (\b -> Signal.message myMailbox.address (Shield Forward b))
  fwd0

cbRear =
  checkbox
  (Signal.message (Signal.forwardTo myMailbox.address (Shield Rear)))
  rear0

step : Action -> Model -> Model
step a m =
  case a of
    NoOp -> m
    Shield Forward b -> {m| forward <- b}
    Shield Rear b    -> {m| rear <- b}

model : Signal Model
model = Signal.foldp step model0 myMailbox.signal

scene m =
  flow down
    [ flow right [cbForward, show "Forward"]
    , flow right [cbRear, show "Rear"]
    , show m
    ]

main = Signal.map scene model
```

## Appendix II
This is the code for the "transformations" (signal of functions) style of mailboxes.

```elm
import Graphics.Input exposing (checkbox)
import Graphics.Element exposing (flow, down, right, show)

type alias Model = {forward : Bool, rear : Bool}

fwd0 = True
rear0 = False
model0 : Model
model0 = Model fwd0 rear0

myMailbox = Signal.mailbox identity

cbForward =
  checkbox
  (\b -> Signal.message myMailbox.address (\m -> {m| forward <- b}))
  fwd0

cbRear =
  checkbox
  (\b -> Signal.message myMailbox.address (\m -> {m| rear <- b}))
  rear0

model : Signal Model
model = Signal.foldp (<|) model0 myMailbox.signal

scene m =
  flow down
    [ flow right [cbForward, show "Forward"]
    , flow right [cbRear, show "Rear"]
    , show m
    ]

main = Signal.map scene model
```
