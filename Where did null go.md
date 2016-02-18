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
`undefined` to indicate to us that the item we were looking for is not present
in the array. This type of API, and the presence of values like `null` and
`undefined` in our programming language put the idea of emptiness on an
equivalent level to the concept of the value of a variable. In dynamic languages
like JavaScript this is a trivial outcome of the dynamic type system. In typed
languages like C# the concept of emptiness requires additional type information
to denote, but the same rules apply: a variable of a type, provided it is
allowed to be `null`, can be `null` at any time.

This requires the us as programmers to perform checks when we expect a value
might be `null` or `undefined`, and - because humans are not perfect - a major
category of bugs manifests itself as the Null Reference Error when we forget to
check or incorrectly expect that a value will never be null.

## `null` is not allowed in Elm

In Elm, values have types and those types are absolutely static. If a function
expects an `Int` argument, the program will only compile if that function is
only called with `Int` values. This prevents us from calling that function with
`String` values, etc., but it also precludes the situation explained earlier
where the value might be `null` or `undefined`. Not are `null` and `undefined`
not included as a part of Elm, they wouldn't work regardless because `undefined`
and `null` are not of type `Int` or any other type. This is a major contributor
to the reliability of Elm applications.

## `Maybe` arrises from these properties of Elm

This rigid static nature of types in Elm makes our programs very reliable and
eliminates an entire class of bugs, but we still need to be able to represent
lack of value! Consider again the scenario where we want to attempt to find
an element in an Elm `List Int`, as opposed to the JavaScript array. We still
may not find the thing we were looking for, and since Elm types are static we
need a single type to represent the possible absence of the `Int` we are trying
to find. `Maybe` is that type.

In the language nomenclature, `Maybe` is a "type constructor", meaning its full
signature is `Maybe a`. Here, `a` stands in for any other type and indicates to
us that `Maybe` is really a container for other types that serves to add some
additional meaning to a value: whether or not the value we want is present.
Additionally, `Maybe` is a union type, and in fact its full definition is as
simple as:

```elm
type Maybe a = Just a | Nothing
```

Now we know what `Maybe` looks like in Elm, but it may not yet be clear what it
is for or why we need it.

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

The base case of our recursion is that where there is nothing left in the list.
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
sense, and instead of reflecting that fact at runtime through a construct like
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
via pattern matching, making our code reliable, robust and complete by default.

Note also that in the above example we simply passed through for the `Nothing`
case. This particular situation is a special case of matching on `Maybe` that is
simplified for us by `Maybe.map` in the core `Maybe` module. We could rewrite
the code as

```elm
shouldBeSeven : Maybe Int
shouldBeSeven =
  Maybe.map (\value -> value + 1) foundSix
```
