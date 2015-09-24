# elm-for-js
Community driven Elm guide for JS people

This guide will augment the official Elm docs as needed, for a smooth transitions for people coming from JavaScript.

## Type Annotations

Writing them is optional, but highly encouraged. Type annotations improves code by helping you think about what the function should be doing, and serve as compiler-verified documentation. In addition, if you ever want to publish a third-party library, you need type annotations.

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
init count = count
```


If you see mistakes, or something you want covered, open an issue or a PR.

## TODO:
Add sections for
- [ ] Package management (compare to node_modules, npm)
- [ ] Type Annotations
- [ ] Data types (try not to duplicate official docs)
- [ ] Basic Syntax
- [ ] Program structure and flow
