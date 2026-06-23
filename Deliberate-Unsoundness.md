NullAway deliberately eschews fully sound analysis in certain cases for simplicity and a reduced annotation burden. This page documents some known false negatives that come from this design decision.

For further background, see Section 4 of the [NullAway paper](https://manu.sridharan.net/files/FSE19NullAway.pdf) and the [[How NullAway Works]] page.

## Zero-argument method calls are assumed pure and deterministic

NullAway assumes that all methods are _pure_, i.e., free of side effects and deterministic.  In other words, it assumes that multiple calls to the same method return the same value, and that method calls do not mutate fields.

For example:

```java
FooHolder f = ...;
if (f.foo != null) {
  f.setFoo(null);
  f.foo.toString(); // NPE!
}
```
Here, NullAway assumes that `setFoo` does not mutate the `foo` field, so it misses the potential NPE at the `f.foo.toString()` call.  NullAway makes the side-effect-free assumption because we've found that examples like the above are very rare in practice, and assuming any call may have side effects leads to many false positives.

Here's an example related to determinism:

```java
FooHolder f = ...;
if (f.getFooOrNull() != null) {
  f.getFooOrNull().toString(); // potential NPE
}
```
Here, NullAway assumes that `getFooOrNull` is deterministic.  But, if the method is non-deterministic (e.g., it randomly returns `null`), the call `f.getFooOrNull().toString()` may through an NPE, and NullAway will not report it.

One can rewrite the code above to avoid any potential NPE:

```java
FooHolder f = ...;
Object o = f.getFooOrNull();
if (o != null) {
  o.toString();
}
```

NullAway makes the assumption of determinism to avoid forcing pervasive use of locals like this.  But, for maximum safety, a code rewrite to introduce a local may be best.

## Nullness facts are not invalidated after variable reassignment

When a variable is reassigned, NullAway does not re-check nullness facts that were learned before the reassignment.

For example:

```java
if (m.containsKey(o)) {
  m = new HashMap();
  m.get(o).toString();
}
```

In this case, NullAway may continue using the fact learned from `m.containsKey(o)` even after `m` has been reassigned. This is another deliberate tradeoff in favor of performance and implementation simplicity, and cases like the above are rare in practice.

## Map lookups are treated optimistically after key checks

As described on the [[Maps]] page, NullAway treats `m.get(k)` as `@NonNull` after checks like `m.containsKey(k)` or after a non-null `put(...)`, even though Java maps can legally store `null` values.

This is also unsound:

```java
if (m.containsKey(key)) {
  m.get(key).toString();
}
```

If `m` stores a `null` value for `key`, the dereference is still unsafe at runtime. NullAway deliberately ignores this possibility because many codebases treat map presence as implying a meaningful non-null value, and warning on every such usage would make the checker substantially noisier.

If your code relies on maps that may contain `null` values, prefer an explicit local check on `m.get(key)` instead of `containsKey(key)`.

## Array element reads are assumed `@NonNull` outside JSpecify mode

Outside JSpecify mode, NullAway cannot represent nullable array element types and unsoundly treats all array element reads as `@NonNull`.

For example:

```java
String[] arr = {"hello", null};
arr[1].toString(); // NullAway does not warn even though arr[1] is null
```

JSpecify mode adds support for nullable array element type annotations, enabling more precise analysis. If array contents are an important source of nullability bugs in your codebase, prefer enabling JSpecify mode and using explicit nullable type annotations on array element types where appropriate.

Even in JSpecify mode, NullAway assumes arrays have been initialized.  So, e.g., it does not warn on:

```java
String[] arr = new String[2];
arr[1].toString(); // NullAway does not warn even though arr[1] is null
```

## Multithreading

NullAway is not sound in the presence of multithreading.  So, for this example:

```java
class Test {
    static class Wrapper {
        @Nullable Wrapper f;
    }
    void test(Wrapper w) {
        // potential NPEs in presence of multithreading
        if (w.f != null && w.f.f != null) {
            w.f.f.hashCode(); 
        }
    }
}
```
NullAway does not report a warning for the above code.  But, in the presence of multithreading, `w.f` could get overwritten between the first check and the subsequent de-references, leading to an NPE.

## When you need more sound checking

The [Checker Framework Nullness Checker](https://checkerframework.org/manual/#nullness-checker) performs more sound checking for nullness issues, particularly around pure methods and maps.  If you require deeper verification, consider using that checker.
