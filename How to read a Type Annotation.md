Here's the basics:

: means "has type".

(X, Y) is the type of tuples with first element X, second element Y. This expands to arbitrarily many elements, i.e. (Int, Float, Int) is a 3-tuple with first element Int, second Float, third Int.

##Functions

-> is for function types. So we have something like:
`String.length : String -> Int`

This means that this is a function that takes a String as an argument, and returns an Integer.

Things get interesting with multiple arrows. Something like this:
`update : Action -> Model -> Model`

In your head, read this as something that takes an an Action and a Model as arguments (in that order), and returns a Model.

In reality, what's happening here is called currying: you can give a function only some of its arguments, and get a function as a result.
So if we have
`update someAction`
then this gives us a function of type
`Model -> Model`
There are implied brackets in the function notation, so we could also write:
`update : Action -> (Model -> Model)`

You probably don't need to worry about Currying too much at first. Just think of the last thing behind an arrow as the return value, and the others as the arguments to your function.

##Higher Order Functions

Now, things get interesting with higher-order stuff.

We can have arguments which are functions. Look at a specialized version of the map function, which takes a function, and applies it to every element of a list of integers, returning a new list of Floats as a result.
specialMap: (Int -> Float) -> List Int -> List Float

The first argument of this function needs to be a function, that takes an Int as a parameter and returns a Float. Here, the brackets DO matter. This is different than:
Int -> Float -> List Int -> List Float
which is a function that takes an Int, a Float, and a list of Int, and returns a float.

##Type Variables

If you look at the LIst library, this isn't actually how List.map is defined. Instead, it has lowercase type names, which are type variables:
List.map : (a -> b) -> List a -> List b

This means that the function works for any types A and B, as long as we've fixed their values. So we could give it an (Int -> Float) and a List Int, or we could give a (String -> Action) and a List String, etc. In other words, we can substitute specific types in for the variables a and b, and the function will still work.

This lets us write generic code. List.map then can traverse a list and apply a function to it, without knowing what's in the list. Only the function applied to each element needs to know what type those elements are.

##Type Constructors

We've already seen "List." Here, we always saw it beside another type or variable, like List Int or List a.

This is because List isn't a type, it's a type constructor. When you give it a type as an argument, you get a type.

So "List Int" is a type, and "List" is a type constructor.

That's what 
`Signal.Address Action`
is. Address is a type constructor, and Action makes it a type. So this is saying that it's an Address which you can give values of type Action.
