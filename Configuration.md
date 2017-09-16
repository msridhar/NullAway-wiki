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

  - `-XepOpt:NullAway:CustomInitializerAnnotations=...`

A list of annotations that should be considered equivalent to `@Initializer` annotations, and thus mark methods as initializers (e.g. `org.junit.Before` and `org.junit.BeforeClass`, which are automatically added to this list by default). Note that any annotation with the simple name `@Initializer`, from any package, will be considered an initializer annotation, but names passed to this configuration option must be fully-qualified class names.

## Library Models

In addition to these options, NullAway will look for any classes implementing the `com.uber.nullaway.LibraryModels` interface, in the annotation processor path, and consider those as plug-in models for third-party unannotated libraries. (We search for such classes using the [ServiceLoader](https://docs.oracle.com/javase/7/docs/api/java/util/ServiceLoader.html) facility.) Models defined in such classes will be loaded in addition to the default models for common Java and Android libraries included with the checker itself. For documentation on writing such custom models, refer to the javadoc documentation for `com.uber.nullaway.LibraryModels` itself.

