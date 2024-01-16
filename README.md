# Kotlin existential types proposal

## Useful ources

* Formalization of Java wildcards using existential types:
  [small](https://ecs.wgtn.ac.nz/foswiki/pub/Main/TechnicalReportSeries/ECSTR09-04.pdf),
  [medium](https://scholar.google.co.nz/citations?view_op=view_citation&hl=en&user=bVyUIHwAAAAJ&citation_for_view=bVyUIHwAAAAJ:2osOgNQ5qMEC) (Recommended),
  [big](https://www.ncameron.org/papers/cameron_thesis.pdf).

## Advantages

The main advantages of the skolem types instead of unidentified projections are:

* More type-safe expressive type-safe code without an explicit generics' introduction.
  For example, we would be possible to express AVL-tree with type-level control of branches balance using this feature (with GADT feature).

* More straightforward and precise implementation of bare types feature.
  With this feature, it may be expressed just like a syntax sugar over instantiation of all parameters with star-projection.
  As the projections became a skolem variable, 
  it will be possible to track bounds from all sources in a more precise, stable and predictable way.

* Less error-prone type system.
  As skolem types will explicitly track their type bounds, it will help to
  * More straightforward tracking soundness of the system.
  * More ease detection of the unreachable code and redundant branches

Implementation will be mostly invisible to the user. Exceptions:

* Error messages. It is quite hard and important to properly express type errors related to the skolem types.
  Possibly, we could do it in the same way as Scala.

* Possibly we could (later?) add an ability to give a name for the unknown generic parameter with underlying skolem for the code like this:
  ```kotlin
  val lst: MutableList<t> = foo()
  val fst: t = lst.first()
  lst.add(fst)
  ```

## Implementation

### Scala reference

In Scala, they are implemented using path-dependent typing. 
I do not see any reasons to do it in such a way (Except clarity in error messages).
AFAIK path is just a debug reference on the place where the skolem were bound.
It is not even used as ID as they are comparing skolems by reference.

But generally, the behaviour of the features could be the same as in Scala.

### Binding

It is not the identification of binding places, as all the skolems are bound on top of the function, 
before the generics' binding.
It is rather collection of all skolems required to successfully type-check a function.

New skolem type have to be introduced when we have a value (even temporary one) 
that has any projection as the direct type parameter.

* If projection is not a direct type parameter, it has not to be instantiated into a skolem. 
  (F.e: in `val l: List<List<*>>` star have to remain a star projection) 
  The reason is that `C<*>` is equal to `\exists T. C<T>`, 
  consequently in case of `List<List<*>>` we have a type `List<\exists T. List<T>>` 
  so our value is not a dependant pair with existential type that could be introduced.

* If a type of the value is declared with the same projection as the value, 
  it should not be instantiated to the skolem type as it may be expressed with common types.


## Backward compatibility

### More precision

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
