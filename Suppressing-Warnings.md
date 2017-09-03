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

