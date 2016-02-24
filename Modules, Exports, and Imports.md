# Modules, Exports, and Imports

Elm's core libraries are organized into modules, as are any third-party packages you may use. For your own large
applications, you can define your own modules. However defined, you need to import from modules to get useful values and
types fro your program.

First we'll introduce the two types of Elm programs and how they affect how you use modules. Then we'll learn how
modules relate to files (very closely), and then how to define your own modules and control what they export. Next we'll
learn the right and wrong ways to import from a module. Then we'll see how exports can be a form of information hiding,
and when that is useful. Finally we'll learn a trick to organize tests or examples.

Along the way, we'll see both ways to structure code well, and ways to shoot yourself in the foot. We'll also learn more
about Elm's tooling and `elm-package.json`.

This shouldn't be the first guide on Elm you read, but if you've playing around for a few hours and have specific
questions about modules and the like, you should be fine. This guide aims to be a comprehensive reference to be read
straight through, but if you have questions about a specific piece of syntax you should do fine skimming for code
blocks.

## Packages and Applications

Elm programs fall into two categories. **Packages** (or **libraries**) are meant to be reused and are typically
published to the [package registry](http://package.elm-lang.org/). **Applications** are programs that can actually be
run and produce output. Applications may depend on packages, which may depend on other packages in turn; applications
are the root of this tree.

The distinction between package and application occurs in `elm-package.json`: A package will export at least one module
in the `"exposed-modules"` field. Packages are compiled with `elm make` and no additional arguments. This compiles the
modules listed, and subjects them to additional compile-time checks ensuring every exported value and type are
documented properly. The compiler does not generate an HTML or JS output file.

If you pass a file to `elm make`, then you are writing an application. That file (and therefore module) is known as
"Main". It's not necessary to call this file `Main.elm` or even declare a module definition in it at all -- but you
should for large applications.

However defined, only Main can define ports (a topic otherwise beyond the scope of this guide). The Main module must
define a `main` value, whose type is restricted to a handful of things that the compiler can display. Other modules
should avoid defining `main`.

Another difference between packages and applications is how they tolerate change. Elm's package manager enforces
[semantic versioning](http://semver.org/), discouraging breaking changes. Additionally anything that is exported must be
documented. Packages therefore want to hide their type definitions and helper functions, in order to present a stable,
understandable, and helpful interface. Applications are more tolerant of change; for example adding a field to a record
type alias or a tag to a union type. Only the application itself relies on these definitions, and the compiler will find
any discrepancies that changes introduce. Therefore, packages must tightly control what is exported, but applications
can afford to be more permissive.

## Modules and Files

Every file contains exactly one module. Filenames should be capitalized and match the name of the module they contain.
For example, `Foo.elm` should declare the module `Foo`. The compiler will not allow module names and file paths to
disagree.

If you only have one file, it's fine to keep it in the top level folder. Otherwise, you should create a `src` folder and
keep all source files in there. You'll need edit `elm-package.json` to read `"source-directories": [ "src" ],` (don't
forget the comma; `elm package` doesn't give a helpful error on invalid JSON).

Module names can contain dots, for example `Json.Encode`, which is in file `src/Json/Encode.elm`. Folders may be nested
as deep as you like, but don't overdo it. Sometimes a package will have a primary module `Foo` and a secondary module
`Foo.Bar`. This is just the coincidence of having a file named the same as a folder (except the `.elm` extension);
there's nothing "magic" about it.

## Declaring a Module and Its Exports

The simplest syntax to declare a module is

```elm
module MyModule where
```

This must be the first line of the file, and the module name must begin with a capital letter. Convention is to
capitalize only the first letter of acronyms (e.g. `Json` and `Http`).

The third-party tool `elm-format` will correct this definition to

```elm
module MyModule (..) where
```

in order to make explicit that you are exporting everything in this module. That is, the parenthesis enclose a list of
exported values, but the two dots are a shortcut indicating that everything is exported. "Everything" comprises all
defined top-level constants, functions, types alias, union types, and their tags.

If you want to limit what is exported, you must list everything that you do want to export (there is no way to
"blacklist" private items). A list of items is separated by commas, and may be broken up across many lines. The one
wrinkle is how union types are exported, but that will be addressed below.

```elm
module MyModule (MyType, myValue, myFunction) where
```

If `MyModule` defines `myPrivateFunction`, it cannot be imported by any other module, regardless of syntax used.

Although uncommon, you should avoid situations like this:

```elm
module Chance (Model, init) where

type Coin = Heads | Tails

type alias Model = { coin : Coin }

init : Model
init = Model Heads
```

The private `Coin` type is visible in the definition of the public `Model` type. Even worse, someone importing this code
can actually obtain a value of type `Coin` through `init`, even though that type doesn't exist to them. And worst of
all, the compiler will not catch this error. It *should*, but it doesn't. So be careful when defining types, and think
about what will and will not be exported.

## Imports
The most common way to import a module is also the simplest:

```elm
import Dict
```

This gives you access to everything in the [Dict
library](http://package.elm-lang.org/packages/elm-lang/core/latest/Dict) by prefacing it with the module name and a dot.
This is sometimes called a **qualified** import. So if you want to use `Dict.insert`, this import statement is sufficient.

Qualified imports are the preferred way to import values from other modules, because you can always tell where something
came from by the module name right in front of it. Conversely, it makes it easy to tell what is defined in *this* module.

The alternative to a qualified import is an **exposed** import, because of the keyword `exposing` (whose length should
serve as a deterrent to using it!). But it's usually okay if you list what you're exposing, for example:

```elm
import Dict exposing (Dict)
import Html exposing (div, span, h1, h2)
```

In the first case, the `Dict` in parentheses is a type, not the module. (It's very common for modules to define types of
the same name as the module itself.) It avoids the need to say `Dict.Dict` in type annotations. In the second case, the
functions being exposed have distinctive names, and are going to be used frequently.

In both cases, the module name is also available to be use qualified. So you can still use `Dict.insert` or `Html.text`.
When you have a list of exposed imports, the order doesn't matter, but typically the types are listed before the values.

The real reason exposed imports are considered an antipattern is because they can be combined with `(..)` to dump the
entire module into the current one. When there are multiple such imports, it becomes impossible to see where something
came from, and the odds of a name collision is much higher.

```elm
-- Antipatten!
import Html exposing (..)
```

Regardless, exposing happens when you import. This makes it different from exporting, which happens at export. Many
people and even Elm's tooling conflate the two, but if you're being technical they are distinct.

Infix operators must be imported exposed, for example as `import Json.Encode exposing (object2, (:=))`. The language
does not have syntax for qualified infix operators. The good news is that, with the exception of the previous example,
all infix ops in core are imported exposed by default.

Specifically, all of the arithmetic operators are in
[Basics](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics), which is imported exposed automatically.
(It's worth becoming familiar with everything in that module.) List cons, `(::)`, is imported along with the `List`
type, as are `Maybe` and its tags, `Just` and `Nothing`.

When importing a module, you can rename it like so:

```elm
import Json.Decode as Decode
import Graphics.Element as Elem exposing (Element, show)
```

As you can see, renaming and exposing can be used together. If you do though, `as Alternative` needs to come before
`exposing (the, functions)`.

Renaming is often used by "-extra" packages, which add extra functions to the core library. Typically they will be
namespaced according the library they extend, and then imported like this:

```elm
import Random
import Random.Extra as Random
```

Yes, you can rename an import to share a name with another module. In theory, you now no longer need to worry about
which functions `Random` exports, and which ones `Random.Extra` does (until you need to look up documentation…).
However, some of these libraries will re-export functions in the library they extend, with the same name. Then when you
try to use this function, the compiler will error because `Random.map` is ambiguous. There is no way to resolve this
error without renaming the import or updating the library.

When you're listing all of your imports, it's helpful to group them in a sensible order. Although you don't have to be
neurotic about it, try to put core libraries towards the top, and third-party packages towards the bottom. Modules from
the current package or application should be listed last.

## Exporting and Importing Union Types
Union types have a little bit of special treatment when it comes to imports and exports. Let's start with a simple union
type, which you might find in a counter demo.

`type Action = Increment | Decrement`

Remember that `Increment` and `Decrement` are called "tags". How would we export this type?

`module MyModule (Action) where` exports `Action` only. That is called an *opaque type*. Clients can see that `Action`
exists, but they can't see into it. This is frequently what you want, and we'll talk about it more in the next section.

`module MyModule (Action(..)) where` exports `Action` and all of its tags, namely `Increment` and `Decrement`. This form
can be useful sometimes if you can commit to keeping `Action` the same. A good example is a [nonempty
list](http://package.elm-lang.org/packages/mgold/elm-nonempty-list/latest/List-Nonempty#Nonempty); there will never be a
reason to change the type's definition so the tag can be safely exported.

`module MyModule (Action(Increment, Decrement)) where` exports `Action` and only the listed tags. In this case it's
listing all the tags explicitly. It's possible to only export *some* of the tags, but there is no reason to do this,
ever.

The trouble with exporting tags is not only that you may want to remove some, which will break any code that relies on
the ones being removed. Even adding tags will break code, because previously exhaustive pattern matches are no longer
exhaustive. If only some of the tags are exported, it's impossible to write a valid `case` statement.

Importing union types exposed follows the exact same syntax. For example, `import MyModule exposing (Action)` will
import only the type, while `import MyModule exposing (Action(..))` will import any exported tags as well. You can
also use exported tags qualified, like `MyModule.Increment`.

## Opaque Types

An **opaque type** is a union type where the type is exported but the tag(s) are not. Someone outside the module can see
that the type exists, pass it around, and store it in records. They only thing they *can't* do is look inside the type,
even if they know what the tags are named. Hence, it's opaque.

An example of an opaque type would be a 2D point. Creating a point would require either `x` and `y`, or `r` and `theta`.
Each value could be accessed individually from a given point. The point might actually store all four, knowing that
there's no way for someone to create a point that's inconsistent.

What this means is that opaque types are Elm's way of enforcing information hiding. They allow a package author to
define an interface of functions to create, update, and read useful values out of the opaque type. The implementation
can change completely, but as long as all functions on the type are updated to match, it's still considered a patch
change. This gives package writers flexibility when writing their libraries. It also lets them rely on invariants,
assured that the client hasn't meddled with values of the type in unexpected ways.

Opaque types are less useful in applications. If you're typing to simply pass information around, exporting record type
aliases is fine. If it makes sense to also define operations on these models, an opaque type might be a better fit.

If your union type contains many tags, you can export functions that wrap them. You can be selective about which ones
you export. Sometimes it can be tedious, but it does make a difference.

```elm
module Road (Action, addCar, redLight) where

import Car exposing (Car)

type Light =
  Red | Yellow | Green

type Action =
  AddCar Car | Light Light | SomethingPrivate

addCar : Car -> Action
addCar =
  AddCar

redLight : Action
redLight =
  Light Red
```

You cannot make an opaque type out of a type alias; those are either exported or not, just like values. But you can
create a union type to hide it. (This is more common when to opaque type represents a model rather than an action.)

```elm
module Person (Person, age) where

type Person =
  P { name : String, age : Int }

age : Person -> Int
age (P {age}) =
  age
```

First, we define the union type `Person` with one tag, `P`, which carries a record. (You can put the record definition
directly in the union type, or define an unexported alias.) Then we can export the `age` function which accesses the
record in a way not possible outside of this module. The definition uses two nifty language features. First, it pattern
matches on the `P` tag in the argument list. This is permitted when there is only one tag, because otherwise it's an
incomplete pattern match. Next, `{age}` destructures the record, assigning the local constant `age` to the value of the
record's `age` field. It's a much more concise way of writing this:

```elm
age person =
  case person of
    P record ->
      record.age
```

## Organizing Tests and Examples

When writing a package, it's often useful to write tests or examples that use it. But, you don't want these to be
included with the package, and certainly not in the compiled code that uses your package. Elm's tooling does not
(currently) have an equivalent of Node's dev-dependencies or Ruby on Rails' development environment. So here is a trick
to keep tests and their libraries separate from your package, while still being able to run the tests without
publishing.

Create a new folder, say `test`, and copy `elm-package.json` there. Edit it and make these changes:
  * Change the version to `0.0.1`. This isn't strictly necessary but it helps prevent the tests from being released as their own package.
  * Under `"source-directories"`, change `"src"` to be `"../src"`. (If it's just `"."`, make it `".."`). Then add `"."` to the list.
  * Change the `"exposed-modules"` to be the empty list, `[]`. Tests are an application, after all.

Notice that we've kept all of the dependencies of the package in place. Now you can install any test toolkits you like
into `test/elm-package.json` and they won't affect the main package. If the package's dependencies change, you will need
to manually sync them to `test/elm-package.json`.

You can now write your tests or examples and import modules in your package as if it was installed in
`elm-package.json`, but instead it's pulling from the parent folder with the original source. You just need to run `elm
package` or `elm reactor` from inside the `test` folder.
