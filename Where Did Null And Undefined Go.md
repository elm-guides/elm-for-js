# Where did `null` and `undefined` go?

In our applications it is often useful to be able to represent the possible
absence of a value. In JavaScript we have `undefined` to represent the result
of trying to access something that was never defined to begin with, and we have
`null` to assign to values that _might_ have a value but could also be empty.
These two are often conflated and used interchangeably both at the core language
level and in libraries, but both are used to communicate the lack of a value
where it is possible to find one.

For example, consider the function `Array.prototype.find` which is defined in
the ECMAscript 2015 spec:

```javascript
const array = [1, 2, 3, 4, 5];
const foundOne = array.find((x) => x === 1); // foundOne === 1
const foundSix = array.find((x) => x === 6); // foundSix === undefined
```

The return value of the `find` method in this scenario is either a `number`, or
`undefined` to indicate to us that the item we we are looking for is not present
in the array. This type of API, and the presence of values like `null` and
`undefined` in our programming language put the idea of emptiness on an
equivalent level to the concept of the value of a variable. In dynamic languages
like JavaScript this is a trivial outcome of the dynamic type system. In typed
languages like C# the concept of emptiness requires additional type information
to denote, but the same rules apply: a variable of a type, provided it is
allowed to be `null`, can be `null` at any time.

This requires us as programmers to perform checks when we expect a value
might be `null` or `undefined`.  Humans are not perfect, therefore a major
category of bugs manifests itself such as the Null Reference Error when we
forget to check or incorrectly expect that a value will never be null.

## `null` is not allowed in Elm

In Elm, values have types and those types are absolutely static. If a function
expects an `Int` argument, the program will only compile if that function is
only called with `Int` values. This prevents us from calling that function with
`String` values, etc., but it also precludes the situation explained earlier
where the value might be `null` or `undefined`. Not only are `null` and
`undefined` not included as a part of Elm, they wouldn't work regardless because
`undefined` and `null` are not of type `Int` or any other type.

## `Maybe` arises from these properties of Elm

Even though we no longer have a concept of `null` and `undefined`, we still need
to be able represent optional values. Consider again the scenario where we want
to attempt to find an element in an Elm `List Int`, as opposed to the JavaScript
array. We still may not find the thing we were looking for, and since Elm types
are static we need a single type to represent the possible absence of the `Int`
we are trying to find. `Maybe` is that type. Furthermore, because types are
static, a function which returns an `Int` rather than a `Maybe Int` will
_always_ return an `Int`, so there is never uncertainty or need for unnecessary
`null` checks. The `Maybe` type fully describes the presence of an optional
value.

## `Maybe` in Elm

In the language nomenclature, `Maybe` is a type constructor, meaning its full
signature is `Maybe a`. Here, `a` stands in for any other type and indicates to
us that `Maybe` is really a container for other types that serves to add some
additional meaning to a value: whether or not the value we want is present.
Additionally, `Maybe` is a union type, and in fact its full definition is as
simple as:

```elm
type Maybe a = Just a | Nothing
```

By defining this, we are establishing a type, `Maybe a`, and two possible _type
constructors_ for that type, `Just a` and `Nothing`. Invoking either type
constructor like a function or value will give us a `Maybe` value which is the
particular member that we used to construct. If we want to represent a non-empty
value of 5, we can invoke `Just 5` to get a `Maybe Int`. If we don't have a
value, we simply pass around `Nothing` since it has no arguments.

As a union type, `Maybe` is also a data structure in the same way as `List`.
`Maybe` is a container for a single element, and as a container we are able to
define a couple functions that are available for container structures like
`List` and `Task`.

#### `map`
```elm
map : (a -> b) -> Maybe a -> Maybe b`
```

Given a `Maybe a` value, transform its contained item from `a` to `b` in
the case that it is `Just a`, and pass through if it is `Nothing`.

#### `andThen`

```elm
andThen : Maybe a -> (a -> Maybe b) -> Maybe b
```

Given a `Maybe a`, chain a computation which produces a new `Maybe b` from the
contained element when the value is `Just a`, and pass through `Nothing` when it
is `Nothing`. This allows us to chain multiple `Maybe`-production computations
together without worrying about `null` checks along the way. If any of the
`Maybe` values in the chain are `Nothing` we will get `Nothing` back without
error.

`Maybe` also falls into a group of types where some possibilities are
represented and one of those possibilities is most often of primary interest. In
the case of `Maybe`, it's most common to be interested in working with the value
under the `Just` case. Types with this property often come with a `withDefault`
function for collapsing the `Maybe` using some acceptable default value.

#### `withDefault`

```elm
withDefault : a -> Maybe a -> a
```

Given a default value of type `a` and some `Maybe a`, return either the value
contained by the `Maybe` when it is `Just a` or the default value when it is
`Nothing`.

## Illustrating `Maybe` by example

Let's return once more to finding an item in a list. To build a basic function
to find an element in a `List Int`, we can start with
the following type signature:

```elm
find : Int -> List Int -> Maybe Int
```

We should read the type signature as "`find` is a function which accepts an
`Int` and a `List` of `Int`s and returns a `Maybe Int`". In the JavaScript
example, `find` returns either a number or `undefined` if the value isn't
in the array. In Elm, we must always return the same type from a function
regardless of the outcome of the logic. We can use the two members of `Maybe`,
`Just` and `Nothing`, to handle the two possibilities and still maintain type
consistency.

```elm
find : Int -> List Int -> Maybe Int
find valueToFind listToSearch =
  case listToSearch of
    currentItem :: remainingItems ->
      if currentItem == valueToFind then
        Just currentItem
      else
        find valueToFind remainingItems

    [] ->
      Nothing
```

We've defined `find` recursively, and used pattern matching to search the list.
The first case, `currentItem :: remainingItems`, is matched when there is at
least one value in the list, and makes available the item at the front of the
list, `currentItem`, and another list `remainingItems` which is everything else
in that list. If there was one item left, then `remainingItems` will be an empty
list. Now that we have access to the front item, we can check if it's equal to
the one we're trying to find. If it is, we can return `Just currentItem`, which
is of type `Maybe Int` and represents the case where we do have the item of
interest. If it isn't equal, we continue our search over the remaining items
recursively.

The base case of our recursion is that when there is nothing left in the list.
We've exhausted every possibility and found nothing that matches. In this case
we return the aptly named `Nothing`, which is also of type `Maybe Int` in this
context, but has no data associated with it because we've got no information to
return.

So far there's not a lot of difference between our function's return value and
one in JavaScript that returns a number or `undefined`. The semantics are
roughly the same, and the main difference has been the implementation details of
using a single type in Elm vs. multiple types in JavaScript to represent value
vs. lack of value. The real difference for Elm programs comes in the way that
`Maybe` is handled.

## Dealing with a `Maybe` value

In JavaScript the language imposes no expectation on how we invoke and use
values, whether they might be `undefined`, `null`, or some real value because
the language is dynamic. From the first example, we could continue on to use
`foundSix` as a number even though it is not, and introduce bugs into our
program.

```javascript
const shouldBeSeven = foundSix + 1; // shouldBeSeven is NaN
```

This case is easy to miss because it is the programmer's responsibility to
understand that the value might not be a number. Recall, though, that in Elm
`Maybe` is defined to be a union type. This means that in order to work with it
we must pattern match, and in doing so we must consciously acknowledge every
possible pattern that we can encounter. Further more, we can't attempt to
fraudulently add a `Maybe Int` to another `Int` because the compiler doesn't
know how to add those two types together. The operation doesn't make logical
sense, and instead of reflecting that fact at run-time through a construct like
`NaN`, the compiler simply refuses to compile the code.

To attempt to add 1 to the result of our search for 6, we must first check that
we found 6 at all and in doing so handle both the case where we were successful
and the case where we were not.

```elm
foundSix : Maybe Int
foundSix =
  find 6 [1, 2, 3, 4, 5]

shouldBeSeven : Maybe Int
shouldBeSeven =
  case foundSix of
    Just value ->
      value + 1
    Nothing ->
      Nothing
```

In the above example we need to match on `Just` in order to extract the value if
it is available. This leaves only one other possible pattern, `Nothing`, and we
just pass through another `Nothing`. Elm has raised the concept of emptiness
from something like `null` that could occur in any variable and brought it
outside the actual data type of the value we want to use. It adds that
information on top of the underlying value and forces us to deal with both cases
via pattern matching.

Note also that in the above example we simply passed through for the `Nothing`
case. In this case we are passing `Nothing` straight through and only interested
in transforming the value under `Just`. This case is handled for us more
succinctly by `Maybe.map`. We could rewrite the code as

```elm
shouldBeSeven : Maybe Int
shouldBeSeven =
  Maybe.map (\value -> value + 1) foundSix
```

## What about representing failed operations?

In JavaScript `null` and `undefined` are sometimes used to encode the fact that
an operation has or has not successfully completed. Consider the canonical
Node.js callback style:

```javascript
function callback(err, data) {
  if (err) {
    // ...handle error case
  } else {
    // ... error is null or undefined so we know everything is Ok
  }
}
```

The absence of an error indicates that there was no error. In Elm we could write
a similar function that might look like this:

```elm
callback : Maybe MyError -> Maybe MyData -> MyOutput
callback maybeError maybeData =
  case maybeError of
    Just error ->
      -- ...
    Nothing ->
      case maybeData of
        Just data ->
          -- ...
        Nothing ->
          -- ...
```

This kind of handling is extremely awkward in Elm, and writing this way discards
what we've already learned about using `Maybe` to represent emptiness.
Furthermore, there is no reason our function must accept exactly one `Just
MyError` or `Just MyData` and one `Nothing`, making this pattern unpredictable.
To deal with cases like this Elm's core libraries also provide the `Result` type
to allow us to represent the possibility of a failure to return something the
same way would use `Maybe` to represent having nothing to return. The definition
of `Result` is also familiar:

```elm
type Result a b = Err a | Ok b
```

We once again have two type constructors to represent two possibilities. On one
side we have the `Err` case and any data type we'd like to associate with error,
and on the other we have `Ok` and a data type that we are interested in
computing. We can then pattern match once to handle both cases and have access
to the information in either case. Like `Maybe`, we also know that if a function
doesn't return a `Result` then there is no possibility its computation will fail
and we can be confident in calling it without safeguards and error handling.
Let's reimagine the previous node-style callback example using `Result`:

```elm
callback : Result MyError MyData -> MyOutput
callback result =
  case result of
    Err error ->
      -- error is a value of type MyError
    Ok data ->
      -- data is a value of type MyData
```

`Result` is most commonly found in the core libraries when working with
`Json.Decode` functions, as parsing JSON is something that may fail at runtime
if the input is malformed. We can't detect that the input is malformed at
compile time, and Elm does not provide a concept of exceptions and `try/catch`
so we can use `Result` to model the output.

## Conclusions

Languages like JavaScript handle the concept of emptiness with low-level values
like `null` and `undefined`, and may also use those to represent failure and
success. These concepts are hard to represent in a statically typed language
without introducing a common class of bugs. Elm uses container types to bring
the concepts of emptiness and failure outside of values and add that information
to a type instead of interleaving it. Instead of having a value which might be a
number or `null`, we have `Just` a number or `Nothing`, and the compiler helps
us deal with that. Instead of representing failure and success by emptiness or
resorting to an exception system, we have either an `Err` with some information,
or some `Ok` data.
