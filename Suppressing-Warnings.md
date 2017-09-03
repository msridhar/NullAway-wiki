= Suppressing Warnings  =

In some cases, you will encounter code where a value cannot ever be null, but because of limitations in our static analysis, the Nullability Checker cannot prove this fact. It is possible to suppress warnings for the Nullability Checker at the method level, but this is strongly discouraged. Here are a few options to consider, in order:

== Add nullability checks if unsure ==

The safest thing to do is to program defensively and add a nullability check (`if (x == null) { ... }`) to your code, which the analysis can reason about, as well as code to gracefully handle the potential case in which the value can be null. 

If you are 100% sure the value can never be null, consider if there is an alternative way to re-write your code so that it is obvious to the analysis that the value can never be null, without impacting code readability or clarity. If there isn't, then consider the next option.

== Use `castToNonNull()` to obtain a `@NonNull` reference from a `@Nullable` reference ==

This method is analogous to a Java explicit type cast and will produce a reference annotated as `@NonNull` from a `@Nullable` value.

Consider the following example, which uses a `Map` and iterates over its `keySet()`:

```
// Note: You might need to add "compile project(':libraries:foundation:codeanalysis')" to your .gradle file.
import static com.uber.codeanalysis.Nullability.castToNonNull;
...
for(String key : map.keySet()) {
  // here m.get(key) is @Nullable as far as the checker can tell, but value is @NonNull
  Object value = castToNonNull(m.get(key));
}
```

Note that if you call `y = castToNonNull(x)`, `x` will still be considered `@Nullable` going forward in the code (if it was `@Nullable` before), but `y` will be `@NonNull`, fitting the "type cast" intuition.

Please keep in mind that this method should only be used for values where it is clear that the value cannot ever be null (at least under the same assumptions for maps and collections we discuss above). When in doubt, please do an explicit nullability check. Note that if `castToNonNull(x)` ever does receive a `null` value at runtime, it will log an error when running in production, but continue ahead with execution, optimistically hoping not to run into an actual `NullPointerException`. In debug builds, it will crash immediately.


== Suppress method-wide ==

As the last resort, annotating a method with `@SuppressWarnings("CheckNullabilityTypes")` will cause the checker to ignore any and all potential nullability issues within the code for that entire method. This will add us as reviewers, and it makes small kittens cry.

One case where unfortunately suppressing warnings at the method level is currently the only option, is if you get spurious warnings about `@NonNull` fields not being properly initialized. If you are absolutely sure the field will always be correctly initialized by the constructors and initializers involved, yet the checker is unable to tell that, then `@SuppressWarnings("CheckNullabilityTypes")` is your only choice.

The `castToNonNull` method does not ship with the NullAway Open-Source release, since its implementation is dependent on whichever logging infrastructure the analyzed code uses. Implementing your own version of this method is both recommended, and trivial:

```
@SuppressWarnings("CheckNullabilityTypes")
public static <T> T castToNonNull(@Nullable T x) {
    if (x == null) {
          // YOUR ERROR LOGGING AND REPORTING LOGIC GOES HERE
    }
    return x;
}
```

