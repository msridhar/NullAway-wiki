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

### Custom Initializer Annotations

  - `-XepOpt:NullAway:CustomInitializerAnnotations=...`

A list of annotations that should be considered equivalent to `@Initializer` annotations, and thus mark methods as initializers (e.g. `org.junit.Before` and `org.junit.BeforeClass`, which are automatically added to this list by default). Note that any annotation with the simple name `@Initializer`, from any package, will be considered an initializer annotation, but names passed to this configuration option must be fully-qualified class names.

### External Init Annotations

  - `-XepOpt:NullAway:ExternalInitAnnotations=...`

A list of annotations for classes that are "externally initialized."  Tools like the [Cassandra Object Mapper](https://docs.datastax.com/en/developer/java-driver/3.2/manual/object_mapper/) do their own field initialization of objects with a certain annotation (like [`@Table`](https://docs.datastax.com/en/drivers/java/3.2/com/datastax/driver/mapping/annotations/Table.html)), after invoking the zero-argument constructor. For any class annotated with an external-init annotation, we don't check that the zero-arg constructor initializes all non-null fields.

### Treat Generated As Unannotated

  - `-XepOpt:NullAway:TreatGeneratedAsUnannotated=...`

If set to `true`, NullAway treats any class annotated with `@Generated` as if its APIs are unannotated when analyzing uses from other classes.  It also does not perform analysis on the code inside `@Generated` classes.  If you can modify the code generator such that at least the APIs of `@Generated` classes are annotated correctly, we recommend using the `-XepOpt:NullAway:ExcludedClassAnnotations` option instead.  Defaults to `false`.

### Restricted Regexp Package Patterns

A few options, marked above, support a restricted regular expression syntax to specify the package names they cover. The main difference between our syntax and standard Java regular expressions, is that the `.` character is interpreted as a literal dot, not as "any character", as dots are part of the standard package name syntax and treating them literally favors the common case.

It is still possible to cover patterns of package names, such as:

`-XepOpt:NullAway:UnannotatedSubPackages=[a-zA-Z0-9.]*.unannotated`

(Will consider any code inside any subpackage named `unannotated`, including subpackages thereof, as unannotated. E.g. `a.unanoted`, `x.y.z.unannotated.z`.)

`-XepOpt:NullAway:UnannotatedSubPackages=com.myorg.generated_[a-zA-Z0-9]*`

(Matches `com.myorg.generated_Foo.subpackage`, but not `com.myorg.source_Foo.subpackage` or `comxmyorg.generated_Foo`.)

## Library Models

In addition to these options, NullAway will look for any classes implementing the `com.uber.nullaway.LibraryModels` interface, in the annotation processor path, and consider those as plug-in models for third-party unannotated libraries. (We search for such classes using the [ServiceLoader](https://docs.oracle.com/javase/7/docs/api/java/util/ServiceLoader.html) facility.) Models defined in such classes will be loaded in addition to the default models for common Java and Android libraries included with the checker itself. For documentation on writing such custom models, refer to the javadoc documentation for `com.uber.nullaway.LibraryModels` itself.  Also see our [sample library model](https://github.com/uber/NullAway/tree/master/sample-library-model) for an example; it is pulled in and used by our sample Java module (see [the `build.gradle` file](https://github.com/uber/NullAway/blob/ac6e3e1b63d357eec5f9e32fb02b024bf9cfb1f9/sample/build.gradle#L28)).  Note that if you can edit the source code of the library, you might be able to add [`@Contract` annotations](https://github.com/uber/NullAway/wiki/Supported-Annotations#contracts) instead of writing a library model.

## Other Build Systems

### Maven

Here is an example Maven build configuration that pulls in Error Prone and NullAway:

```xml
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.5</version>
        <configuration>
          <compilerId>javac-with-errorprone</compilerId>
          <forceJavacCompilerUse>true</forceJavacCompilerUse>
          <source>1.8</source>
          <target>1.8</target>
          <showWarnings>true</showWarnings>
          <annotationProcessorPaths>
            <path>
               <groupId>com.uber.nullaway</groupId>
               <artifactId>nullaway</artifactId>
               <version>0.3.0</version>
            </path>
          </annotationProcessorPaths>
          <compilerArgs>
            <arg>-Xep:NullAway:ERROR</arg>
            <arg>-XepOpt:NullAway:AnnotatedPackages=com.uber</arg>
          </compilerArgs>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>org.codehaus.plexus</groupId>
            <artifactId>plexus-compiler-javac-errorprone</artifactId>
            <version>2.8</version>
          </dependency>
          <!-- override plexus-compiler-javac-errorprone's dependency on
               Error Prone with the latest version -->
          <dependency>
            <groupId>com.google.errorprone</groupId>
            <artifactId>error_prone_core</artifactId>
            <version>2.1.3</version>
          </dependency>
        </dependencies>        
      </plugin>
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
