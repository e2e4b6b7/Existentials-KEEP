# Kotlin existential types proposal

## Useful sources

* Why existentials were removed in Scala 3: [GH issue](https://github.com/lampepfl/dotty/issues/4353)

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

* Possibly we could later add an ability to give a name for the unknown generic parameter with underlying existential type for the code like this:
  ```kotlin
  val lst: MutableList<t> = foo()
  val fst: t = lst.first()
  lst.add(fst)
  ```

## Implementation

### Scala reference

In Scala, they are saying that it is implemented using path-dependent typing. 
I do not see any reasons to do it in such a way (Except clarity in error messages).
AFAIK path is just a debug reference on the place where the skolem were bound.
It is not even used as ID as they are comparing skolems by reference.

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

#### Mutable variables

There may be two different types of mutable variables.
The one which is actually the variable of the existential pair and it changes the value of their type on each reassignment:

```kotlin
fun sameBoxes(b1: Box<T>, b2: Box<T>)

fun foo(l: List<Box<*>>) {
    val fst = l.first()
    var currentBox = fst
    for (el in l) {
        // el is actually a dependant pair.
        // In case of full-fledged existential types, we will reassign the type in the currentBox variable on each iteration.
        // In case of skolemized existentials, we are able to express it using another skolem variable (not from any of the boxes) 
        // which will be a supertype of variables from all the boxes
        currentBox = el
        // succeeds for full-fledged existential types?
        // fails for skolemized existential types
        sameBoxes(currentBox, el)
    }
    // fails
    sameBoxes(currentBox, fst)
}
```

The others do not repack their existential variables:

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
}
```

##### New denotable type

TODO:
If we would like to differentiate this cases we may introduce a new syntax (f.e. `List<_>`) 
that will mean that we would like to use a skolem there but everywhere the same as in the first assignment.

#### Non-local values

TODO: Options: 
* capture the skolem globally and use it everywhere without an explicit copy to the local variable. (result value may be more precise)
* implicitly transform or explicitly ask a programmer to assign this value to the local variable.

#### Scoped skolems

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

### Bounds tracking

TODO:
We may infer local bounds in the same way as for GADT + generics, the mapping from variable to the inferred bounds.

## Backward compatibility

### More precision

TODO: merge with 'Mutable variables' section

For mutable variables, it is possible to infer a more precise type using skolem type. 
This may lead to type errors in the code where more general types were implicitly supposed to be.
For example:

```kotlin
fun minOrDefault(list: List<Box<*>>, default: Box<*>): Box<*> {
  // Here types are:  
  // list: List<Box<s1>>
  // default: Box<s2>
  // s1, s2 are different skolem types  
  var min = list.first()
  // before: min: Box<*> == \exists T. Box<T>
  // after:  min: Box<s1>
  for (elem in list) {
      if (elem.ord() < min.ord()) {
          min = elem
      }
  }
  if (min.ord() > default.ord()) {
      min = default // after: type error: s1 != s2
  }
  return min
}
```

Possible solutions: 

* Deskolemize variables to type them as before. 
* Use a flexible type to postpone choice between those types.
