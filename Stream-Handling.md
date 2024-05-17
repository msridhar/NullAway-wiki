NullAway has specialized handling for certain stream APIs, in particular those from [`java.util.stream`](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html) and [RxJava](https://github.com/ReactiveX/RxJava).

## Knowledge of filter operations

It is a common pattern to filter a stream based on nullness and then perform further operations assuming those nullness facts hold.  Example:

```java
class Foo { @Nullable Bar f; }
...
Stream<Foo> stream = ...;
stream
    .filter(x -> x.f != null)
    .forEach(x -> System.out.println(x.f.g)); // no warning reported
```
Without specialized handling, a checker might treat the `x.f()` call on the last line as returning `@Nullable`, and hence would report a dereference-of-`@Nullable` error for the call `x.f().g()`.  NullAway is able to process the body of the lambda passed to `filter` and understand that in the subsequent `forEach` operator, `x.f()` will not return `null` for any `x` remaining in the stream.

NullAway is able to handle several cases of filtering but still may be missing some cases.  Please report an issue if you see a false positive due to a lack of understanding of stream filtering.

## Knowledge of synchronous callbacks

In general, it is not sound to propagate all nullability facts from a method to an enclosed lambda, as the lambda may be invoked asynchronously, at which point the facts may no longer hold.  For example:
```java
class Foo { @Nullable Bar f; }
...
Foo x = ...;
if (x.f != null) {
    runAsync(() -> System.out.println(x.f.g));
}
```
Here, it is possible that by the time the lambda runs, `x.f` may have been set back to `null`, the nullability info from the `x.f != null` check is not used when checking the lambda body.

However, there are some cases where a lambda is known to be run immediately and not asynchronously, e.g., the lambda passed to [`Map.forEach`](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html#forEach-java.util.function.BiConsumer-) or [`Collection.removeIf`](https://docs.oracle.com/javase/8/docs/api/java/util/Collection.html#removeIf-java.util.function.Predicate-).  NullAway has built-in knowledge of some of these cases, and will more aggressively propagate nullability facts from the containing method to lambda bodies for such cases.  E.g., NullAway does not report any errors for the following code:

```java
Foo x = ...;
Collection<String> c = ...;
if (x.f != null) {
    c.removeIf((y) -> x.f.toString().equals(y));
}
```

NullAway assumes that callbacks passed to _all_ methods of `java.util.stream.Stream` are run synchronously.  This is unsound in general, as a `Stream` object may be stored and only have its terminal operation invoked much later.  But, storing `Stream` objects in fields to be invoked later is an anti-pattern ([the Javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html) explicitly states that "A stream is not a data structure that stores elements"), so this assumption is almost always safe in practice.