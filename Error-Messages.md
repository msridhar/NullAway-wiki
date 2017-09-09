Here we explain the different warning messages that NullAway produces and how to address them.  For an overview of NullAway see [the main README](https://github.com/uber/NullAway/blob/master/README.md).

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
To fix, either make the parameter `@NonNull`, place the method call under an appropriate null check for the parameter, or make the parameter `@Nullable`.

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

### initializer method does not guarantee @NonNull field is initialized / @NonNull field  not initialized

This error indicates that a `@NonNull` field may not be properly initialized in some case.  There are two ways to initialize a `@NonNull` field: (1) in the constructors of a class, or (2) in a designated initializer method.  For the case of constructors, each `@NonNull` field must be initialized in //every// constructor (a constructor that invokes another constructor via `this(...)` is excluded).  A field can also be initialized in a method called by the constructor, provided that the method is either `private` or `final` (to prevent overriding) and the constructor invokes the method at the top-level, not under an `if` condition (to guarantee the call always executes).

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
Overrides of these methods will be treated as initializers.  (If you think any third-party library methods should be added to the list, please file an issue.)  Within our codebase, you can specify a method as an initializer with the `@Initializer` annotation, but this should be used very sparingly; it is best not to introduce one-off complex initialization patterns.  As with constructors, initializers can invoke private or final methods at the top level to perform initialization.

If you get a field initialization error, you can fix it by ensuring the field is initialized in all cases, or by making the field `@Nullable`.
