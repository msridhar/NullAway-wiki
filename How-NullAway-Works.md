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
path.  So, if there is a call _ap1_`.get(`_ap2_`)` underneath a
conditional `if (`_ap1_`.containsKey(`_ap2_`)`, where _ap1_ and _ap2_
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

