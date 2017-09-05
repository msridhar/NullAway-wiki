NullAway assumes the return value of a call `java.util.Map.get()` or `com.google.common.collect.ImmutableMap.get()` is `@Nullable` unless it finds an appropriate check operation.  You can check the nullness of a `get()` call directly:
```
if (m.get(key) != null) {
  // here m.get(key) is @NonNull
}
```
Or you can use `containsKey()`:
```
if (m.containsKey(key)) {
  // here m.get(key) is @NonNull
}
```
Or you can initialize the value using `put()`:
```
if (!m.containsKey(key)) {
  m.put(key,newValue);
}
// here m.get(key) is @NonNull, assuming newValue is @NonNull
```
The checker does not yet understand more complex cases like iterating over a map's `keySet()`; see [[Suppressing Warnings]] for workarounds.  The checker also does not yet understand autoboxing, so it may complain if, e.g., you directly use an `int` or a `char` in calls to map methods.

If you use a complex expression to hold the key, the checker may complain, e.g.:
```
if (m.containsKey(x.foo(y).baz(a,b))) {
  // checker cannot show that m.get(x.foo(y).baz(a,b)) is @NonNull here
  ...
}
```
These cases can be addressed by storing the complex expression in a local, e.g.:
```
Object key = x.foo(y).baz(a,b);
if (m.containsKey(key)) {
  // checker knows m.get(key) is @NonNull here
}
```

Finally, you may have noticed that the checker is being a bit lax: just because `m.containsKey(key)` is true, it does not mean that `m.get(key)` is `@NonNull`, since the map may contain null values.  We deliberately ignore this case to make the checker more usable.  It is best to avoid storing `null` values in a map; e.g. the Guava collections [disallow `null` values](https://github.com/google/guava/wiki/UsingAndAvoidingNullExplained).