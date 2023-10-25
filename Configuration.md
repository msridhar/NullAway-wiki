## Command-Line Options

The NullAway checker takes multiple configuration flags, in the format of [Error Prone's Command-Line Flags](http://errorprone.info/docs/flags). It will throw an error immediately upon loading if the `-XepOpt:NullAway:AnnotatedPackages` flag is not specified.

The following flags are currently supported; each of them can take multiple values as a comma separated list:

### Annotated Packages

  - `-XepOpt:NullAway:AnnotatedPackages=...`

The list of packages that should be considered properly annotated according to the NullAway convention (every possibly null parameter / return / field annotated `@Nullable`).  E.g., `-XepOpt:NullAway:AnnotatedPackages=com.foo,org.bar`.

This option supports [restricted regexp syntax](#restricted-regexp-package-patterns).

### Unannotated Sub-packages

  - `-XepOpt:NullAway:UnannotatedSubPackages=...`

A list of subpackages to be excluded from the AnnotatedPackages list.  E.g., if all code under `com.foo` packages is properly annotated _except_ for code under `com.foo.baz`, you could use the options `-XepOpt:NullAway:AnnotatedPackages=com.foo -XepOpt:NullAway:UnannotatedSubPackages=com.foo.baz`.

This option supports [restricted regexp syntax](#restricted-regexp-package-patterns).

### Unannotated Classes

  - `-XepOpt:NullAway:UnannotatedClasses=...`

A list of classes within annotated packages that should be treated as unannotated.  E.g., if all code under `com.foo` should be treated as annotated except for the class `com.foo.UnAnnot`, you could use the options `-XepOpt:NullAway:AnnotatedPackages=com.foo -XepOpt:NullAway:UnannotatedClasses=com.foo.UnAnnot`.

### Known Initializers

  - `-XepOpt:NullAway:KnownInitializers=...`

The fully qualified name (without argument's, e.g. `android.app.Activity.onCreate`) of those methods from third-party libraries that NullAway should treat as initializers (equivalent to being annotated with an `@Initializer` annotation).

### Excluded Class Annotations

  - `-XepOpt:NullAway:ExcludedClassAnnotations=...`

A list of annotations that cause classes to be excluded from nullability analysis.  Note that while NullAway does not analyze the code of these classes, it still assumes the APIs are annotated correctly when analyzing callers into methods of the classes.

### Excluded Classes

  - `-XepOpt:NullAway:ExcludedClasses=...`

A list of classes to be excluded from the nullability analysis.  Note that while NullAway does not analyze the code of these classes, it still assumes the APIs are annotated correctly when analyzing callers into methods of the classes.

### Excluded Field Annotations

  - `-XepOpt:NullAway:ExcludedFieldAnnotations=...`

A list of annotations that cause fields to be excluded from being checked for proper initialization (e.g. `javax.inject.Inject`).

This option supports [restricted regexp syntax](#restricted-regexp-package-patterns).

### Custom Nullability Annotations
[Since 0.9.3]

Please note that NullAway requires no configuration to support any annotations with simple name `@Nullable` (such as `javax.annotation.Nullable` or `androidx.annotation.Nullable`) and natively supports a number of other common nullability annotations. The same is true for common variations of `@NonNull`, when relevant. See [Supported Annotations](https://github.com/uber/NullAway/wiki/Supported-Annotations). 

There is also specific optional support for [Android's "Recent" Nullability Annotations](#acknowledge-android-recent-nullability-annotations), which can be configured separately. 

However, if your project uses annotations with non-standard names for nullability, which aren't part of a library NullAway is aware of, you can still specify them by using the following two configuration options:

  - `-XepOpt:NullAway:CustomNullableAnnotations=...`

A list of annotations that should be considered equivalent to `@Nullable` annotations. Note that any annotation with the simple name `@Nullable`, from any package, will be considered a nullable annotation (and thus doesn't need to be explicitly configured), but names passed to this configuration option must be fully-qualified class names.

  - `-XepOpt:NullAway:CustomNonnullAnnotations=...`

A list of annotations that should be considered equivalent to `@NonNull` annotations, for the cases where NullAway cares about such annotations (see e.g. [AcknowledgeRestrictiveAnnotations](#acknowledge-more-restrictive-annotations-from-third-party-jars)). Note that any annotation with the simple name `@NonNull`, `@NotNull`, or `@Nonnull`, from any package, will be considered a non-null annotation (and thus doesn't need to be explicitly configured), but names passed to this configuration option must be fully-qualified class names.

### Custom Initializer Annotations

  - `-XepOpt:NullAway:CustomInitializerAnnotations=...`

A list of annotations that should be considered equivalent to `@Initializer` annotations, and thus mark methods as initializers (e.g. `org.junit.Before` and `org.junit.BeforeClass`, which are automatically added to this list by default). Note that any annotation with the simple name `@Initializer`, from any package, will be considered an initializer annotation, but names passed to this configuration option must be fully-qualified class names.

### External Init Annotations

  - `-XepOpt:NullAway:ExternalInitAnnotations=...`

A list of annotations for classes that are "externally initialized."  Tools like the [Cassandra Object Mapper](https://docs.datastax.com/en/developer/java-driver/3.2/manual/object_mapper/) do their own field initialization of objects with a certain annotation (like [`@Table`](https://docs.datastax.com/en/drivers/java/3.2/com/datastax/driver/mapping/annotations/Table.html)), after invoking the zero-argument constructor. 

For any class annotated with an external-init annotation, we don't check that the _zero-arg constructor_ initializes all non-null fields.

Additionally, these external-init annotations will also be acknowledged if they annotate a _zero-arg constructor_ directly, rather than the full class, with equivalent semantics to `@SuppressWarnings("NullAway.Init")` on that same constructor. Note, however, that external-init annotations will be ignored if they appear in non-zero arg constructors or non-constructor methods (e.g. initializers).

### Acknowledge Assertions As Dynamic Checks

  - `-XepOpt:NullAway:AssertsEnabled=...`

By default, NullAway ignores assertions in the analyzed code. The reason for this is that assertions are not guaranteed to be checked at runtime.

Thus, in the following code:

```
T1 t1 = new T1();
assert t1.obj != null;
t1.obj.toString();
```

Whether `t1.obj` can or can't be `null` at the point `toString()` is called on it depends on the way the code is run and whether or not `assert` statements are being checked by the JVM (usually they are not in production runs). By default, NullAway assumes `assert` statements won't always be there at runtime, and thus it doesn't rely on nullness checks within assertions. 

Setting `AssertsEnabled=true` overrides this behavior and makes NullAway assume that assertions will always be enabled at runtime (e.g. that the code will always be run with `-enableassertions` being passed to the JVM).

### Treat Generated As Unannotated

  - `-XepOpt:NullAway:TreatGeneratedAsUnannotated=...`

If set to `true`, NullAway treats any class annotated with `@Generated` as if its APIs are unannotated when analyzing uses from other classes.  It also does not perform analysis on the code inside `@Generated` classes.  If you can modify the code generator such that at least the APIs of `@Generated` classes are annotated correctly, we recommend using the `-XepOpt:NullAway:ExcludedClassAnnotations` option instead.  Defaults to `false`.

### Acknowledge More Restrictive Annotations from Third-Party Jars

  - `-XepOpt:NullAway:AcknowledgeRestrictiveAnnotations=...`

This option adds some extra safety when dealing with third-party code that includes potentially incomplete nullability annotations. If set to `true`, NullAway will acknowledge nullability annotations whenever they are available in _unannotated_ code and also more restrictive than it's optimistic defaults. 

For example, when this flag is turned on, a method `Object foo(Object o)` in an unannotated package will still be assumed to take a nullable argument and return non-null (an optimistic assumption, made for usability reasons). However, a method `Object bar(@NonNull Object o)` will be interpreted by NullAway as taking a non-null argument, and a method `@Nullable Object baz(Object o)` will be acknowledged as potentially returning `null`, even in code otherwise marked as unannotated. If this flag is turned off, which is the default, all three examples are treated as the first and optimistic defaults are assumed to reduce the number of warnings unless the package is marked as fully [annotated](#annotated-packages).

Note that e.g. an `@NonNull` annotation in [annotated](#annotated-packages) code is meaningless, whether or not this flag is set, as that's NullAway's default for that code. But this affects NullAway's handling of code marked as _unannotated_. 

Also, specific method annotations can always be overridden by explicit [Library Models](#library-models), which take precedence over both the optimistic defaults and any annotations in the code, whether marked as annotated or unannotated.

### Acknowledge Android "Recent" Nullability Annotations

  - `-XepOpt:Nullaway:AcknowledgeAndroidRecent=...`

If this flag is set to `true` *and* `-XepOpt:Nullaway:AcknowledgeRestrictiveAnnotations` is set to `true`, we treat the Android `@RecentlyNullable` annotation as `@Nullable`, and similarly for `@RecentlyNonNull`.

### Specify `castToNonNull` Method

  - `-XepOpt:NullAway:CastToNonNullMethod=[...]`

Specifies the method being used for casting `@Nullable` expressions to `@NonNull`; see [downcasting docs](https://github.com/uber/NullAway/wiki/Suppressing-Warnings#downcasting).  When this option is passed, NullAway will emit a warning if a downcast is ever performed on an expression that is already `@NonNull`.

### Perform Exhaustive Override Checks

  - `-XepOpt:NullAway:ExhaustiveOverride=...`

By default, NullAway relies on the `@Override` annotation to know which methods override a super-type method and thus must be checked for consistency in the nullness of their return value and parameters. This is done for performance reasons.

In codebases where `@Override` is not used consistently, this flag should be set to `true`. It forces NullAway to check every method to see whether or not it overrides a method of a super-type. 

### Optional Emptiness Check

  - `-XepOpt:NullAway:CheckOptionalEmptiness=...`

If set to `true`, NullAway will check for `.get()` accesses to potentially empty `Optional` values, analogously to how it handles dereferences to `@Nullable` values. Calling `.get()` on an `Optional` value that hasn't been previously tested with `Optional.isPresent(...)` (or otherwise tested for non-emptiness in a way NullAway understands) will result in an error.

This works out of the box for `java.util.Optional`. Additionally, the following option can be used to tell NullAway of other `Optional` implementations (e.g. Guava's `com.google.common.base.Optional`).

  - `-XepOpt:NullAway:CheckOptionalEmptinessCustomClasses=...`
 
Optional handling (for both JDK and custom `Optional` classes) is currently disabled by default, but it is under active development. Feel free to try it out and let us know what might still be missing.

### Assertions in Test Libraries

  - `-XepOpt:NullAway:HandleTestAssertionLibraries=...`

By default, NullAway does not handle assertions from test libraries. If set to `true`, NullAway will handle assertions from test libraries, like `assertThat(...).isNotNull()`, and use that to reason about the possibility of null dereferences in the code that follows these assertions.

### Contract Checking

  - `-XepOpt:NullAway:CheckContracts=...`

NullAway has [support for JetBrains `@Contract` annotations](https://github.com/uber/NullAway/wiki/Supported-Annotations#contracts).  By default, these annotations are trusted but not checked.  If this option is set to `true`, NullAway will check that `@Contract` annotations are valid (for the subset of `@Contract` annotations supported by NullAway).


### Custom Contract Annotations

  - `-XepOpt:NullAway:CustomContractAnnotations=...`

By default, NullAway recognizes `org.jetbrains.annotations.Contract` as a [contract annotation](https://github.com/uber/NullAway/wiki/Supported-Annotations#contracts).  This option allows for specifying an additional list of annotations that should be recognized as providing contracts (with the same contract syntax as `org.jetbrains.annotations.Contract`).

### Restricted Regexp Package Patterns

A few options, marked above, support a restricted regular expression syntax to specify the package names they cover. The main difference between our syntax and standard Java regular expressions, is that the `.` character is interpreted as a literal dot, not as "any character", as dots are part of the standard package name syntax and treating them literally favors the common case.

It is still possible to cover patterns of package names, such as:

`-XepOpt:NullAway:UnannotatedSubPackages=[a-zA-Z0-9.]*.unannotated`

(Will consider any code inside any subpackage named `unannotated`, including subpackages thereof, as unannotated. E.g. `a.unanoted`, `x.y.z.unannotated.z`.)

`-XepOpt:NullAway:UnannotatedSubPackages=com.myorg.generated_[a-zA-Z0-9]*`

(Matches `com.myorg.generated_Foo.subpackage`, but not `com.myorg.source_Foo.subpackage` or `comxmyorg.generated_Foo`.)

## Library Models

In addition to these options, NullAway will look for any classes implementing the `com.uber.nullaway.LibraryModels` interface, in the annotation processor path, and consider those as plug-in models for third-party unannotated libraries. (We search for such classes using the [ServiceLoader](https://docs.oracle.com/javase/7/docs/api/java/util/ServiceLoader.html) facility.) Models defined in such classes will be loaded in addition to the default models for common Java and Android libraries included with the checker itself. For documentation on writing such custom models, refer to the javadoc documentation for [`com.uber.nullaway.LibraryModels`](https://github.com/uber/NullAway/blob/master/nullaway/src/main/java/com/uber/nullaway/LibraryModels.java) itself.  Also see our [sample library model](https://github.com/uber/NullAway/tree/master/sample-library-model) for an example; it is pulled in and used by our sample Java module (see [the `build.gradle` file](https://github.com/uber/NullAway/blob/ac6e3e1b63d357eec5f9e32fb02b024bf9cfb1f9/sample/build.gradle#L28)).  Note that if you can edit the source code of the library, you might be able to add [`@Contract` annotations](https://github.com/uber/NullAway/wiki/Supported-Annotations#contracts) instead of writing a library model.

Sometimes, it is useful to suppress some library models on a per-compilation target basis, without changing or adding custom library models classes. In these cases, it is possible to skip all Library Models (included with NullAway and custom) for a specific method, simply by passing it to the following option:

  - `-XepOpt:NullAway:IgnoreLibraryModelsFor=com.example.Foo.bar,com.example.Foo.baz`

This takes a list of comma-separated methods, given as their fully-qualified class name plus the method simple name. 

Note that, for simplicity of dealing with command-line escaped characters and the like, there is currently no way to give the full signature of the method (i.e. the argument types) when using this option. Thus, methods passed to ``-XepOpt:NullAway:IgnoreLibraryModelsFor` will technically refer to all methods matching the given class and method name.

## Other Build Systems

### Maven

For configuring Error Prone with Maven, see [the docs](http://errorprone.info/docs/installation#maven).  It's also good to be familiar with how flags should be passed to Error Prone with Maven; see [these docs](http://errorprone.info/docs/flags#maven).
Here is an example Maven build configuration (based on [the Error Prone example](https://errorprone.info/docs/installation#maven)) that pulls in Error Prone and NullAway:

```xml
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>11</source>
          <target>11</target>
          <encoding>UTF-8</encoding>
          <compilerArgs>
            <arg>-XDcompilePolicy=simple</arg>
            <arg>-Xplugin:ErrorProne -XepOpt:NullAway:AnnotatedPackages=com.uber</arg>
          </compilerArgs>
          <annotationProcessorPaths>
            <path>
              <groupId>com.google.errorprone</groupId>
              <artifactId>error_prone_core</artifactId>
              <version>2.23.0</version>
            </path>
            <path>
              <groupId>com.uber.nullaway</groupId>
              <artifactId>nullaway</artifactId>
              <version>0.10.15</version>
            </path>
          </annotationProcessorPaths>
        </configuration>
      </plugin>
    </plugins>
  </build>
```

### Bazel

An example (based on [this](https://github.com/uber/NullAway/issues/119#issue-294049136)).

WORKSPACE:
```python
maven_jar(
	name='jsr305',
	artifact='com.google.code.findbugs:jsr305:3.0.2',
)

maven_jar(
	name="nullaway",
	artifact="com.uber.nullaway:nullaway:0.3.4"
)
```

BUILD:
```python
java_library(
	name='x',
	srcs=['X.java'],
	deps=['@jsr305//jar'],
	plugins=['nullaway'],
	javacopts=[
		'-Xep:NullAway:ERROR',
		'-XepOpt:NullAway:AnnotatedPackages=com.example',
	],
)

java_plugin(
	name='nullaway',
	deps=[
		'@nullaway//jar'
	],
)
```
### Other

The first step is to get Error Prone running on your build, as documented [here](http://errorprone.info/docs/installation).  Then, you need to get the NullAway jar on the annotation processor path for the Javac invocations where Error Prone is running.  Finally, you need to pass the appropriate compiler arguments to configure NullAway (at the least, the `-XepOpt:NullAway:AnnotatedPackages` option).

## Removed Command-Line Options

### Acknowledge Library Models of Annotated Code

[Only in version 0.9.9]

  - `-XepOpt:NullAway:AcknowledgeLibraryModelsOfAnnotatedCode=...`

This option allows [library models](#library-models) to override the annotations on methods within an annotated package.  This is useful, e.g., if you want to treat Guava's `com.google.common` packages as annotated, but also want the default library models of Guava APIs (e.g., of [`Strings.isNullOrEmpty()`](https://github.com/uber/NullAway/blob/a8051cd9672e88eabeec6d29ae58c142165fb9f1/nullaway/src/main/java/com/uber/nullaway/handlers/LibraryModelsHandler.java#L520)) to still apply.

In NullAway 0.9.10+, this option is removed, and library models always override annotations on methods within an annotated packaged (i.e., the option is always on).  

