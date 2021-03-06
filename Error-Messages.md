Here we explain the different warning messages that NullAway produces and how to address them.  For an overview of NullAway see [the main README](https://github.com/uber/NullAway/blob/master/README.md).

## Table of Contents

* [dereferenced expression is @Nullable](#dereferenced-expression-is-nullable)
* [returning @Nullable expression from method with @NonNull return type](#returning-nullable-expression-from-method-with-nonnull-return-type)
* [passing @Nullable parameter where @NonNull is required](#passing-nullable-parameter-where-nonnull-is-required)
* [assigning @Nullable expression to @NonNull field](#assigning-nullable-expression-to-nonnull-field)
* [method returns @Nullable, but superclass method returns @NonNull](#method-returns-nullable-but-superclass-method-returns-nonnull)
* [referenced method returns @Nullable, but functional interface method returns @NonNull](#referenced-method-returns-nullable-but-functional-interface-method-returns-nonnull)
* [parameter is @NonNull, but parameter in superclass method is @Nullable](#parameter-is-nonnull-but-parameter-in-superclass-method-is-nullable)
* [parameter is @NonNull, but parameter in functional interface method is @Nullable](#parameter-is-nonnull-but-parameter-in-functional-interface-method-is-nullable)
* [unbound instance method reference cannot be used, as first parameter of functional interface method is @Nullable](#unbound-instance-method-reference-cannot-be-used-as-first-parameter-of-functional-interface-method-is-nullable)
* [initializer method does not guarantee @NonNull field is initialized / @NonNull field  not initialized](#initializer-method-does-not-guarantee-nonnull-field-is-initialized--nonnull-field--not-initialized)
* [read of @NonNull field before initialization](#read-of-nonnull-field-before-initialization)
* [unboxing of a @Nullable value](#unboxing-of-a-nullable-value)

## Messages

### dereferenced expression is @Nullable

This error occurs when code reads a field, writes a field, or invokes a method on some expression, and that expression might be null.  Example:

```java
Object x = null;
x.toString(); // dereferencing x, which is null
```

To fix this error, you can move the dereference under a check that the expression is not null.  E.g.:

```java
Object x = null;
if (x != null) {
  x.toString();  // this is safe now
}
```

Be sure to add appropriate logic to handle the case where the expression is null!  Or, even better, try to find a way to rewrite the code so the expression can't be null in the first place.

### returning @Nullable expression from method with @NonNull return type

This error occurs when code is returning a `@Nullable` expression from a method whose return type is not annotated as `@Nullable`.  Example:

```java
Object m(@Nullable Object x) {
  return x; // oops!  return type is unannotated, hence @NonNull
}
```

To fix this error, you can add a `@Nullable` annotation to the return type, e.g.:

```java
@Nullable
Object m(@Nullable Object x) {
  return x; 
}
```

Or, you can place the `return` statement under an appropriate null check:

```java
Object m(@Nullable Object x) {
  if (x != null) {
    return x;
  } else {
    return new Object();
  }
  // this also works: return (x == null) ? new Object() : x;
}
```

### passing @Nullable parameter where @NonNull is required

This can happen in cases like the following:
```java
void nonNullParam(Object x) {}
void caller(@Nullable Object y) {
  nonNullParam(y); // bad!
}
```
To fix, either:
* Make sure `y` is non-null at the point `nonNullParam(y)` is called (e.g. by making the parameter to `caller` itself `@NonNull`, or adding an appropriate null check before `nonNullParam(y)` is called), **or**
* Make the parameter `@Nullable` in `nonNullParam`'s definition.

### assigning @Nullable expression to @NonNull field

Here's an example of how this can happen:
```java
class Foo {

  Object myField;  // @NonNull since it's not annotated

  void writeToField(@Nullable Object z) {
    this.myField = z; // bad!
  }
}
```

To fix, either make the right-hand side of the assignment `@NonNull`, place the assignment under an appropriate null check for the right-hand side, or make the field `@Nullable`.

### method returns @Nullable, but superclass method returns @NonNull

This error means you have overridden a superclass method in an invalid way.  Here's an example of why it's bad.

```java
class Super {
  Object getObj() { return new Object(); }
}
class Sub extends Super {
  @Override
  @Nullable 
  getObj() { return null; }
}
class Main {
  void caller() {
    Super x = new Sub();
    x.getObj().toString(); // NullPointerException!
  }
}
```

The key idea is that when code gets an object of type `Super`, it needs to be able to rely on the fact that `Super.getObj()` returns a `@NonNull` value.  If subclassing breaks this guarantee, it can lead to `NullPointerException`s.

### referenced method returns @Nullable, but functional interface method returns @NonNull

This is a version of the previous error specific to method references.  Example:
```java
interface NoArgFunc {
  Object apply();
}
class Test {
  static Object doApply(NoArgFunc f) {
    return f.apply();
  }
  @Nullable
  static Object returnNull() { return null; }
  static void test() {
    doApply(Test::returnNull).toString(); // NullPointerException!
  }
}
```
The key idea is that when code invokes `NoArgFunc.apply()`, it needs to be able to rely on the fact that the return value will be `@NonNull`.  If a method reference breaks this guarantee, it can lead to `NullPointerException`s.

### parameter is @NonNull, but parameter in superclass method is @Nullable

This error is similar to the previous one regarding bad overriding and return types.  Here's an example of why it's needed:
```java
class Super {
  void handleObj(@Nullable Object obj) { }
}
class Sub extends Super {
  @Override
  void handleObj(@NonNull Object obj) { 
    obj.toString(); 
  }
}
class Main {
  void caller() {
    Super x = new Sub();
    x.handleObj(null); // this will cause a NullPointerException
  }
}
```
Subclasses cannot safely override a method and annotate a parameter as `@NonNull` if the overridden method has a `@Nullable` parameter.

### parameter is @NonNull, but parameter in functional interface method is @Nullable

This is just like the previous error, but specific to lambdas and method references.  Example:
```java
interface NullableArgFunc {
  void apply(@Nullable Object o);
}
class Test {
  static void doApply(NullableArgFunc f, @Nullable Object p) {
    f.apply(p);
  }
  static void derefArg(Object q) {
    q.toString();
  }
  static void test() {
    doApply((Object x) -> { print(x.hashCode()); }, null); // NullPointerException
    doApply(Test::derefArg, null); // also NullPointerException
  }
}
```
Lambdas / method references cannot assume a parameter is `@NonNull` if the functional interface declares it as `@Nullable`.

Note that if the first `doApply` call were written as `doApply(x -> { print(x.hashCode()); }`, without the explicit type for the parameter, NullAway would treat the parameter as `@Nullable` (inherited from the functional interface method), and hence would report an error at the dereference `x.hashCode()`.

### unbound instance method reference cannot be used, as first parameter of functional interface method is @Nullable

This is the same as the previous error, specialized to the case of the first parameter of an unbound instance method reference (or a "Reference to an instance method of a arbitrary object supplied later"; see [here](http://baddotrobot.com/blog/2014/02/18/method-references-in-java8/)).  Example:

```java
interface NullableArgFunc<T> {
  void apply(@Nullable T o);
}
class Test {
  static <T> void doApply(NullableArgFunc<T> f, @Nullable T p) {
    f.apply(p);
  }
  void instMethod() {}
  static void test() {
    doApply(Test::instMethod, null); // NullPointerException
  }
}
```
Here, writing `Test::instMethod` is equivalent to writing `x -> { x.instMethod(); }`.  Hence, the `doApply` call will cause an NPE.

### initializer method does not guarantee @NonNull field is initialized / @NonNull field  not initialized

This error indicates that a `@NonNull` field may not be properly initialized in some case.  There are two ways to initialize a `@NonNull` field: (1) in the constructors of a class, or (2) in a designated initializer method.  For the case of constructors, each `@NonNull` field must be initialized in _every_ constructor (a constructor that invokes another constructor via `this(...)` is excluded).  A field can also be initialized in a method called by the constructor, provided that the method is either `private` or `final` (to prevent overriding) and the constructor invokes the method at the top-level, not under an `if` condition (to guarantee the call always executes).  NullAway only detects cases where a field is initialized via a direct call to another method from a constructor, not via a chain of calls.

An example:
```java
class C1 {
  // non-null fields
  Object f1, f2;

  // good constructor
  C1(Object f1) {
    this.f1 = f1;
    // initializing f2 in doInit() is ok, since it's a
    // private method and called at the top-level
    doInit();
  }

  private void doInit() {
    this.f2 = new Object();
  }

  // bad constructor
  C1(boolean flag) {
    if (flag) {      
      // bad: method called under a condition
      doInit();
    }
    // missing initialization of f1
  }
} 
```

Initialization can also occur in special initializer method to handle alternate initialization protocols, like Android lifecycles.  The checker has built-in knowledge of the following initialization methods:
```
  'android.view.View.onFinishInflate',
  'android.app.Service.onCreate',
  'android.app.Activity.onCreate',
  'android.app.Fragment.onCreate',
  'android.app.Application.onCreate',
  'javax.annotation.processing.Processor.init'
```
Overrides of these methods will be treated as initializers.  (If you think any third-party library methods should be added to the list, please file an issue.)  You can specify a method as an initializer with any `@Initializer` annotation, but this should be used very sparingly; it is best not to introduce one-off complex initialization patterns.  As with constructors, initializers can invoke private or final methods at the top level to perform initialization.

Note that for a field to be considered initializer by an initializer method, a non-null value must be assigned to that field over all possible paths through that method. The following code would not be considered to have initialized `foo`, as there is at least one case (due to the early return) where `foo` was not initialized.

```java
class C2 {
  // non-null fields
  Object foo;

  C2() { }

  @Initializer
  public int initMe() {
    if (!sanityCheck()) {
      return -1; // Error code.
    }
    this.foo = new Object();
    return 0
  }
}
```

If you get a field initialization error, you can fix it by ensuring the field is initialized in all cases, or by making the field `@Nullable`.

In addition to any annotation with the `@Initializer` simple name, Null Away recognizes as equivalent "initializer annotations" the following:

```
  'org.junit.Before'
  'org.junit.BeforeClass'
```

As well as any fully-qualified annotation name passed using the `-XepOpt:NullAway:CustomInitializerAnnotations=` configuration option.

### read of @NonNull field before initialization

This error is reported when a `@NonNull` field is used before it is
initialized, e.g.:
```java
class C {

  Object foo;
  
  C() {
    this.foo.toString(); // foo not initialized yet!
    this.foo = new Object();
  }
}
```

To fix this error, perform the initialization before reading the field.

### unboxing of a @Nullable value

This error is reported when some `@Nullable` expression gets [implicitly unboxed](https://docs.oracle.com/javase/tutorial/java/data/autoboxing.html) by some operation.  For example:

```java
Integer i1 = null;
int i2 = i1 + 3; // NullPointerException
```

These errors can be fixed in the same manner as the [dereferenced expression is @Nullable](#dereferenced-expression-is-nullable) error.