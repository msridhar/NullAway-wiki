This page documents the different code annotations that are supported by NullAway by default.  Custom annotations can be supported for certain functionality via the [command-line flags](https://github.com/uber/NullAway/wiki/Configuration).

### Nullability

NullAway treats any annotation whose simple (un-qualified) name is `@Nullable` as marking a parameter / return / field as nullable.  Checker Framework's [`@NullableDecl`](https://checkerframework.org/api/org/checkerframework/checker/nullness/compatqual/NullableDecl.html) is also supported.

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

### Field Contracts (precondition and postcondition)
* Precondition: `RequiresNonnull({"class_fields"})` 
* Postcondition: `EnsuresNonNull({"class_fields"})`

Allows to set preconditions and postconditions on class methods. 
If a method is annotataited with `RequiresNonnull` annotation, it is allowed to assume `Nullable` fields given in the parameter are `Nonnull`.
If a method is annotataited with `EnsuresNonnull` annotation, it must make sure that all class fields given in the parameter are `Nonnull` along all paths at exit point.


The following syntax rules applies to both annotations:
1. Cannot annotate a method with empty param set.
2. The receiver of selected fields in annotation can only be the receiver of the method.
3. All parameters given in the annotation must be one of the fields of the class or its super classes.
