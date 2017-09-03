## Command-Line Options

The NullAway checker takes multiple configuration flags, in the format of [Error Prone's Command-Line Flags](http://errorprone.info/docs/flags). It will throw an error immediately upon loading if the `-XepOpt:NullAway:AnnotatedPackages` flag is not specified.

The following flags are currently supported; each of them can take multiple values as a comma separated list:

  - `-XepOpt:NullAway:AnnotatedPackages=...`

The list of packages that should be considered properly annotated according to the NullAway convention (every possibly null parameter / return / field annotated `@Nullable`).  E.g., `-XepOpt:NullAway:AnnotatedPackages=com.foo,org.bar`.

  - `-XepOpt:NullAway:UnannotatedSubPackages=...`

A list of subpackages to be excluded from the AnnotatedPackages list.  E.g., if all code under `com.foo` packages is properly annotated _except_ for code under `com.foo.baz`, you could use the options `-XepOpt:NullAway:AnnotatedPackages=com.foo -XepOpt:NullAway:UnannotatedSubPackages=com.foo.baz`.

  - `-XepOpt:NullAway:KnownInitializers=...`

The fully qualified name (without argument's, e.g. `android.app.Activity.onCreate`) of those methods from third-party libraries that NullAway should treat as initializers (equivalent to being annotated with an `@Initializer` annotation).

  - `-XepOpt:NullAway:ExcludedClassAnnotations=...`

A list of annotations that cause classes to be excluded from nullability analysis.

  - `-XepOpt:NullAway:ExcludedFieldAnnotations=...`

A list of annotations that cause fields to be excluded from being checked for proper initialization (e.g. `javax.inject.Inject`).

In addition to these options, NullAway will look for any classes implementing the `com.uber.nullaway.LibraryModels` interface, in the same classpath as the checker itself, and consider those as plug-in models for third-party unannotated libraries. Models defined in such classes will be loaded in addition to the default models for common Java and Android libraries included with the checker itself. For documentation on writing such custom models, refer to the javadoc documentation for `com.uber.nullaway.LibraryModels` itself.

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

