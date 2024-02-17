This page documents the different code annotations that are supported by NullAway by default.  Custom annotations can be supported for certain functionality via the [command-line flags](https://github.com/uber/NullAway/wiki/Configuration).

### Nullability

NullAway treats any annotation whose simple (un-qualified) name is `@Nullable` as marking a parameter / return / field as nullable.  Checker Framework's [`@NullableDecl`](https://checkerframework.org/api/org/checkerframework/checker/nullness/compatqual/NullableDecl.html) and `javax.annotation.CheckForNull` are also supported.

For code considered annotated by NullAway, the tool assumes that the absence of a `@Nullable` annotation means the type is non-null. However, there are a number of features of NullAway, such as it's optional support for acknowledging [restrictive annotations](https://github.com/uber/NullAway/wiki/Configuration#acknowledge-more-restrictive-annotations-from-third-party-jars) in third-party jars, which involve checking for explicit `@NonNull` annotations. For this purpose, NullAway treats any annotation whose simple (un-qualified) name is `@NonNull`, `@Nonnull`, or `@NotNull`, as denoting an explicitly non-null type.

While NullAway considers non-null the default in annotated code, other tools might expect any of the above annotations. In particular, the annotation `javax.validation.constraints.NotNull` (and `@NotEmpty`) is actually used for dynamic validation of deserialized data (see https://beanvalidation.org/1.1/spec/) and is a good candidate for being added to `-XepOpt:NullAway:ExcludedFieldAnnotations=...`, since it implies external initialization of a field. 

There is also [optional support](https://github.com/uber/NullAway/wiki/Configuration#acknowledge-android-recent-nullability-annotations) for treating Android's `@RecentlyNullable` and `@RecentlyNonNull` as `@Nullable` and `@NonNull` annotations, respectively.

Finally, while we strongly recommend the use of standard nullability annotations (such as `org.jspecify.annotations.Nullable` or `javax.annotation.Nullable`) for broader tool compatibility, NullAway supports configuring additional [custom nullness annotations](https://github.com/uber/NullAway/wiki/Configuration#custom-nullability-annotations).

### Initialization

Any annotation whose simple (un-qualified) name is `@Initializer` is treated as marking a method as an initializer (see [here](https://github.com/uber/NullAway/wiki/Error-Messages#initializer-method-does-not-guarantee-nonnull-field-is-initialized--nonnull-field--not-initialized) for more information on initialization checking).  We also support JUnit's `@Before` and `@BeforeClass` for marking initializers.

### Contracts

NullAway has partial support for [JetBrains `@Contract` annotations](https://www.jetbrains.com/help/idea/contract-annotations.html).  Some examples:
```java
public class NullnessChecker {

  @Contract("_, null -> true")
  static boolean isNull(boolean flag, @Nullable Object o) { return o == null; }

  @Contract("null -> false")
  static boolean isNonNull(@Nullable Object o) { return o != null; }

  @Contract("null -> fail")
  static void assertNonNull(@Nullable Object o) { if (o == null) throw new Error(); }

  @Contract("!null -> !null")
  static @Nullable Object id(@Nullable Object o) { return o; }
}
```

For now, the `@Contract` annotations are trusted, not checked.  NullAway will warn if it sees a call to a method with a `@Contract` annotation it recognizes as invalid. 

Not all possible clauses of `@Contract` annotations are fully parsed or supported by NullAway (e.g. `@Contract("null, false -> fail; !null, true -> fail")` will be ignored, since NullAway cannot generally reason about the runtime truth value of the second argument).