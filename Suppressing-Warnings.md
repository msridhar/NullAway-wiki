In some cases, you will encounter code where a value can never be null, but due to limitations NullAway cannot prove this fact.  It is of course possible to add a redundant null check to address this issue, but sometimes this can clutter code and make the invariants less clear.  In such cases, we suggest the following warning suppression mechanisms.

### Method- or class-wide suppression

You can write `@SuppressWarnings("NullAway")` on any method, field, or class to suppress all NullAway warnings on that entity.  This suppression is often too coarse, suppressing warnings on much more than the problematic code; we encourage use of downcasting (described below) for most cases to narrow the suppression.

### Downcasting

One can easily write a "downcast" method that takes an argument of `@Nullable` type and returns the same value as `@NonNull`:
```
public static <T> T castToNonNull(@Nullable T x) {
    if (x == null) {
          // YOUR ERROR LOGGING AND REPORTING LOGIC GOES HERE
    }
    return x;
}
```
In the case where `x` is `null`, `castToNonNull` could throw an NPE for the greatest safety.  But, this runs the risk of introducing new failures into production code (e.g., if other code has defensive null checks even on `@NonNull` values).  Alternately, `castToNonNull` could just log the unexpected null value but then continue without failing fast, in which case the method should be annotated with `@SuppressWarnings("NullAway")`.  We do not provide a `castToNonNull` implementation with NullAway as it is quite simple and often has to integrate with a particular project's logging infrastructure.  

When would you use `castToNonNull`? Consider the following example, which uses a `Map` and iterates over its `keySet()`:

```
for(String key : map.keySet()) {
  // here m.get(key) cannot be null (assuming null values cannot be 
  // stored in the map), but it is @Nullable as far as the checker can tell,
  // as it does not yet reason about keySet()
  Object value = castToNonNull(m.get(key));
}
```