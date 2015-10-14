# Scope in Elm

In Elm as in JavaScript, *scope* refers to what is defined, and where. Scope in Elm is often simpler because ordinary
values do not change over time. Understanding scope is helpful because it allows you to figure out where values and
types are coming from when you read someone else's code, including examples. It also gives us a tour of the language.

Not counting syntax (e.g. `if`, `->`, and records), pretty much everything in Elm is either a literal, something you imported, or
something you defined.

## Literals
These are pretty simple, and most are identical or very similar to JS. The types of literals are built into the
language.
```elm
True : Bool

42 : Int
6.28 : Float

"hello" : String
'x' : Char

["welcome", "to", "elm"] : List String
```

## Imports
Elm's core library is divided into many modules, and any third-party library you are using will also be broken up into
modules. The most common way to import a module is also the simplest:

```elm
import Dict
```

This gives you access to everything in the [Dict
library](http://package.elm-lang.org/packages/elm-lang/core/latest/Dict) by prefacing it with the module name and a dot.
So if you see `Dict.insert`, this is where it comes from.

For values, this is preferred because names are not unique across modules. In fact, they are deliberately consistent.
For example, the Array, Set, and Dict modules all expose an `empty` value, so the module names help you tell them apart.

For types, things aren't as nice. It's extremely common for a module to export a type of the same name as the module
itself. If you don't want to keep talking about `Dict.Dict` in your type annotations, use

```elm
import Dict exposing (Dict)
```

The `Dict` in parentheses refers to the type, not the module. All the module-scoped values like `Dict.insert` are still
available. You can expose as many values and types from a module by separating them with commas inside the parentheses.
You can find more details on the syntax page, but this practice in general is discouraged. (This is why the language
forces you to type the long `exposing` keyword.)

Elm also imports some values and types by default. The full list is
[here](http://package.elm-lang.org/packages/elm-lang/core/2.1.0/#default-imports), but the most important thing to know
is that all of [Basics](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics) is imported exposed. The List,
Maybe, and Signal modules and types are also available to you without an explicit import.

## Top-Level Definitions
The following code is valid Elm and JS:

```elm
answer = 42
```

In JS, `answer` is a global variable, but best practice is to use `var` and function scope so that it's no longer
global. Elm is immutable, so while it's still global, it's not longer a variable. It can't vary. Therefore, having it
global (at least to the module) is harmless.

In JavaScript, functions can either be declared or assigned to variables.

```javascript
function add(a, b){ return a + b }
var add = function(a, b){ return a + b }
```

These two lines of JS do subtly different things, thanks to hoisting, and in both cases the definition of `add` is
mutable. In some ways, the following two snippets of Elm code also preserve the distinction between declaring a named
function, and assigning an anonymous function to a named variable. However, they behave identically.

```elm
add a b = a + b
add = \a b -> a + b
```

Note that `\arg1 arg2 -> expression` is Elm's syntax for anonymous functions. The backslash is traditionally pronounced
*lambda*, after the Greek letter used by programming language theorists, but you're welcome to say *function* if that
helps you.

You can also define types at the top level, like `type alias Model = Int`.

## Local Definitions

The most common form of a local definition is a function argument. Exactly like JavaScript, any argument is visible from
anywhere inside the function.

The other form of local definitions are created using a `let... in...` statement. In this example, some values are
function arguments, some are defined in the `let`, and some (the math operators) are imported automatically from Basics.

```elm
distanceFrom (originX, originY) (x, y) =
    let dx = x - originX
        dy = y - originY
    in sqrt (dx^2 + dy^2)
```

After the `let`, you can place as many definitions as you like, just like at the top level. They can be fixed values or
functions, and you can even write type annotations, although you can't define new types.

The expression after the `in`, where all the definitions are in scope, is what the entire `let` expression becomes.
Actually, the definitions are in scope even as you write more definitions. Here's a somewhat contrived example.

```elm
radToDeg rad =
    let piInDegrees = 180
        conversionFactor = piInDegrees/pi
    in conversionFactor * rad
```

Be aware that if you define the same name multiple times, the innermost definition is used. Usually you should just
avoid the issue entirely by using unique names.

```elm
foo = 0

silly foo =
  let foo = 12
  in foo

silly 5 == 12
```
