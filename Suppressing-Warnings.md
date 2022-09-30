In some cases, you will encounter code where a value can never be null, but due to limitations NullAway cannot prove this fact.  It is of course possible to add a redundant null check to address this issue, but sometimes this can clutter code and make the invariants less clear.  In such cases, we suggest the following warning suppression mechanisms.

### Method- or class-wide suppression

You can write `@SuppressWarnings("NullAway")` on any method, field, or class to suppress all NullAway warnings on that entity.  This suppression is often too coarse, suppressing warnings on much more than the problematic code; we encourage use of downcasting (described below) for most cases to narrow the suppression.

### Suppressing only initialization warnings

You can write `@SuppressWarnings("NullAway.Init")` on a field declaration to only suppress warnings related to possibly-missing initialization of that field.  You can also write `@SuppressWarnings("NullAway.Init")` on a constructor to suppress initialization-related warnings for that constructor, or `@SuppressWarnings("NullAway.Init")` on a class to suppress all initialization-related warnings for the class.

### Downcasting

One can easily write a "downcast" method that takes an argument of `@Nullable` type and returns the same value as `@NonNull`:
```java
public static <T> T castToNonNull(@Nullable T x) {
    if (x == null) {
          // YOUR ERROR LOGGING AND REPORTING LOGIC GOES HERE
    }
    return x;
}
```
In the case where `x` is `null`, `castToNonNull` could throw an NPE for the greatest safety.  But, this runs the risk of introducing new failures into production code (e.g., if other code has defensive null checks even on `@NonNull` values).  Alternately, `castToNonNull` could just log the unexpected null value but then continue without failing fast, in which case the method should be annotated with `@SuppressWarnings("NullAway")`.  We do not provide a `castToNonNull` implementation with NullAway as it is quite simple and often has to integrate with a particular project's logging infrastructure.  

When would you use `castToNonNull`? Consider the following (contrived for simplicity) example:

```java
boolean hasHelloKey(Map<String,String> map) {
  return map.containsKey("hello");
}
void foo(Map<String,String> map) {
  if (hasHelloKey(map)) {
    // here map.get("hello") cannot be null (assuming null values cannot be 
    // stored in the map), but it is @Nullable as far as the checker can tell,
    // as it does not reason about the containsKey() call inside hasHelloKey()
    String value = castToNonNull(map.get("hello"));
    ...
  }
}
```
In real-world code, there are a variety of scenarios in which an expression can never be null, but NullAway cannot prove it, and use of `castToNonNull` is often appropriate for these cases.

When auto-patching (see below), it can be useful to tell NullAway about your `castToNonNull` method. The option `-Xep:NullAway:CastToNonNullMethod=[...]` does just that, allowing NullAway to add calls to this method rather than standard suppressions in some instances.  Adding this option will also lead NullAway to emit a warning if your `castToNonNull` method is invoked with a `@NonNull` argument.

### Auto Suppressing

If you pass the option `-XepOpt:NullAway:SuggestSuppressions=true` to NullAway, it will use Error Prone's suggested fix functionality to suggest suppressing any warning that it finds.  In combination with Error Prone's [patching functionality](http://errorprone.info/docs/patching) you can use this feature to auto-suppress all existing warnings in a code base.

You might also wish to add a comment to the suggested suppressions with `--Xep:NullAway:AutoFixSuppressionComment=\"[ some comment ]\"`. This string will be added as `/* some comment */` alongside the `@SuppressWarnings("NullAway")` annotation. Note that, when building with gradle, this string might not contain spaces. We have still found it useful for e.g. linking an issue/task number to a series of suppressions. 