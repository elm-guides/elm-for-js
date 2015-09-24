# elm-for-js
Community driven Elm guide for JS people

This guide will augment the official Elm docs as needed, for a smooth transitions for people coming from JavaScript.

## Getting started
1. Try the [Hello World](http://elm-lang.org/try) online
2. Install Elm via [npm](https://www.npmjs.com/package/elm) or the [installer](http://elm-lang.org/install).
3. Get Hello World [running locally](https://github.com/elm-guides/elm-for-js#getting-hello-world-running-locally)
4. Work through the [architecture examples](https://github.com/evancz/elm-architecture-tutorial/#the-elm-architecture). Type manually, look in `examples/` for help.

## Getting Hello World running locally
1. Install Elm
2. Create a new file `Main.elm` in a new folder.
3. `$ elm package install` to set up `elm-package.json` and `elm-stuff/` for deps (similar to node).
4. Paste the source from the [Hello World example](http://elm-lang.org/examples/hello-html) to your local `Main.elm` file.
5. `$ elm install package evancz/elm-html` to install the missing package dependency.
6. `$ elm-reactor` to get the dev server running at `http://0.0.0.0:8000/`.
7. Success! 
8. For a standalone file you can run `$ elm make Main.elm --output=main.html` (default is a JS file).

## Type Annotations

Writing them is optional, but highly encouraged. Type annotations improve code by helping you think about what the function should be doing, and serve as compiler-verified documentation. In addition, if you ever want to publish a third-party library, you need type annotations.

```elm
-- A variable of type Int.
answer : Int
answer = 42

-- An update function that takes 2 params (Action, Model) and returns a Model (last).
update : Action -> Model -> Model
-- could also be written as
update : (Action, Model) -> Model
```

In JavaScript, params are handled at the same time. In Elm, they are [curried](https://gist.github.com/jamischarles/3c22acd58e6d4ab26a41). For example:
```elm
-- Update is a function that will take an Action param, and return a function that will take a Model param. THAT fn will return a Model.
update : Action -> Model -> Model
```


If you see mistakes, or something you want covered, open an issue or a PR.

## TODO:
Add sections for
- [x] Package management (compare to node_modules, npm)
- [x] Type Annotations
- [ ] Data types (try not to duplicate official docs)
- [ ] Basic Syntax
- [ ] Program structure and flow
