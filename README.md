# Kotlin existential types proposal

## Useful sources

* Why existential types were removed in Scala 3: [GH issue](https://github.com/lampepfl/dotty/issues/4353)

* Formalization of Java wildcards using existential types:
  [small](https://ecs.wgtn.ac.nz/foswiki/pub/Main/TechnicalReportSeries/ECSTR09-04.pdf),
  [medium](https://scholar.google.co.nz/citations?view_op=view_citation&hl=en&user=bVyUIHwAAAAJ&citation_for_view=bVyUIHwAAAAJ:2osOgNQ5qMEC) (Recommended),
  [big](https://www.ncameron.org/papers/cameron_thesis.pdf).

## Advantages

The main advantages of the skolem types instead of unidentified projections are:

* More expressive type-safe code without an explicit generics' introduction.
  For example, we would be possible to express AVL-tree with type-level control of branches balance
  using this feature (with addition to GADT feature).

* More straightforward and precise implementation of bare types feature.
  With this feature,
  bare types may be expressed just like a syntax sugar over instantiation of all parameters with star-projection.
  As the projections became a skolem variable,
  it will be possible to track bounds from all sources in a more precise, stable and predictable way.

* Less error-prone type system.
  As skolem types will explicitly track their type bounds, it will help to
  * More straightforward tracking soundness of the system.
  * More ease and precise detection of the unreachable code and redundant branches

* TODO...

Implementation will be mostly invisible to the user. Exceptions:

* Error messages. It is quite hard and important to properly express type errors related to the skolem types.
  Possibly, we could do it in the same way as Scala.

* Possibly we could later add an ability to give a name for the unknown generic parameter with an underlying existential type for the code like this:
  ```kotlin
  val lst: MutableList<t> = foo()
  val fst: t = lst.first()
  lst.add(fst)
  ```

## Implementation

### Scala reference

TODO: Remove this section

In Scala, they are saying that it is implemented using path-dependent typing.
I do not see any reasons to do it in such a way (Except clarity in error messages).
AFAIK path is just a debug reference on the place where the skolem were bound.
It is not even used as ID as they are comparing skolem types by reference.

But generally, the behaviour of the features could be the same as in Scala.

### Current state (Capturing)

Limited existential types are highly related to the captured types.
To be precise, captured types are equivalent to the existential types with implicit packing-unpacking.
Let's review the moments of packing and unpacking that are used for now.

Currently, projections in the type of value are captured in the places of referencing this value.
For example, if we have function call, receiver and arguments will be captured.
But as capturing is performed in each referencing independently,
the resulting captured types of single immutable value will not be equal.
It leads to overly restrictive typing in several cases:

```kotlin
fun <T> shuffle(f: MutableList<T>, s: MutableList<T>)

fun foo() {
  val l: MutableList<*>
  shuffle(l, l) // Argument type mismatch: actual type is 'kotlin.collections.MutableList<CapturedType(*)>', but 'kotlin.collections.MutableList<T>' was expected.
}
```

or

```kotlin
fun foo() {
  val l: MutableList<*>
  l.add(l.first())
}
```

And in case of the return value with a captured type, it will be approximated.
If a captured type remains to be a type parameter of the top-level type,
it will be generalized (`List<X> -> \exists T. List<T>`) (preserving constraints).
If a captured type became the top-level type, it will be approximated with its upper bounds.
So, there are no unpacked existential type variables anywhere except the scope of each single function call.

### Proposed state

We propose to not approximate or generalize return types and allow captured types to be a valid type.
To achieve this, we should introduce a new existential type when we receive a value (even temporary one)
that has any projection as the direct type parameter.

* If projection is not a direct type parameter, it has not to be instantiated into a skolem.
  (F.e: in `val l: List<List<*>>` star have to remain a star projection)
  The reason is that `C<*>` is equal to `\exists T. C<T>`,
  consequently in case of `List<List<*>>` we have a type `List<\exists T. List<T>>`
  so our value is not a dependant pair with existential type that could be introduced.

* If a type of the value is declared with the same projection as the value,
  it should not be instantiated to the existential type variable as it may be expressed with common types.

TO DISCUSS: (Why not implicit packing-unpacking)
To be precise, we should consider this not as introduction of a new existential type,
but rather a first occurrence of the skolem variable.
This is because we should skolemize all of them for the easier inference.
Thus, the scope of existential variables is the whole function.

## Issues

### Different types of mutable variables with projections

There may be two different types of mutable variables.
The one that is actually the variable of the existential pair,
and it changes the value of their type on each reassignment (`var v: exists T. Box<T>`).

```kotlin
fun sameBoxes(b1: Box<T>, b2: Box<T>)

fun foo(l: List<Box<*>>) {
    val fst = l.first()
    var currentBox = fst
    for (el in l) {
        // el is actually a dependant pair.
        // In case of full-fledged existential types, we will reassign the type in the currentBox variable on each iteration.
        // In case of skolem types, we are able to express it using star projection
        // which will be a supertype of all the boxes with any specific skolem type.
        currentBox = el
      
        // TO DISCUSS:
        // succeeds for full-fledged existential types?
        // fails for skolemized existential types
        sameBoxes(currentBox, el)
    }
  
    // fails
    sameBoxes(currentBox, fst)
}
```

The others do not repack their existential variables (`exists T. var v: Box<T>`):

```kotlin
fun sameBoxes(b1: Box<T>, b2: Box<T>)

fun <T> transform(b: Box<T>): Box<T>

fun foo(l: List<Box<*>>) {
    val fst = l.first()
    var b = fst
    if (cond) {
        b = transform(b)
    }
    
    // succeeds
    sameBoxes(b, fst)
    
    for (el in l) {
        // fails
        b = el
    }
}
```

As both of them allow some code not allowed by the other one, we should allow both of them.

At first sight, the type with the skolem inferred from the first assignment is more precise.
But it will break some existing code as currently it behaves like the `var v: exists T. Box<T>`.
Consequently, we have to treat such variables (with and without explicit type annotation) this way.

Then the question is how to express the type of the variable that is not repacking its existential type.

The simplest solution is not to provide a possibility to express variables with such a type.
But it may be useful in some cases as it allows some additional behaviour.
For example:

```kotlin
TODO()
```

The first way to resolve it is
to allow explicitly specify the second type for some parameters of mutable variable's type.
Currently, Kotlin has only projection types to express existential types
that in context of mutable variables transform into `var v: exists T. Box<T>`, 
thus we have to introduce a new syntax for this case.

* For example, it may be `List<_>` or `List<?>`.
  Such a syntax will mean
  that we would like to use some skolem there that should be inferred from the initializing expression.
* The other option is if we introduce a syntax for the denotable existential type.
  For example, `List<t>` will mean that we would like to give a name for the skolem that will be used there.
  And this name could be used anywhere in the same way as the function's generic type parameter.
  In this case, programmer can just give a new fresh name instead of the underscore.
  But it will be more verbose and less clear
  and should be considered only
  if we would not like to introduce a new syntax for another type of unknown type parameter.

The other way is to try to infer the type of the variable from the usage.
We may use the flexible type `{List<*>, List<s1>}`, 
where the upper bound is the type as for star projection (`var v: exists T. Box<T>`), 
while the lower bound is the type with the existential type inferred from the first assignment 
(`exists T. var v: Box<t>`).
But this solution may lead to some problems with the inference
as it introduces a new class of flexible types which is not easy to handle,
fragile and not as useful as it is hard to support.

#### Immutable variables

This issue does not apply to the immutable variables.
As we are not able to reassign them, we may safely infer the most precise type for them.
In their types, we should replace all projections as the first level type parameters with the existential types.
Because such types will remain a subtype of the type with projections, it will not break any existing code.

### Non-local variables

The other issue is to track existential types in the non-local values and variables.
At first sight, we access them through the getter and setter functions.
As the other functions, they have to consume and return types with projections.
(Intuitively, this is because they are the values of type `Function<...>`, thus all projections are not first-level)
But programmer may want to use them as the values with the existential type parameters.

In this context, there are several issues:

* How to handle existential types with such scopes (class and global cases)?
* How to express the type of the non-local value with the existential type parameter?
* When are we able to use the existential type parameter implicitly?

TODO

### Tracking of the origin of the existential type

TODO:
We have to track the origin of the existential type to be able to give a proper error message.

### Scopes

TODO:
We have to care about scopes of the skolems as in this code:

```kotlin
fun foo(l: List<Box<*>>) {
    for (el in l) {
        // we have to care that any references to 't' does not escape the scope 
        // as on different iteration of cycle we have different values of t
        val curEl/*: Box<t> */ = el
    }
}
```

## Interactions with other features

### GADT

TODO:
We may infer local bounds in the same way as for GADT + generics, the mapping from variable to the inferred bounds.

### Bare types

TODO:
Using existential types + GADT inference, 
we are able to express bare types as a syntax sugar over instantiation of all parameters with star-projection.

### Builder inference

TODO: 
Care about the scope of the existential types.
But is not it the same as for the generic types?
