Destructuring(or pattern matching) is a way used to extract data from a data structure(tuple, list, record) that mirros the construction. Compare to [other](http://yang-wei.github.io/blog/2016/01/15/javascript-destructuring-assignment-and-spread-operator/) [languages](https://gist.github.com/john2x/e1dca953548bfdfb9844), Elm support much less destructuring but let's see what it got !

## Tuple

```elm
myTuple = ("A", "B", "C")
myNestedTuple = ("A", "B", "C", ("X", "Y", "Z"))

let
  (a,b,c) = myTuple
in 
  a ++ b ++ c
-- "ABC" : String

let
  (a,b,c,(x,y,z)) = myNestedTuple
in
  a ++ b ++ c ++ x ++ y ++ z
-- "ABCXYZ" : String

```

Make sure to match every tuple(no more no less) or you will get an error like:
```elm
let
  (a,b) = myTuple
in
  a ++ b
-- TYPE MISMATCH :(
```

In Elm community, the underscore `_` is commonly used to bind to unused element.

```elm
let
  (a,b,_) = myTuple
in 
  a ++ b
-- "AB" : String
```

It's also more elegant to decrale some constant of your app using destructuring.
```elm
-- with no destructuring
width = 200
height = 100

-- with destrcuturing
(width, height) = (200, 100)

```

Thanks to @robertjlooby, I learned that we can match exact value of comparable.
This is useful when you want to explicitly renaming the variable in your branches of `case .. of`.

```elm
isOrdered : (String, String, String) -> String
isOrdered tuple =
 case tuple of
  ("A","B","C") as orderedTuple ->
    toString orderedTuple ++ " is an ordered tuple."
    
  (_,_,_) as unorderedTuple ->
    toString unorderedTuple ++ " is an unordered tuple."


isOrdered myTuple
-- "(\"A\",\"B\",\"C\") is an ordered tuple."

isOrdered ("B", "C", "A")
-- "(\"B\",\"C\",\"A\") is an unordered tuple."
```

> Exact values of comparables can be used to match when destructuring (also works with String, Char, etc. and any Tuple/List/union type built up of them) - @robertjlooby 

## List

Compare to tuple, List almost do not support destructuring. One of the case is used to find the first element of a list by utilizing the cons operator, ie `::`w

```elm
myList = ["a", "b", "c"]

first list =
  case list of
    f::_ -> Just f
    [] -> Nothing

first myList
-- Just "a"
```

This is much more cleaner than using `List.head` but at the same time increase codebase complexity. By stacking up the `::` operator, we can also use it to match second or other value.

```elm
listDescription : List String -> String
listDescription list =
 case list of
    [] -> "Nothing here !"
    [_] -> "This list has one element"
    [a,b] -> "Wow we have 2 elements: " ++ a ++ " and " ++ b
    a::b::_ -> "A huge list !, The first 2 are: " ++ a ++ " and " ++ b
```

## Record

```elm
myRecord = { x = 3, y = 4 }

sum record =
  let
    {x,y} = record
  in
    x + y

sum myRecord
-- 7
```

Or more cleaner:
```elm
sum {x,y} =
  x + y
```

Notice that the variable declared on the left side must match the key of record:

```elm
sum {a,b} =
  a + b

sum myRecord
-- The argument to function `sum` is causing a mismatch.
```

As long as our variable match one of the key of record, we can ignore other.

```elm
onlyX {x} =
  x

onlyX myRecord
-- 3 : number
```

## Union Type

Again, thanks to @robertjlooby, we can even destruct the arguments of union type.

```elm
type MyThing
  = AString String
  | AnInt Int
  | ATuple (String, Int)

unionFn : MyThing -> String
unionFn thing =
  case thing of
    AString s -> "It was a string: " ++ s
    AnInt i -> "It was an int: " ++ toString i
    ATuple (s, i) -> "It was a string and an int: " ++ s ++ " and " ++ toString i
```

I don't think Elm support destructuring in nested record (I tried) because Elm encourages [sparse record](http://package.elm-lang.org/packages/evancz/focus/2.0.0/Focus)
