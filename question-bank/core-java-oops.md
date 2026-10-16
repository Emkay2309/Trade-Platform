# Core Java & OOP

## Q: Java doesn't support multiple inheritance of classes but does of interfaces — what problem does diamond inheritance actually cause, and how do Java 8 default methods partially reintroduce it? How is it resolved?
**Asked/relevant on:** 2026-10-16 (warm-up)
**Your answer:**


**Refined answer:**

Java restricts a class to a single superclass specifically to avoid the classic diamond problem: if class `C` could inherit from both `A` and `B`, and both defined a method `foo()` with conflicting implementations (or conflicting instance state), there would be no unambiguous way to decide which version `C` should use. By allowing only one superclass, Java guarantees every inherited method and field has exactly one unambiguous origin.

Interfaces, by contrast, historically only declared method signatures with no implementation, so implementing several interfaces never created a runtime conflict — there was nothing to collide over.

Java 8 introduced **default methods**, giving interfaces a method body, and this reintroduces a *limited* version of the diamond problem: if interfaces `A` and `B` both declare a default method `foo()` with different implementations, and a class `C` implements both without overriding `foo()`, the compiler raises a hard error:

```
class C inherits unrelated defaults for foo() from types A and B
```

This is a **compile-time error**, not a silent runtime ambiguity — the key difference from C++-style multiple inheritance. Java forces the conflict to be resolved before the code can even build.

**Resolution rules, in order:**
1. A concrete method in a class always wins over any default method from an interface.
2. If one interface extends another and overrides the default method, the more specific interface's version wins.
3. Otherwise, the implementing class must explicitly override the method — and can still choose to delegate to a specific parent:

```java
interface A { default void foo() { System.out.println("A"); } }
interface B { default void foo() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void foo() {
        A.super.foo(); // explicitly resolve which default to use
    }
}
```

**Follow-ups to expect:**
- What happens if two interfaces give the same default method and a class implements both without overriding?
- Why doesn't this same ambiguity happen with fields?
