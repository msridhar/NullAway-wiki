This page gives some further details on the inner workings of
NullAway.

## Type Inference

The most complex part of the NullAway implementation is its
intra-procedural type inference, which infers nullability information
for local variables and some types of expressions based on code within
the same method.  The inference can infer different types for locals /
expressions at different program points, e.g.:
```java
void foo(@Nullable Object x) {
  if (x != null) {
    // inference learns x is @NonNull here
    x.toString();
  }
  // x is still @Nullable here, hence error is reported
  x.hashCode();
}
```

The inference also learns nullability facts for more complex
expressions, e.g.:
```java
class ListNode { 
  int data; 
  @Nullable ListNode next; 
  @Nullable ListNode getNext() { return next; }
  
  void doStuff() {
    if (this.next != null && this.next.getNext() != null) {
       // NullAway infers this.next.getNext() to be @NonNull here
       this.next.getNext().toString();
    }
  }
}
```

Type inference is implemented as a dataflow analysis, leveraging the
Checker
Framework's
[dataflow library](https://checkerframework.org/manual/checker-framework-dataflow-manual.pdf).
An abstract state is a mapping from
[access paths](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/dataflow/AccessPath.java) to a
corresponding
[nullability state](https://github.com/uber/NullAway/blob/master/nullaway/src/main/java/com/uber/nullaway/Nullness.java).
For further background on access paths see Section 5.2
of
[this document](https://manu.sridharan.net/files/aliasAnalysisChapter.pdf).
NullAway allows for zero-argument method calls (like getters) to be
included in access paths along with field names.  NullAway also makes
the simplifying (but unsound) assumption that callees perform no
mutation, and hence does not invalidate nullability state for access
paths across method calls.

The dataflow transfer functions are
implemented
[here](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/dataflow/AccessPathNullnessPropagation.java).
For the most part, the transfer functions simply propagate information
learned from assignments and null checks for any expression
representable as an access path.  Some care must be taken
in
[constructing the initial store](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/dataflow/AccessPathNullnessPropagation.java#L193) when
analyzing the body of a lambda expression.  Unlike normal methods,
lambda parameters inherit the nullability of their corresponding
functional interface methods, to eliminate verbosity, so some logic is
required to "infer" the initial nullability of the lambda parameters.

For matching of `containsKey()` and `get()` calls
for [maps](https://github.com/uber/NullAway/wiki/Maps), NullAway
additionally tracks the receiver and first argument to such calls as an access
path.  So, if there is a call `ap1.get(ap2)` underneath a
conditional `if (ap1.containsKey(ap2))`, where `ap1` and `ap2`
are representable access paths, NullAway treats the `get()` call as
returning `@NonNull`.

## Error Checking

Given the results of type inference, in some cases, error checking
involves directly checking the conditions that would lead to
each
possible
[error message](https://github.com/uber/NullAway/wiki/Error-Messages).
The logic is in the
main
[`NullAway`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/NullAway.java) class.
E.g., to
check
[fields assignments](https://github.com/uber/NullAway/wiki/Error-Messages#assigning-nullable-expression-to-nonnull-field),
we [ensure](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/NullAway.java#L342) that if the field is `@NonNull`, the right-hand side
expression of the assignment is not `@Nullable`.  Checking of
[dereferences](https://github.com/uber/NullAway/wiki/Error-Messages#dereferenced-expression-is-nullable),
[parameter passing](https://github.com/uber/NullAway/wiki/Error-Messages#passing-nullable-parameter-where-nonnull-is-required) and
[returns](https://github.com/uber/NullAway/wiki/Error-Messages#returning-nullable-expression-from-method-with-nonnull-return-type),
and
[method](https://github.com/uber/NullAway/wiki/Error-Messages#assigning-nullable-expression-to-nonnull-field) [overriding](https://github.com/uber/NullAway/wiki/Error-Messages#parameter-is-nonnull-but-parameter-in-superclass-method-is-nullable) is
similarly straightforward.

Checking
for
[proper initialization of `@NonNull` fields](https://github.com/uber/NullAway/wiki/Error-Messages#initializer-method-does-not-guarantee-nonnull-field-is-initialized--nonnull-field--not-initialized) is
more involved.  Here, we must again leverage the results of dataflow
analysis, to show that the relevant field is `@NonNull` at the exit
node of the relevant constructor / initializer (i.e., it is
initialized on *all* paths).  Similarly, dataflow analysis is used to
check
for
[read-before-init errors](https://github.com/uber/NullAway/wiki/Error-Messages#read-of-nonnull-field-before-initialization),
by inferring field nullability at the program point before the read.
Our dataflow analysis has been designed such that we can analyze a
method *once* (an expensive operation) and re-use the result for type
inference and all initialization checking.  NullAway also has targeted
inter-procedural reasoning to allow, e.g., for field initialization in
a private method that is always invoked from an initializer.

Some other special cases are worth mentioning.  Java 8 lambdas are
treated as overrides of the corresponding functional interface
methods, so override checking proceeds similarly to that of normal
methods, but with inherited nullability for unannotated parameters (as
discussed in the above section on inference).  Method references are
also checked against the expected functional interface method, but
careful handling of corner cases
like
[unbound instance methods](https://github.com/uber/NullAway/wiki/Error-Messages#unbound-instance-method-reference-cannot-be-used-as-first-parameter-of-functional-interface-method-is-nullable) is
required.  Finally, unboxing of a `null` value leads to an NPE, so we
introduce checks for
nullability
[for](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/NullAway.java#L348) [various](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/NullAway.java#L393) [operations](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/NullAway.java#L537) that
cause unboxing.

## Extensibility through Handlers

The main way to extend the behavior of NullAway beyond the core inference and checking described above, is to use a handler. 

Through the NullAway codebase, there are multiple extension points, at which the core code calls specific methods of the [`Handler`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/Handler.java) interface for all registered handler objects. To extend or override the default behavior for NullAway, it often suffices to scan that interface for the corresponding extension point method(s), then subclass [`BaseNoOpHandler`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/BaseNoOpHandler.java) overriding only said method(s), and finally add an instance of the new handler at the end of the list returned by [`Handlers.buildDefault()`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/Handlers.java#L37).

For example, consider [`LibraryModelsHandler`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/LibraryModelsHandler.java), which is used to provide models for unannotated third-party library methods. This handler first uses a [service loader](https://docs.oracle.com/javase/7/docs/api/java/util/ServiceLoader.html) to find custom models for methods, in addition to those declared inside the `DefaultLibraryModels` class. It then registers itself as a handler overriding, among others, [`Handler.onDataflowVisitMethodInvocation`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/Handler.java#L195). This particular extension hook allows it to change, on a callsite by callsite basis, the nullability assumption for the method's return value. By default, NullAway assumes unannotated methods return `@NonNull`. However, the `LibraryModelsHandler` changes that value for those methods for which an explicit library model exists which shows the return method to be `@Nullable`, and the NullAway core dataflow simply moves forward propagating that `@Nullable` value in those cases. 

`LibraryModelsHandler` overrides similar extension points to mark the arguments to certain library methods as `@NonNull` and check for correct overriding of modeled library methods. Because of the handlers mechanism, the core NullAway classes don't need to know about the existence of library models at all, they only need the general hooks to override the default return nullability and argument nullability of unannotated methods. These same hooks can be used by other handlers that derive this information from sources other than explicit library models (e.g. `ContractHandler` below, or even our [experimental bytecode analysis](https://github.com/uber/NullAway/blob/4c1b1beedd0aca420d91371455a8a15efa12d262/nullaway/src/main/java/com/uber/nullaway/handlers/InferredJARModelsHandler.java)).

Some other NullAway extensions implemented as handlers include:

* [`RxNullabilityPropagator`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/RxNullabilityPropagator.java): This is a handler which propagates nullability information across call boundaries for select methods inside an [Rx stream](https://github.com/ReactiveX/RxJava/wiki). For example, this allows handling the following code: `observable.filter(o -> o != null).map(o -> o.toString())`. This is an example of handlers being used to add limited forms of inter-procedural inference, which is prohibitively expensive in the general case.
* [`ContractHandler`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/ContractHandler.java): Adds support for a subset of the specifications of the [`@Contract` annotation](https://www.jetbrains.com/help/idea/contract-annotations.html) (e.g. `@Contract("!null -> !null")` means that if the method's sole argument is `@NonNull`, then the dataflow analysis should also assume the return is `@NonNull`).
* [`ApacheThriftIsSetHandler`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/ApacheThriftIsSetHandler.java): Correctly interprets `isSetXXXX()` calls inside code generated by the [Apache Thrift library](https://thrift.apache.org) as checking the nullability of property `XXXX`.
* [`OptionalEmptinessHandler`](https://github.com/uber/NullAway/blob/4a4b59127786b6da6c358ba04158003d29dfa590/nullaway/src/main/java/com/uber/nullaway/handlers/OptionalEmptinessHandler.java): Implements our support for preventing `.get(...)` calls on `Optional` values without checking that the `Optional` is non-empty, effectively extending NullAway to handle accesses to potentially empty optional values the same it way it handles dereference of `@Nullable` values, without any optional-specific changes to the core tool.

Note that handlers overriding the same extension method are chained together, and the order of the handler chain in `Handlers.buildDefault()` can matter in certain cases. Please refer to the documentation of the corresponding [`Handler`](https://github.com/uber/NullAway/blob/cfb1e2449b4e6d4187fcfa73ff638e3bc591603f/nullaway/src/main/java/com/uber/nullaway/handlers/Handler.java) interface method. In general, due to the performance focus of NullAway, handler chains should remain shallow for most extension methods, and handlers are expected to be as performant as core code. One important advantage of handlers, besides helping to keep the core code readable, is that they allow turning on and off specific non-core NullAway features by simply adding or removing the corresponding handler from `Handlers.buildDefault()`.

## JSpecify

Work is underway to implement support for the [JSpecify](https://jspecify.dev) specification of the semantics of Java nullability annotations.  This work includes support for nullability annotations on generic type arguments.  For now this support is mostly off by default, but can be enabled by passing the `-XepOpt:NullAway:JSpecifyMode` flag.  An exception is support for the [`@NullMarked`](https://jspecify.dev/docs/api/org/jspecify/annotations/NullMarked.html) and [`@NullUnmarked`](https://jspecify.dev/docs/api/org/jspecify/annotations/NullUnmarked.html) annotations, which is on by default (except for module support, which is not yet implemented).

Relevant code for implementing JSpecify support sits in the [`com.uber.nullaway.generics`](https://github.com/uber/NullAway/tree/09db47a4d99440e60d03e80d17e358299a618127/nullaway/src/main/java/com/uber/nullaway/generics) package, mainly in the [`GenericsChecks`](https://github.com/uber/NullAway/blob/09db47a4d99440e60d03e80d17e358299a618127/nullaway/src/main/java/com/uber/nullaway/generics/GenericsChecks.java) class.  Tests are in the [`com.uber.nullaway.jspecify`](https://github.com/uber/NullAway/tree/719b167ded3f65c4eb454d1909f436e9dd514076/nullaway/src/test/java/com/uber/nullaway/jspecify) package.