# Section 1: Core Java — OOP & Language Fundamentals

---

## Basic Questions

---

### What are the four pillars of OOP? Explain each with a one-line real-world analogy.

**Core Explanation:**

The four pillars of OOP are **Encapsulation**, **Abstraction**, **Inheritance**, and **Polymorphism**.

- **Encapsulation**: Bundling data (fields) and behavior (methods) together and restricting direct access to internals. *Analogy: A car hides its engine internals — you just use the steering wheel and pedals.*
- **Abstraction**: Exposing only what is necessary, hiding implementation details. *Analogy: A TV remote shows buttons but hides the circuit logic inside.*
- **Inheritance**: A class inherits fields and behavior from a parent class, enabling reuse. *Analogy: A child inherits traits from a parent.*
- **Polymorphism**: One interface, many implementations. *Analogy: A "draw" command behaves differently for Circle, Rectangle, and Triangle.*

**Practical Example:**
```java
// Encapsulation
public class BankAccount {
    private double balance; // hidden
    public void deposit(double amount) { balance += amount; }
    public double getBalance() { return balance; }
}

// Abstraction
public abstract class Shape {
    public abstract double area(); // what, not how
}

// Inheritance
public class Circle extends Shape {
    private double radius;
    public Circle(double r) { this.radius = r; }
    @Override
    public double area() { return Math.PI * radius * radius; }
}

// Polymorphism
Shape s = new Circle(5); // Shape reference, Circle behavior
System.out.println(s.area()); // dynamic dispatch
```

**Edge Cases / Pitfalls:**
- Don't confuse abstraction (hiding complexity) with encapsulation (hiding data). They work together but serve different purposes.
- Overusing inheritance creates tight coupling. Prefer composition over inheritance when there's no true "is-a" relationship.

**Best Practices:**
- Use interfaces + composition to achieve polymorphism without deep inheritance trees.
- Keep fields private and expose only necessary behavior through public methods.

**Common Follow-Up Questions:**
- *What is the difference between abstraction and encapsulation?* Abstraction is about **what** (interface/concept); encapsulation is about **how** (hiding data).
- *Why is composition preferred over inheritance?* Inheritance is compile-time and creates tight coupling; composition is more flexible and testable.

---

### What is the difference between an abstract class and an interface? When would you choose one over the other?

**Core Explanation:**

| Feature | Abstract Class | Interface |
|---|---|---|
| Instantiation | Cannot be instantiated | Cannot be instantiated |
| Methods | Can have abstract + concrete methods | All methods abstract by default (Java 8+: can have `default`, `static`) |
| Fields | Can have instance fields | Only `public static final` constants |
| Constructors | Yes | No |
| Multiple inheritance | Single only | A class can implement multiple |
| Access modifiers | Any | Methods are `public` by default |

**Choose abstract class when:**
- You want to share code (concrete methods) among related classes.
- Classes share state (fields).
- You have a true "is-a" relationship with a common base.

**Choose interface when:**
- You need to define a contract that unrelated classes can implement.
- You need multiple inheritance of type.
- You're defining capabilities (e.g., `Serializable`, `Comparable`).

**Practical Example:**
```java
// Abstract class — shared state and behavior
public abstract class Vehicle {
    protected String brand;
    public void startEngine() { System.out.println("Vroom"); } // shared
    public abstract void refuel(); // subclass must implement
}

// Interface — capability contract
public interface Flyable {
    void fly();
    default void land() { System.out.println("Landing"); }
}

public class FlyingCar extends Vehicle implements Flyable {
    @Override public void refuel() { System.out.println("Refueling"); }
    @Override public void fly() { System.out.println("Flying"); }
}
```

**Edge Cases / Pitfalls:**
- Java 8 added `default` methods to interfaces, blurring the line. But interfaces still can't have instance fields or constructors.
- A class can `extend` only one abstract class but `implement` multiple interfaces — this matters for framework design.

**Best Practices:**
- Use interfaces to define APIs; use abstract classes for partial implementations.
- In modern Java, prefer interfaces with `default` methods for most abstractions.

**Common Follow-Up Questions:**
- *Can an interface extend another interface?* Yes, and a class that implements it must implement all methods from both.
- *Can an abstract class implement an interface without implementing all methods?* Yes — the concrete subclass must implement remaining methods.

---

### What is the difference between method overloading and method overriding?

**Core Explanation:**

| | Overloading | Overriding |
|---|---|---|
| Definition | Same method name, different parameters | Subclass redefines a superclass method |
| Resolution | Compile-time (static dispatch) | Runtime (dynamic dispatch) |
| Inheritance required | No | Yes |
| Return type | Can differ (but not sufficient alone) | Must be same or covariant |
| Access modifier | Can change freely | Cannot be more restrictive |
| `@Override` | Not used | Should always use |

**Practical Example:**
```java
// Overloading — compile-time resolution
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
}

// Overriding — runtime resolution
public class Animal {
    public void speak() { System.out.println("..."); }
}
public class Dog extends Animal {
    @Override
    public void speak() { System.out.println("Woof"); }
}

Animal a = new Dog();
a.speak(); // prints "Woof" — runtime polymorphism
```

**Edge Cases / Pitfalls:**
- You **cannot** override `static` methods — calling them on a superclass reference always calls the superclass version (method hiding, not overriding).
- Overloading can cause ambiguity with `null` arguments: `add(null)` when both `add(String)` and `add(Object)` exist.
- Changing only return type is NOT valid overloading — it causes a compile error.

**Best Practices:**
- Always use `@Override` annotation when overriding — it catches mistakes at compile time.
- Avoid overloading with ambiguous parameter types (e.g., `int` vs `Integer`).

**Common Follow-Up Questions:**
- *Can you override a private method?* No — private methods are not visible to subclasses, so they cannot be overridden.
- *What is covariant return type?* An overriding method can return a subtype of the parent's return type (e.g., parent returns `Animal`, child can return `Dog`).

---

### What is a constructor? What are its types and what restrictions does it have?

**Core Explanation:**

A constructor is a special method called when an object is created (`new ClassName()`). It initializes the object's state. Constructors have the same name as the class, no return type, and are not inherited.

**Types:**
1. **Default constructor**: No-arg constructor; auto-generated by compiler if no constructor is defined.
2. **Parameterized constructor**: Accepts arguments to initialize fields.
3. **Copy constructor**: Accepts another object of the same class (Java doesn't have built-in copy constructors, but you can write one).
4. **Private constructor**: Prevents external instantiation (used in Singleton, utility classes).

**Restrictions:**
- Cannot be `static`, `abstract`, `final`, or `synchronized`.
- Cannot have a return type.
- `this()` (call another constructor) or `super()` (call parent constructor) must be the **first statement**.

**Practical Example:**
```java
public class Person {
    private String name;
    private int age;

    // Default constructor
    public Person() { this("Unknown", 0); } // calls parameterized

    // Parameterized constructor
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // Copy constructor
    public Person(Person other) {
        this.name = other.name;
        this.age = other.age;
    }
}
```

**Edge Cases / Pitfalls:**
- If you define any constructor, the compiler no longer generates the default one. This breaks code that does `new MyClass()`.
- `this()` and `super()` calls cannot both appear in the same constructor (only one can be first).

**Best Practices:**
- Use constructor chaining (`this(...)`) to avoid code duplication.
- Use the Builder pattern when there are many optional fields.

**Common Follow-Up Questions:**
- *Is a constructor inherited?* No, constructors are not inherited. Subclass constructors must explicitly call `super()`.
- *Can a constructor throw an exception?* Yes — any checked or unchecked exception can be thrown from a constructor.

---

### What is the purpose of the `static` keyword? Can a top-level class be declared `static`?

**Core Explanation:**

`static` means "belongs to the class, not to any instance." Static members are shared across all instances.

**Uses of `static`:**
- **Static fields**: Class-level variables, shared state (e.g., counters, constants).
- **Static methods**: Utility/helper methods that don't need instance state (e.g., `Math.abs()`).
- **Static blocks**: Initialization code that runs once when the class is loaded.
- **Static nested classes**: Nested class not tied to the outer class instance.
- **Static imports**: Import static members directly.

**Can a top-level class be `static`?** No. Only **nested classes** (inner classes) can be `static`. A top-level class cannot be declared `static` — it would be a compile error.

**Practical Example:**
```java
public class Counter {
    private static int count = 0; // shared across all instances
    private int id;

    public Counter() {
        count++;
        this.id = count;
    }

    public static int getCount() { return count; } // no 'this'

    static {
        System.out.println("Counter class loaded"); // runs once
    }

    // Static nested class
    public static class Config {
        String name = "default";
    }
}
```

**Edge Cases / Pitfalls:**
- Static methods cannot access instance fields directly — they don't have a `this` reference.
- Overusing `static` makes code hard to test (can't mock static methods easily without frameworks like Mockito's `mockStatic`).
- Static fields can cause memory leaks if they hold references to objects (e.g., static `List` growing indefinitely).

**Best Practices:**
- Use `static final` for constants (prefer over magic numbers).
- Avoid mutable static state in multithreaded applications.

**Common Follow-Up Questions:**
- *Can a static method be synchronized?* Yes — it locks on the `Class` object, not an instance.
- *What is the difference between a static nested class and an inner class?* Inner class holds an implicit reference to the outer instance; static nested class does not.

---

### What is the difference between `==` and `equals()`?

**Core Explanation:**

- `==` compares **references** (memory addresses) for objects; for primitives, it compares **values**.
- `equals()` compares **content** — by default it's the same as `==`, but most classes override it to compare meaningful fields.

**Practical Example:**
```java
String a = new String("hello");
String b = new String("hello");

System.out.println(a == b);       // false — different objects
System.out.println(a.equals(b));  // true — same content

Integer x = 127;
Integer y = 127;
System.out.println(x == y); // true — Integer cache (-128 to 127)

Integer p = 200;
Integer q = 200;
System.out.println(p == q); // false — outside cache range
```

**Edge Cases / Pitfalls:**
- **String literals** are interned in the String Pool: `"hello" == "hello"` is `true`. `new String("hello") == "hello"` is `false`.
- **Integer caching**: Java caches `Integer` values from -128 to 127. Using `==` on cached integers appears to work but breaks outside that range.
- `null.equals(x)` throws `NullPointerException`. Prefer `Objects.equals(a, b)` for null-safe comparison.

**Best Practices:**
- Always use `equals()` for object content comparison.
- For null-safe comparison: `Objects.equals(a, b)`.
- For comparing strings to literals: `"literal".equals(variable)` (Yoda conditions prevent NPE).

**Common Follow-Up Questions:**
- *What is the contract of `equals()`?* It must be reflexive, symmetric, transitive, consistent, and `x.equals(null)` must return `false`.
- *If you override `equals()`, what else must you override?* `hashCode()` — objects that are equal must have the same hash code.

---

### What is the difference between `final`, `finally`, and `finalize()`?

**Core Explanation:**

These three are unrelated keywords that share similar spelling.

| Keyword | Context | Purpose |
|---|---|---|
| `final` | Keyword | Makes variable/method/class unchangeable |
| `finally` | Exception handling | Block that always executes after `try/catch` |
| `finalize()` | Method | Called by GC before object is garbage collected (deprecated) |

**Practical Example:**
```java
// final
final int MAX = 100;       // constant — cannot reassign
final class Immutable {}   // cannot be subclassed
final void doSomething() {} // cannot be overridden

// finally
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("Caught");
} finally {
    System.out.println("Always runs"); // even with return/exception
}

// finalize() — deprecated since Java 9
@Override
protected void finalize() throws Throwable {
    // cleanup before GC — unreliable, avoid
    super.finalize();
}
```

**Edge Cases / Pitfalls:**
- `finally` does NOT run if `System.exit()` is called or the JVM crashes.
- A `final` reference variable means the reference can't change, but the object it points to is still mutable.
- `finalize()` is unreliable — GC may never call it, or call it much later. **Do not use for resource cleanup.**

**Best Practices:**
- Use `final` for constants, immutable references, and to prevent subclassing.
- Use `finally` for cleanup (closing connections) — or better, use try-with-resources.
- Replace `finalize()` with `java.lang.ref.Cleaner` or implement `AutoCloseable`.

**Common Follow-Up Questions:**
- *Can a `finally` block override a `return` value in a `try` block?* Yes — if `finally` has a `return` statement, it overrides the try's return.
- *Can a `final` field be set in a constructor?* Yes — it must be assigned exactly once, and a constructor is the right place.

---

### What is the difference between `String`, `StringBuilder`, and `StringBuffer`?

**Core Explanation:**

| | `String` | `StringBuilder` | `StringBuffer` |
|---|---|---|---|
| Mutability | Immutable | Mutable | Mutable |
| Thread-safe | Yes (immutable) | No | Yes (synchronized) |
| Performance | Slow for concatenation | Fast | Slower than StringBuilder |
| Since | Java 1.0 | Java 5 | Java 1.0 |

**Practical Example:**
```java
// String — immutable, new object created on every concatenation
String s = "Hello";
s += " World"; // creates a NEW String object

// StringBuilder — mutable, NOT thread-safe
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World").append("!"); // modifies same object
System.out.println(sb.toString()); // "Hello World!"

// StringBuffer — thread-safe but slower
StringBuffer sbf = new StringBuffer("Hello");
sbf.append(" World"); // synchronized
```

**Benchmark insight:**
```java
// BAD — creates 10,000 intermediate String objects
String result = "";
for (int i = 0; i < 10_000; i++) result += i;

// GOOD — O(n) vs O(n²)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10_000; i++) sb.append(i);
String result = sb.toString();
```

**Edge Cases / Pitfalls:**
- The Java compiler automatically converts string concatenation (`+`) to `StringBuilder` in simple cases, but NOT inside loops — hence the performance trap.
- `StringBuffer` is rarely needed in modern code. `StringBuilder` + external synchronization is preferred.

**Best Practices:**
- Use `String` for constants and when mutability is not needed.
- Use `StringBuilder` for string building in single-threaded contexts.
- Avoid `StringBuffer` unless you specifically need thread-safe string building.

**Common Follow-Up Questions:**
- *Why is `String` immutable?* Security (class names, file paths), thread safety, String Pool caching, hashCode caching.
- *Does Java compile `s + "literal"` to StringBuilder?* Yes for simple cases, but not inside loops — always use StringBuilder in loops.

---

### What is the String Pool? How does `intern()` work?

**Core Explanation:**

The **String Pool** (also called String Intern Pool) is a special memory area in the JVM heap where literal strings are stored. When you write `"hello"`, Java checks if that string already exists in the pool. If it does, it returns the existing reference. This saves memory.

- `String literals` are automatically added to the pool.
- `new String("hello")` creates a new object on the heap — **not** in the pool.
- `intern()` forces a heap string into the pool (or returns the existing pooled version).

**Practical Example:**
```java
String a = "hello";              // goes to String Pool
String b = "hello";              // reuses same pool object
String c = new String("hello");  // new heap object, NOT pooled

System.out.println(a == b);         // true — same pool reference
System.out.println(a == c);         // false — different objects
System.out.println(a == c.intern()); // true — intern() returns pool ref
System.out.println(a.equals(c));    // true — same content
```

**Edge Cases / Pitfalls:**
- Calling `intern()` on very many strings can fill the pool and cause performance issues.
- In Java 7+, the String Pool was moved to the heap (from PermGen), so it can be garbage collected when strings are no longer referenced.

**Best Practices:**
- Rely on literal strings for pooling automatically.
- Use `intern()` sparingly — mainly for memory optimization with very large numbers of duplicate strings (e.g., parsing CSV with repeated field names).
- Don't rely on `==` for string comparison — always use `equals()`.

**Common Follow-Up Questions:**
- *Where is the String Pool located in Java 7+?* In the main heap (moved from PermGen in Java 7), so strings can be GC'd.
- *What happens when you do `"a" + "b"` at compile time?* The compiler optimizes it to `"ab"` — a single pool entry.

---

### What is autoboxing and unboxing?

**Core Explanation:**

- **Autoboxing**: Automatic conversion of a primitive to its wrapper type (`int` → `Integer`).
- **Unboxing**: Automatic conversion of a wrapper to its primitive (`Integer` → `int`).

Java performs these conversions implicitly to allow primitives to be used in collections and generic types.

**Practical Example:**
```java
// Autoboxing
Integer boxed = 42;              // int → Integer
List<Integer> list = new ArrayList<>();
list.add(5);                     // int 5 → Integer.valueOf(5)

// Unboxing
int value = boxed;               // Integer → int
int sum = list.get(0) + 10;     // Integer → int, then arithmetic

// Dangerous unboxing — NullPointerException!
Integer nullable = null;
int x = nullable; // NullPointerException at runtime
```

**Edge Cases / Pitfalls:**
- **NPE on unboxing null**: If a wrapper is null and you unbox it, you get a `NullPointerException`.
- **Performance in loops**: Autoboxing in tight loops creates many short-lived objects and stresses GC.
- **Unexpected `==` behavior**: `Integer a = 200; Integer b = 200; a == b` is `false` (different objects outside cache range -128 to 127).

**Best Practices:**
- Use primitives for performance-critical code (arithmetic, large arrays).
- Null-check wrapper types before unboxing.
- Prefer `int` over `Integer` in method parameters when null is not a valid value.

**Common Follow-Up Questions:**
- *What is the Integer cache?* Java caches `Integer` values from -128 to 127. Autoboxed integers in this range return the same object.
- *How does autoboxing affect `==` comparisons?* Always use `.equals()` for wrapper type comparison.

---

### What is the difference between checked and unchecked exceptions?

**Core Explanation:**

| | Checked | Unchecked |
|---|---|---|
| Superclass | `Exception` (not `RuntimeException`) | `RuntimeException` or `Error` |
| Compile-time enforcement | Yes — must catch or declare with `throws` | No |
| Examples | `IOException`, `SQLException`, `ClassNotFoundException` | `NullPointerException`, `ArrayIndexOutOfBoundsException`, `IllegalArgumentException` |
| Typical cause | External resources (files, DB, network) | Programming bugs |

**Practical Example:**
```java
// Checked — must handle at compile time
public void readFile(String path) throws IOException {
    BufferedReader reader = new BufferedReader(new FileReader(path));
    // compiler forces you to handle IOException
}

// Unchecked — no compile-time requirement
public int divide(int a, int b) {
    return a / b; // throws ArithmeticException if b == 0, no declaration needed
}
```

**Edge Cases / Pitfalls:**
- Swallowing checked exceptions with empty `catch {}` blocks silently hides bugs.
- Converting checked exceptions to unchecked (wrapping in `RuntimeException`) is common in streams and lambdas — be careful not to lose context.
- `Error` (like `OutOfMemoryError`) is also unchecked but represents JVM-level problems — don't catch `Error` unless you know exactly what you're doing.

**Best Practices:**
- Use checked exceptions for recoverable conditions (file not found, network timeout).
- Use unchecked exceptions for programming errors (null passed where not allowed, invalid state).
- Wrap checked exceptions into a custom `RuntimeException` with meaningful context when crossing API boundaries.

**Common Follow-Up Questions:**
- *Should you use checked or unchecked exceptions in your own APIs?* Unchecked (Runtime) exceptions are generally preferred in modern Java — they're less verbose and don't force callers to handle errors they can't recover from.
- *What is exception chaining?* Preserving the original cause: `throw new ServiceException("Failed", e)`.

---

### What is the difference between `throw` and `throws`?

**Core Explanation:**

- **`throw`**: Used to explicitly throw an exception at runtime inside a method body.
- **`throws`**: Used in a method signature to declare that the method might throw a checked exception — it's a warning to callers.

**Practical Example:**
```java
// throws — declaration in signature
public void processOrder(Order order) throws OrderException {
    if (order == null) {
        throw new OrderException("Order cannot be null"); // throw — actual throwing
    }
    // process...
}

// throws multiple
public void connect() throws IOException, SQLException {
    // ...
}
```

**Edge Cases / Pitfalls:**
- `throws` is only required for **checked** exceptions. Unchecked exceptions can be thrown without declaring them (though you can declare them if it helps documentation).
- `throw` takes a single `Throwable` instance; you cannot `throw` a class, only an object.
- If a method declares `throws Exception` (too broad), it forces callers to catch the most generic type, losing specificity.

**Best Practices:**
- Declare the most specific exception type in `throws`.
- Avoid declaring `throws Exception` or `throws Throwable` — be specific.
- Use `throw` with a meaningful message: `throw new IllegalArgumentException("Amount must be positive, got: " + amount)`.

**Common Follow-Up Questions:**
- *Can you `throw` a checked exception without declaring it in `throws`?* No — the compiler enforces this for checked exceptions.
- *Can you `throw null`?* Technically `throw null` compiles, but throws `NullPointerException` at runtime.

---

### What is try-with-resources and why was it introduced?

**Core Explanation:**

Try-with-resources (Java 7+) automatically closes resources that implement `AutoCloseable` when the try block exits — whether normally or via an exception. It was introduced to eliminate the verbose and error-prone pattern of closing resources in `finally` blocks.

**Practical Example:**
```java
// Before Java 7 — verbose and error-prone
Connection conn = null;
try {
    conn = dataSource.getConnection();
    // use conn
} finally {
    if (conn != null) {
        try { conn.close(); } catch (SQLException e) { /* swallowed */ }
    }
}

// Java 7+ — clean, guaranteed close
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT 1")) {
    // use conn and ps
} // both automatically closed in reverse order: ps, then conn
```

**Implementing AutoCloseable:**
```java
public class DatabaseSession implements AutoCloseable {
    public DatabaseSession() { System.out.println("Opened"); }

    @Override
    public void close() { System.out.println("Closed"); }
}

try (DatabaseSession session = new DatabaseSession()) {
    // use session
} // "Closed" is printed automatically
```

**Edge Cases / Pitfalls:**
- If both the `try` block and `close()` throw exceptions, the exception from `close()` is **suppressed** (accessible via `getSuppressed()`).
- Resources are closed in **reverse** order of declaration.
- The resource variable is effectively final — you cannot reassign it inside the try block.

**Best Practices:**
- Always use try-with-resources for `Connection`, `InputStream`, `OutputStream`, `PreparedStatement`, and similar resources.
- Implement `AutoCloseable` (not just `Closeable`) for your own resources.

**Common Follow-Up Questions:**
- *What is the difference between `Closeable` and `AutoCloseable`?* `Closeable` extends `AutoCloseable` and restricts `close()` to throw only `IOException`. `AutoCloseable.close()` can throw any `Exception`.
- *Does try-with-resources work without a catch block?* Yes — the resource is still closed. You can combine with catch and finally.

---

### What is the difference between a process and a thread?

**Core Explanation:**

| | Process | Thread |
|---|---|---|
| Definition | A running program with its own memory space | A unit of execution within a process |
| Memory | Isolated — separate heap, stack, code | Shared heap; each has its own stack |
| Communication | IPC (pipes, sockets, shared memory) | Shared memory directly (with synchronization) |
| Creation cost | Heavy — OS allocates new memory | Light — shares process resources |
| Crash impact | Crashes don't affect other processes | A thread crash can affect the whole process |
| Example | JVM instance, browser tab | HTTP request handler, background job |

**Practical Example:**
```java
// Each Java application runs in a JVM process
// Within that JVM, you create threads:
Thread t = new Thread(() -> System.out.println("Thread running"));
t.start();

// Multiple threads share the same heap:
List<String> shared = new ArrayList<>(); // on heap — accessible by all threads
// Each thread has its own stack:
void method() { int local = 5; } // local is on the thread's stack — NOT shared
```

**Edge Cases / Pitfalls:**
- Java threads share the heap, so shared mutable state requires synchronization.
- In Java, you don't create OS processes directly (though `ProcessBuilder` can spawn child processes).
- Java 21 virtual threads are much lighter than OS threads, but they still share the same JVM heap.

**Best Practices:**
- Use thread pools (`ExecutorService`) rather than creating raw threads.
- For isolation, use separate processes (e.g., microservices). For shared state with concurrency, use threads with proper synchronization.

**Common Follow-Up Questions:**
- *Can threads share data?* Yes — all threads in a Java process share the heap. Primitives and objects on the heap are accessible to all threads.
- *What is the difference between a daemon thread and a user thread?* Daemon threads (e.g., GC thread) are background threads; the JVM exits when all user threads complete, regardless of daemon threads.

---

### What is typecasting? What is the difference between widening and narrowing?

**Core Explanation:**

Typecasting is converting a value from one type to another.

- **Widening (implicit)**: Converting a smaller type to a larger type. Safe — no data loss. Done automatically by the compiler.
- **Narrowing (explicit)**: Converting a larger type to a smaller type. Can lose data. Requires explicit cast.

**Type hierarchy (widening direction):** `byte → short → int → long → float → double`

**Practical Example:**
```java
// Widening — automatic
int i = 100;
long l = i;         // int → long, automatic
double d = i;       // int → double, automatic

// Narrowing — explicit cast required
double pi = 3.14159;
int truncated = (int) pi;  // 3 — fractional part lost!

long bigNum = 130L;
byte b = (byte) bigNum;    // data loss — 130 overflows byte (-126)

// Object casting
Object obj = "Hello";        // widening reference cast — always safe
String str = (String) obj;   // narrowing reference cast — ClassCastException if wrong type
```

**Edge Cases / Pitfalls:**
- Narrowing can cause **overflow** (e.g., `(byte) 130` = `-126`) and **truncation** (decimal parts lost).
- Reference narrowing throws `ClassCastException` at runtime if the actual object isn't the target type.
- Use `instanceof` before narrowing reference casts.

**Best Practices:**
- Always use `instanceof` (or pattern matching `instanceof` in Java 16+) before casting object references.
- Be explicit about narrowing conversions — name the variable to make the truncation obvious.

```java
// Java 16+ pattern matching instanceof
if (obj instanceof String s) {
    System.out.println(s.length()); // no explicit cast needed
}
```

**Common Follow-Up Questions:**
- *Does widening always work?* Almost — `int` to `float` can lose precision (e.g., large `int` values may not be representable exactly as `float`).
- *What is autoboxing widening?* Autoboxing (`int → Integer`) is separate from widening — they don't chain automatically.

---

## Intermediate Questions

---

### What are the SOLID principles? Explain each with a Java code-level example.

**Core Explanation:**

SOLID is an acronym for five object-oriented design principles that lead to maintainable, flexible code.

**S — Single Responsibility Principle (SRP):** A class should have one and only one reason to change.
```java
// BAD — OrderService does too much
class OrderService {
    void processOrder(Order o) { /* business logic */ }
    void saveOrder(Order o) { /* DB logic */ }       // SRP violation
    void sendConfirmationEmail(Order o) { /* email */ } // SRP violation
}

// GOOD
class OrderService { void processOrder(Order o) {} }
class OrderRepository { void save(Order o) {} }
class EmailService { void sendConfirmation(Order o) {} }
```

**O — Open/Closed Principle (OCP):** Open for extension, closed for modification.
```java
// BAD — add new shape = modify existing code
double area(Shape s) {
    if (s instanceof Circle) return Math.PI * ...;
    if (s instanceof Rectangle) return ...;  // must modify for every new shape
}

// GOOD
interface Shape { double area(); }
class Circle implements Shape { public double area() { return Math.PI * r * r; } }
class Rectangle implements Shape { public double area() { return w * h; } }
```

**L — Liskov Substitution Principle (LSP):** Subtypes must be substitutable for their base types without altering correctness.
```java
// BAD — Square extends Rectangle but breaks LSP
class Rectangle { int width, height; void setWidth(int w) {width=w;} }
class Square extends Rectangle {
    @Override void setWidth(int w) { width = height = w; } // unexpected side effect
}

// GOOD — use composition or separate hierarchy
```

**I — Interface Segregation Principle (ISP):** Clients should not be forced to depend on interfaces they don't use.
```java
// BAD — fat interface
interface Worker { void work(); void eat(); void sleep(); }

// GOOD — segregated
interface Workable { void work(); }
interface Eatable { void eat(); }
class Robot implements Workable { public void work() {} } // Robot doesn't eat
```

**D — Dependency Inversion Principle (DIP):** High-level modules should not depend on low-level modules — both should depend on abstractions.
```java
// BAD
class OrderService {
    MySQLOrderRepository repo = new MySQLOrderRepository(); // hard dependency
}

// GOOD
class OrderService {
    private final OrderRepository repo; // depends on abstraction
    OrderService(OrderRepository repo) { this.repo = repo; } // injected
}
```

**Best Practices:**
- Apply these principles progressively — don't over-engineer small classes.
- SOLID + DI containers (Spring) naturally enforce DIP.

---

### What is the difference between abstraction and encapsulation in terms of design intent?

**Core Explanation:**

- **Abstraction**: Focuses on **what** an object does, hiding implementation details. It's about designing interfaces and contracts.
- **Encapsulation**: Focuses on **how** to protect internal state by restricting direct access to fields.

They work together but serve different purposes: abstraction hides complexity at the design level; encapsulation protects data at the implementation level.

**Practical Example:**
```java
// Abstraction — PaymentGateway defines WHAT, not HOW
public interface PaymentGateway {
    boolean charge(String customerId, double amount);
}

// Encapsulation — StripeGateway protects its internal state
public class StripeGateway implements PaymentGateway {
    private final String apiKey;      // hidden
    private HttpClient client;        // hidden

    public StripeGateway(String apiKey) { this.apiKey = apiKey; }

    @Override
    public boolean charge(String customerId, double amount) {
        // internal implementation hidden from callers
        return client.post("/charge", buildRequest(customerId, amount));
    }
}
```

**Edge Cases / Pitfalls:**
- You can have encapsulation without abstraction (private fields in a concrete class).
- You can have abstraction without full encapsulation (public interface with no state hiding concern).

**Common Follow-Up Questions:**
- *Which comes first?* Design abstraction first (what should this object do?), then implement encapsulation (how to protect its state).

---

### How does Java resolve which overloaded method to call at compile time vs which overridden method at runtime?

**Core Explanation:**

Java uses **two phases** of method resolution:
1. **Compile-time (static dispatch)**: Determines which overloaded version to call based on the declared (reference) type and argument types.
2. **Runtime (dynamic dispatch)**: Determines which overridden version to call based on the actual (runtime) type of the object.

**Practical Example:**
```java
class Animal {
    void sound(Animal a) { System.out.println("Animal sound(Animal)"); }
    void sound(Dog d) { System.out.println("Animal sound(Dog)"); }
}

class Dog extends Animal {
    @Override
    void sound(Animal a) { System.out.println("Dog sound(Animal)"); }
}

Animal a = new Dog();
Dog d = new Dog();

a.sound(d); // Output: "Dog sound(Animal)"
// Compile-time: reference type = Animal, arg type = Dog → selects sound(Animal) [no sound(Dog) in Dog's compiled view]
// Wait: Animal has sound(Dog), so compiler picks sound(Dog) based on declared arg type...
// Actually: declared ref type = Animal → picks sound(Animal) via overloading (Dog IS-A Animal)
// Runtime: actual type = Dog → calls Dog.sound(Animal)
```

More clearly:
```java
class Printer {
    void print(Object o) { System.out.println("Object: " + o); }
    void print(String s) { System.out.println("String: " + s); }
}

Object obj = "hello"; // declared as Object, actual is String

new Printer().print(obj);   // "Object: hello" — compile picks based on declared type (Object)
new Printer().print("hi");  // "String: hi" — compile picks based on declared type (String)
```

**Key rule:**
- Overloading resolution: decided at **compile time** using declared types → selects overload.
- Override resolution: decided at **runtime** using actual object type → dispatches to the correct subclass method.

---

### What is the `equals()` and `hashCode()` contract? What breaks if you violate it?

**Core Explanation:**

The contract:
1. If `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` **must** be true.
2. The reverse is NOT required: equal hash codes don't mean equal objects (hash collision is OK).
3. `hashCode()` must return the same value consistently across calls during a program's execution.

**What breaks if violated:**
- If you override `equals()` but NOT `hashCode()`, HashMap/HashSet will use the default `hashCode()` (based on memory address). Two logically equal objects will have different hash codes, so they'll land in different buckets — you can put an object in a `HashSet` and then `contains()` will return `false` for an equal object.

**Practical Example:**
```java
// BROKEN — equals without hashCode
class User {
    String name;
    User(String name) { this.name = name; }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof User)) return false;
        return name.equals(((User) o).name);
    }
    // Missing hashCode!
}

Set<User> set = new HashSet<>();
set.add(new User("Alice"));
System.out.println(set.contains(new User("Alice"))); // FALSE! (different hash codes)

// FIXED
@Override
public int hashCode() {
    return Objects.hash(name);
}
```

**Best Practices:**
- Use `Objects.hash(field1, field2, ...)` for a clean implementation.
- Include the same fields in both `equals()` and `hashCode()`.
- Use IDE generation or Lombok `@EqualsAndHashCode`.

---

### What is immutability? Write the steps to make a class with a mutable `List` field fully immutable.

**Core Explanation:**

An **immutable** object's state cannot change after construction. All fields are final, no setters exist, and mutable fields are defensively copied.

**Steps to make a class immutable:**
1. Declare the class `final` (prevent subclassing).
2. Declare all fields `private final`.
3. No setters.
4. Initialize all fields in the constructor.
5. **Defensively copy** mutable fields on input AND return copies on output.

**Practical Example:**
```java
public final class ImmutableOrder {
    private final String orderId;
    private final List<String> items; // List is mutable!

    public ImmutableOrder(String orderId, List<String> items) {
        this.orderId = orderId;
        // Step 5a: defensive copy on input
        this.items = Collections.unmodifiableList(new ArrayList<>(items));
    }

    public String getOrderId() { return orderId; }

    public List<String> getItems() {
        // Step 5b: return unmodifiable view (already done in constructor)
        return items;
    }
}

// Test
List<String> original = new ArrayList<>(List.of("a", "b"));
ImmutableOrder order = new ImmutableOrder("O1", original);
original.add("c");                          // modifying original doesn't affect order
System.out.println(order.getItems());       // [a, b] — protected
order.getItems().add("d");                  // throws UnsupportedOperationException
```

**Common Follow-Up Questions:**
- *Is `String` immutable in Java?* Yes — that's why it's safe to use in HashMap keys and thread-safe without synchronization.
- *What is a shallow vs deep immutable object?* If a field contains mutable objects, you need deep copies for true immutability.

---

### Why is `String` immutable? How does immutability help in multithreading, caching, and security?

**Core Explanation:**

`String` is immutable by design for several critical reasons:

1. **String Pool**: Immutable strings can be safely shared across the JVM. If they were mutable, changing a pooled string would affect all references.
2. **Thread safety**: Immutable objects need no synchronization — multiple threads can read the same `String` concurrently without locks.
3. **HashMap/HashSet keys**: `String`'s `hashCode()` is computed once and cached. Mutable keys would corrupt hash-based collections.
4. **Security**: Class names, file paths, and network addresses are `Strings`. If they were mutable, malicious code could change them after security checks.

**Practical Example:**
```java
// Security scenario
void loadClass(String className) {
    securityCheck(className); // check "com.trusted.MyClass"
    // If String were mutable, another thread could modify className here!
    Class.forName(className); // would load a different class
}

// Thread safety — safe to share without synchronization
String config = "jdbc:mysql://localhost/mydb";
// Multiple threads can read 'config' safely — no mutex needed

// HashMap key caching
String key = "userId";
int hash1 = key.hashCode(); // computed and cached
key = key + "123"; // creates a new String — original hash still valid
```

**Common Follow-Up Questions:**
- *How is `String` immutability implemented in the JVM?* Via `private final char[]` (Java 8 and below) or `private final byte[]` (Java 9+ with compact strings). The internal array is never exposed.
- *Can reflection break String immutability?* Yes — reflection can access the private `value` array, but this is a misuse and bypasses Java's memory model guarantees.

---

### What is serialization and deserialization? What risks does Java serialization introduce?

**Core Explanation:**

- **Serialization**: Converting an object's state to a byte stream (for storage or transmission).
- **Deserialization**: Reconstructing an object from a byte stream.

Classes must implement `Serializable` (marker interface). A `serialVersionUID` should be declared to avoid version mismatch errors.

**Practical Example:**
```java
import java.io.*;

public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password; // NOT serialized

    // constructors, getters...
}

// Serialize
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.ser"));
oos.writeObject(new User("Alice", "secret"));

// Deserialize
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.ser"));
User user = (User) ois.readObject();
```

**Risks of Java serialization:**
1. **Security vulnerabilities (deserialization attacks)**: Malicious byte streams can trigger arbitrary code execution via gadget chains (Log4Shell-style attacks, Apache Commons deserialization exploits).
2. **Version incompatibility**: Without `serialVersionUID`, changing the class breaks deserialization of old data.
3. **Performance**: Java serialization is slow and produces large byte streams compared to JSON or Protobuf.
4. **Tight coupling**: Serialized format is tied to Java internals.

**Best Practices:**
- Prefer JSON (Jackson), Protocol Buffers, or Avro over Java serialization.
- Never deserialize data from untrusted sources using native Java serialization.
- Use `serialVersionUID` explicitly.
- Mark sensitive fields as `transient`.

---

### What is the difference between `transient` and `volatile` keywords?

**Core Explanation:**

| | `transient` | `volatile` |
|---|---|---|
| Purpose | Excludes field from serialization | Ensures visibility of changes across threads |
| Context | Serialization | Multithreading / Java Memory Model |
| Effect on value | Field is not written to stream; set to default on deserialization | Reads/writes go directly to main memory, bypassing CPU caches |
| Thread safety | Not related to threads | Guarantees visibility but NOT atomicity |

**Practical Example:**
```java
public class Session implements Serializable {
    private String userId;         // serialized
    private transient String token; // NOT serialized (security)
    private volatile boolean active; // thread-safe visibility flag
}

// volatile in action
class Service {
    private volatile boolean running = true;

    void stop() { running = false; } // written to main memory immediately
    void process() {
        while (running) { // always reads from main memory
            // work...
        }
    }
}
```

**Edge Cases / Pitfalls:**
- `volatile` does NOT make compound operations atomic: `count++` on a volatile is still a race condition (read-modify-write).
- `transient` fields get their default values on deserialization (`null` for objects, `0` for primitives).

---

### What is the difference between shallow copy and deep copy? How do you implement each?

**Core Explanation:**

- **Shallow copy**: Creates a new object but copies only the references to nested objects — the copy and original share mutable sub-objects.
- **Deep copy**: Creates a new object AND recursively creates new copies of all nested objects — fully independent.

**Practical Example:**
```java
class Address {
    String city;
    Address(String city) { this.city = city; }
}

class Person implements Cloneable {
    String name;
    Address address;

    Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    // Shallow copy
    @Override
    protected Person clone() throws CloneNotSupportedException {
        return (Person) super.clone(); // address reference is SHARED
    }

    // Deep copy
    protected Person deepCopy() {
        return new Person(this.name, new Address(this.address.city));
    }
}

Person original = new Person("Alice", new Address("NYC"));
Person shallow = original.clone();
Person deep = original.deepCopy();

shallow.address.city = "LA";
System.out.println(original.address.city); // "LA" — shared reference!

deep.address.city = "SF";
System.out.println(original.address.city); // "LA" — independent
```

**Best Practices:**
- Prefer copy constructors or static factory methods over `clone()`.
- For complex deep copies, use serialization/deserialization or a mapping library.

---

### What is a marker interface? How does `Serializable` work without any methods?

**Core Explanation:**

A **marker interface** is an interface with no methods or constants. It's used to convey metadata about a class to the JVM or framework. The JVM or library checks `instanceof` to decide behavior.

`Serializable` works because `ObjectOutputStream.writeObject()` internally checks `if (!(obj instanceof Serializable)) throw new NotSerializableException(...)`. The interface itself does no work — it just marks the class as "safe to serialize."

**Practical Example:**
```java
// Custom marker interface
public interface Auditable {}

// Framework checks for it
public class AuditLogger {
    public void log(Object entity) {
        if (entity instanceof Auditable) {
            // log the audit trail
            System.out.println("Auditing: " + entity.getClass().getSimpleName());
        }
    }
}

// Modern alternative: annotations (preferred in Java 5+)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Auditable {}
```

**Common Follow-Up Questions:**
- *Why are marker interfaces still used today?* `Serializable`, `Cloneable`, and `RandomAccess` are legacy. Modern Java prefers annotations for metadata.
- *What is the difference between `Cloneable` and `Serializable`?* Both are marker interfaces, but `Cloneable` changes the behavior of `Object.clone()`, while `Serializable` affects `ObjectOutputStream`.

---

### What are generics? What is type erasure and how does it affect runtime behavior?

**Core Explanation:**

**Generics** (Java 5+) allow classes and methods to be parameterized by type, enabling type-safe collections and algorithms without casting.

**Type erasure**: Generic type parameters exist only at compile time. The compiler uses them for type checking, then **erases** them — replacing type parameters with their bounds (`Object` or the upper bound). At runtime, there are no generic types.

**Practical Example:**
```java
// Compile time
List<String> strings = new ArrayList<>();
strings.add("hello");
// strings.add(42); // compile error

// Runtime — after type erasure, it's just List
// The bytecode looks like: List list = new ArrayList(); list.add("hello");

// Consequences of type erasure:
List<String> strList = new ArrayList<>();
List<Integer> intList = new ArrayList<>();
System.out.println(strList.getClass() == intList.getClass()); // TRUE — same class

// Cannot do at runtime:
// if (list instanceof List<String>) // compile error
// new T() // cannot instantiate generic type
// T[] array = new T[10] // cannot create generic arrays

// Bounded type parameters
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}
```

**Edge Cases / Pitfalls:**
- **Heap pollution**: Mixing raw types and generics can cause `ClassCastException` at unexpected points.
- You cannot create instances of type parameters (`new T()`) or arrays (`new T[10]`).
- Reflection works on the erased type, not the generic type.

---

### What happens if an exception is thrown inside a `finally` block?

**Core Explanation:**

If an exception is thrown inside a `finally` block, it **replaces** (suppresses) any exception that was being propagated from the `try` or `catch` blocks. The original exception is lost.

**Practical Example:**
```java
public void riskyMethod() throws Exception {
    try {
        throw new Exception("Original exception");
    } finally {
        throw new Exception("Finally exception"); // REPLACES original!
    }
}
// Caller sees: "Finally exception" — the original is gone!

// How to preserve the original:
public void safeMethod() throws Exception {
    Exception original = null;
    try {
        throw new Exception("Original");
    } catch (Exception e) {
        original = e;
        throw e;
    } finally {
        try {
            // risky cleanup
        } catch (Exception finallyEx) {
            if (original != null) {
                original.addSuppressed(finallyEx); // attach, don't replace
            }
            // don't re-throw finallyEx
        }
    }
}
```

**Best Practices:**
- Avoid throwing exceptions from `finally` blocks.
- Use try-with-resources — it handles suppressed exceptions automatically.

---

### What is exception propagation? How does it differ for checked vs unchecked exceptions?

**Core Explanation:**

Exception propagation is the process of an exception moving up the call stack until it's caught or reaches the top (causing program termination).

- **Unchecked exceptions**: Propagate automatically without any declaration requirement. Any method in the chain that doesn't catch it gets it passed up.
- **Checked exceptions**: Must be explicitly declared in each method's `throws` clause as they propagate. Every method in the chain must either catch or declare it.

**Practical Example:**
```java
// Unchecked — propagates silently
void a() { b(); }
void b() { c(); }
void c() { throw new RuntimeException("error"); } // propagates through b() and a()

// Checked — each method must declare
void a() throws IOException { b(); }            // must declare
void b() throws IOException { c(); }            // must declare
void c() throws IOException { throw new IOException("error"); }
```

---

### How do you design custom exception hierarchies for a layered Spring Boot application?

**Core Explanation:**

A well-designed exception hierarchy mirrors the application layers and maps to HTTP responses cleanly.

**Practical Example:**
```java
// Base application exception
public abstract class AppException extends RuntimeException {
    private final String errorCode;
    protected AppException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }
    public String getErrorCode() { return errorCode; }
}

// Domain/business exceptions
public class OrderNotFoundException extends AppException {
    public OrderNotFoundException(String orderId) {
        super("Order not found: " + orderId, "ORDER_NOT_FOUND");
    }
}

public class InsufficientInventoryException extends AppException {
    public InsufficientInventoryException(String product) {
        super("Insufficient inventory for: " + product, "INSUFFICIENT_INVENTORY");
    }
}

// Infrastructure exceptions
public class DatabaseException extends AppException {
    public DatabaseException(String msg, Throwable cause) {
        super(msg, "DB_ERROR");
        initCause(cause);
    }
}

// Global handler maps to HTTP
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(OrderNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }

    @ExceptionHandler(InsufficientInventoryException.class)
    public ResponseEntity<ErrorResponse> handle(InsufficientInventoryException ex) {
        return ResponseEntity.status(409).body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }
}
```

---

### What is association, aggregation, and composition? How are they different at the JVM level?

**Core Explanation:**

These describe relationships between objects with varying degrees of coupling:

| Relationship | Dependency | Lifecycle | Example |
|---|---|---|---|
| **Association** | Loose — just uses | Independent | Teacher uses Classroom |
| **Aggregation** | Has-a — weak | Independent | Department has Employees |
| **Composition** | Has-a — strong | Dependent | House has Rooms (room dies with house) |

**JVM-level difference:**
- Association/Aggregation: objects can exist independently; the reference can be null or reassigned.
- Composition: the child object is typically created inside the parent and not shared.

```java
// Association — uses, but lives independently
class Driver { void drive(Car car) {} } // car exists outside Driver

// Aggregation — has-a, but Employee can exist without Department
class Department { List<Employee> employees; } // Employee can outlive Department

// Composition — strong ownership
class House {
    private final Room livingRoom; // created by House, destroyed with House
    House() { this.livingRoom = new Room("Living"); }
}
```

---

### Can a static method be overridden? Explain method hiding with an example.

**Core Explanation:**

**No**, static methods cannot be overridden. They can be **hidden** (method hiding). The difference: method hiding uses compile-time type resolution; method overriding uses runtime type resolution.

**Practical Example:**
```java
class Parent {
    static void staticMethod() { System.out.println("Parent static"); }
    void instanceMethod() { System.out.println("Parent instance"); }
}

class Child extends Parent {
    static void staticMethod() { System.out.println("Child static"); } // HIDING
    @Override
    void instanceMethod() { System.out.println("Child instance"); } // OVERRIDING
}

Parent p = new Child();
p.staticMethod();   // "Parent static" — compile-time type decides (method hiding)
p.instanceMethod(); // "Child instance" — runtime type decides (overriding)
```

---

### What is a DTO? Why should you never expose JPA entities directly in REST responses?

**Core Explanation:**

A **DTO (Data Transfer Object)** is a plain object used to carry data between layers, particularly between the service/controller layer and the client. It shapes what the API exposes, decoupling the persistence model from the API contract.

**Why not expose JPA entities directly:**
1. **Lazy loading traps**: Accessing lazy-loaded relations outside a transaction causes `LazyInitializationException`.
2. **Circular references**: Bidirectional JPA mappings cause infinite JSON serialization loops.
3. **Security**: You may accidentally expose sensitive fields (passwords, internal IDs).
4. **Tight coupling**: Any schema change breaks the API contract.
5. **Over-fetching**: Entities often carry more data than the client needs.

```java
// Entity — persistence concern
@Entity
class User {
    @Id Long id;
    String email;
    String passwordHash; // NEVER expose this!
    @OneToMany List<Order> orders; // lazy — risky to serialize
}

// DTO — API concern
record UserDTO(Long id, String email) {}

// Mapping
UserDTO toDTO(User user) {
    return new UserDTO(user.getId(), user.getEmail());
}
```

---

### What is the difference between Java IO (blocking) and NIO (non-blocking)?

**Core Explanation:**

| | Java IO (`java.io`) | Java NIO (`java.nio`) |
|---|---|---|
| Model | Blocking — thread waits for I/O | Non-blocking — thread can do other work |
| Streams | Stream-oriented (byte by byte) | Buffer-oriented (block operations) |
| Selectors | No | Yes — single thread can monitor multiple channels |
| Performance | Good for simple use cases | Better for high-concurrency servers |
| API | `InputStream`, `OutputStream`, `Reader`, `Writer` | `Channel`, `Buffer`, `Selector` |

```java
// Blocking IO
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) { // blocks until data available
        process(line);
    }
}

// NIO — non-blocking with Selector (simplified)
Selector selector = Selector.open();
ServerSocketChannel channel = ServerSocketChannel.open();
channel.configureBlocking(false); // non-blocking mode
channel.register(selector, SelectionKey.OP_ACCEPT);
// Single thread can handle many connections via selector
```

---

## Advanced Questions

---

### Why is catching generic `Exception` or `Throwable` considered harmful? What should you catch instead?

**Core Explanation:**

Catching `Exception` or `Throwable` is harmful because:
1. You catch exceptions you don't know how to handle (e.g., `OutOfMemoryError`, `StackOverflowError`, `InterruptedException`).
2. It hides bugs — programming errors (`NullPointerException`, `IllegalArgumentException`) get silently swallowed.
3. You lose specificity in error handling and logging.
4. `InterruptedException` especially must not be swallowed — it causes threads to ignore interrupt signals.

```java
// BAD
try {
    processRequest(request);
} catch (Exception e) { // catches EVERYTHING including bugs
    log.error("Error", e);
    return "error";
}

// GOOD — specific handlers
try {
    processRequest(request);
} catch (OrderNotFoundException e) {
    return ResponseEntity.notFound().build();
} catch (ValidationException e) {
    return ResponseEntity.badRequest().body(e.getMessage());
} catch (ServiceException e) {
    log.error("Service failure", e);
    return ResponseEntity.internalServerError().build();
}
```

**What should you catch instead?**
- Catch the **most specific exception** you can actually handle.
- Let unhandled exceptions propagate to a global handler.
- If you must catch broadly, rethrow after logging: `catch (Exception e) { log.error(...); throw e; }`.

---

### What is reflection? How does Spring use it internally, and why do performance-critical libraries avoid it?

**Core Explanation:**

**Reflection** (`java.lang.reflect`) allows inspecting and manipulating classes, methods, and fields at runtime — bypassing compile-time type checking.

**How Spring uses reflection:**
- Dependency injection: finds `@Autowired` constructors/fields and injects dependencies.
- `@Transactional`, `@Cacheable`: Creates CGLIB proxies by subclassing beans and intercepting method calls.
- `@Value`, `@ConfigurationProperties`: Injects configuration into private fields.

**Why performance-critical libraries avoid it:**
- Reflective method calls are 10-50x slower than direct calls (JIT can't inline them easily).
- Reflection bypasses JVM optimizations (escape analysis, inlining).
- Reflection requires security manager checks (additional overhead).

```java
// Reflective access
Class<?> clazz = Class.forName("com.example.UserService");
Method method = clazz.getDeclaredMethod("processUser", String.class);
method.setAccessible(true); // bypass private access
Object result = method.invoke(serviceInstance, "alice");

// Spring alternative — annotation processing at startup (not every call)
// Spring caches reflection results at startup to avoid repeated lookups
```

**Modern alternatives to reflection:**
- `MethodHandles` (faster, JIT-friendly)
- Annotation processors (compile-time code generation, e.g., Lombok, MapStruct)
- GraalVM native image requires declaring reflection usage ahead of time.

---

### Why is `finalize()` deprecated? What are the alternatives (`Cleaner`, `PhantomReference`)?

**Core Explanation:**

`finalize()` (deprecated Java 9, removed Java 18) is called by the GC before collecting an object. It's unreliable because:
1. No guaranteed timing — GC may never call it, or call it much later.
2. Causes the object to be **resurrected** (added back to finalizer queue), delaying collection.
3. Exceptions in `finalize()` are silently ignored.
4. Finalizers run on a single finalizer thread — a bottleneck that can cause memory pressure.

**Alternatives:**

**1. `AutoCloseable` + try-with-resources** (preferred for deterministic cleanup):
```java
public class FileResource implements AutoCloseable {
    private FileInputStream fis;
    public FileResource(String path) throws IOException { fis = new FileInputStream(path); }
    @Override public void close() throws IOException { fis.close(); }
}
try (FileResource r = new FileResource("data.txt")) { /* use r */ }
```

**2. `java.lang.ref.Cleaner`** (Java 9+, for non-deterministic cleanup):
```java
public class NativeResource {
    private static final Cleaner cleaner = Cleaner.create();
    private final Cleaner.Cleanable cleanable;

    public NativeResource(long pointer) {
        cleanable = cleaner.register(this, new CleanupAction(pointer));
    }

    private static class CleanupAction implements Runnable {
        private final long pointer;
        CleanupAction(long pointer) { this.pointer = pointer; }
        @Override public void run() { freeNativeMemory(pointer); } // called by GC
    }
}
```

---

### What is the cost of autoboxing in tight loops? How can it cause `OutOfMemoryError`?

**Core Explanation:**

Each autoboxing operation creates a new object on the heap. In a tight loop, this creates millions of short-lived objects, pressuring GC and potentially causing `OutOfMemoryError` if objects accumulate faster than GC can collect them.

```java
// Terrible — creates 1,000,000 Integer objects
Long sum = 0L; // Long, not long!
for (int i = 0; i < 1_000_000; i++) {
    sum += i; // unbox sum, add i (autobox i to Integer internally), rebox to Long
}

// Fixed — zero boxing
long sum = 0L;
for (int i = 0; i < 1_000_000; i++) {
    sum += i; // pure primitive arithmetic
}

// OOM scenario — infinite accumulation
List<Long> cache = new ArrayList<>();
while (true) {
    for (int i = 0; i < 10_000; i++) {
        cache.add((long) i); // 10,000 Long objects per iteration, never GC'd
    }
    // OOM eventually
}
```

Use `int[]`, `long[]`, or specialized collections like `IntStream`, Eclipse Collections primitives, or Trove for performance-critical code.

---

### What are the 12-Factor App principles? Which ones directly influence how you write Java microservices?

**Core Explanation:**

The **12-Factor App** is a methodology for building scalable, maintainable SaaS applications. The most relevant for Java microservices:

1. **Codebase** (1 repo, many deploys) → One Git repo per microservice.
2. **Dependencies** (explicit declaration) → `pom.xml` / `build.gradle` — no implicit classpath dependencies.
3. **Config** (store in environment) → Use `application.yml` + env vars; never hardcode DB URLs. In Spring: `@ConfigurationProperties`.
4. **Backing services** (treat as attached resources) → Database, Kafka, Redis are external — swappable via config.
5. **Build, Release, Run** (strict separation) → CI/CD pipeline: build JAR → release with config → run container.
6. **Processes** (stateless) → No in-memory session. Store state in Redis/DB. Enables horizontal scaling.
7. **Port binding** (self-contained) → Spring Boot's embedded Tomcat: `server.port=8080`.
8. **Concurrency** (scale via processes) → Kubernetes Horizontal Pod Autoscaler, not thread pools.
9. **Disposability** (fast startup, graceful shutdown) → Spring Boot graceful shutdown + liveness/readiness probes.
10. **Dev/prod parity** → Use Docker + Testcontainers to match prod environment in tests.
11. **Logs** (event streams) → Output to stdout; use ELK or CloudWatch to aggregate.
12. **Admin processes** (run as one-off tasks) → DB migrations (Flyway), data fixes as separate jobs, not embedded in the app.

---

### How does Java achieve platform independence? What are the limitations of "write once, run anywhere"?

**Core Explanation:**

Java achieves platform independence via:
1. **Bytecode**: `javac` compiles `.java` to `.class` files (bytecode) — a platform-neutral intermediate representation.
2. **JVM abstraction**: Each OS has a specific JVM implementation that interprets/compiles bytecode to native machine code at runtime.

**Limitations:**
1. **Native code** (`JNI`/`JNA`): Calls to OS-specific native libraries are NOT portable.
2. **GUI applications**: Swing/JavaFX may render differently across OSes.
3. **File paths**: `/` vs `\` separator (use `File.separator` or `Path` API).
4. **Performance variations**: JVM behavior (GC, JIT) can differ across implementations (OpenJDK vs GraalVM vs IBM J9).
5. **Unicode edge cases**: File system encoding differences.
6. **Unsigned integers**: Java has no `unsigned` types; interacting with C/OS APIs requires careful casting.
7. **GraalVM native images**: Ahead-of-time compiled — excellent startup but NOT "run anywhere" (platform-specific binary).

---

### What is the Dependency Inversion Principle? How does Spring's IoC container implement it?

**Core Explanation:**

DIP states:
- High-level modules should not depend on low-level modules — both should depend on abstractions.
- Abstractions should not depend on details — details should depend on abstractions.

**Spring's implementation:** Spring's IoC (Inversion of Control) container inverts who creates and wires dependencies. Instead of classes creating their own dependencies (`new MySQLRepo()`), Spring injects them via constructor/setter.

```java
// Without DIP — high-level depends on low-level
class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository(); // tight coupling
}

// With DIP — both depend on abstraction
interface OrderRepository { void save(Order o); }

class MySQLOrderRepository implements OrderRepository { /* ... */ }
class InMemoryOrderRepository implements OrderRepository { /* ... */ }

@Service
class OrderService {
    private final OrderRepository repo; // depends on abstraction

    @Autowired
    OrderService(OrderRepository repo) { this.repo = repo; } // Spring injects
}

// Spring context
@Configuration
class AppConfig {
    @Bean
    OrderRepository orderRepository() { return new MySQLOrderRepository(); }
}
```

---

### Explain a scenario where violating the Liskov Substitution Principle caused a real bug.

**Core Explanation:**

LSP: If `S` is a subtype of `T`, then objects of type `T` may be replaced with objects of type `S` without breaking program correctness.

**Classic violation — Square extends Rectangle:**
```java
class Rectangle {
    int width, height;
    void setWidth(int w) { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}

class Square extends Rectangle {
    @Override
    void setWidth(int w) { this.width = this.height = w; } // enforces square constraint
    @Override
    void setHeight(int h) { this.width = this.height = h; }
}

// Code that worked with Rectangle is BROKEN with Square:
void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(3);
    assert r.area() == 15; // FAILS with Square (area is 9, not 15)
}

Rectangle r = new Square(); // caller thinks it's a Rectangle
resize(r); // assertion failure — LSP violated!
```

**Real-world scenario:**
An e-commerce platform had `ReadOnlyRepository` extending `CrudRepository`. The `ReadOnlyRepository` threw `UnsupportedOperationException` for `save()` and `delete()`. Generic code that accepted `CrudRepository` and called `save()` failed at runtime with `UnsupportedOperationException`. The fix: `ReadOnlyRepository` should NOT extend `CrudRepository`. Instead, define a separate `Readable` interface.
# 2. Core Java — Design Patterns

---

## Basic

---

### What is the Singleton pattern? Write a basic implementation.

**Core Explanation**

The Singleton pattern restricts a class to exactly one instance and provides a global point of access to it. It is a creational pattern used when a single shared resource (configuration, connection pool, registry) must coordinate actions across the system.

**Practical Example**

```java
public class AppConfig {

    private static AppConfig instance;

    private final Properties properties;

    private AppConfig() {
        properties = new Properties();
        // load from file, environment, etc.
    }

    public static AppConfig getInstance() {
        if (instance == null) {
            instance = new AppConfig();
        }
        return instance;
    }

    public String get(String key) {
        return properties.getProperty(key);
    }
}
```

> This basic (lazy) version is **not thread-safe**. See the Advanced section for thread-safe variants.

**Edge Cases / Pitfalls**

- The simple lazy version above will create multiple instances in a multithreaded environment.
- Singletons introduce hidden global state, making unit testing harder unless you design for it (e.g., provide a `resetForTest()` method or use dependency injection).
- Serialization and reflection can break the single-instance guarantee (covered in Advanced).

**Best Practices**

Use an Enum Singleton or the Bill Pugh holder idiom for production code. Prefer dependency injection frameworks (Spring `@Component` with default singleton scope) over hand-rolled singletons.

**Common Follow-Up Questions**

1. **How do you make this thread-safe?** -- Use `synchronized`, double-checked locking, enum, or the static inner class holder pattern.
2. **How does Spring manage singletons differently?** -- Spring maintains a singleton *per ApplicationContext* in a `ConcurrentHashMap`, not via a private constructor; the class itself is not forced to be a singleton.
3. **When should you avoid Singleton?** -- When the object carries mutable shared state that complicates testing, or when multiple instances are needed for different configurations.

---

### What is the Factory pattern? Where have you seen it used in the JDK or Spring?

**Core Explanation**

The Factory Method pattern defines an interface for creating objects but lets subclasses or helper methods decide which class to instantiate. It decouples client code from concrete implementations, centralises creation logic, and makes it easy to add new types without modifying existing code.

**Practical Example**

```java
// Product interface
public interface Notification {
    void send(String message);
}

// Concrete products
public class EmailNotification implements Notification {
    public void send(String message) { System.out.println("Email: " + message); }
}

public class SmsNotification implements Notification {
    public void send(String message) { System.out.println("SMS: " + message); }
}

// Factory
public class NotificationFactory {

    public static Notification create(String channel) {
        return switch (channel.toLowerCase()) {
            case "email" -> new EmailNotification();
            case "sms"   -> new SmsNotification();
            default -> throw new IllegalArgumentException("Unknown channel: " + channel);
        };
    }
}

// Client
Notification n = NotificationFactory.create("email");
n.send("Hello!");
```

**JDK and Spring Examples**

| Location | Factory Usage |
|---|---|
| `Calendar.getInstance()` | Returns a locale-specific `Calendar` subclass |
| `NumberFormat.getInstance()` | Returns a formatter for the default locale |
| `DriverManager.getConnection()` | Returns a vendor-specific `Connection` |
| `BeanFactory` / `ApplicationContext` | Spring's IoC container is fundamentally a factory |
| `LoggerFactory.getLogger()` (SLF4J) | Returns the underlying logging implementation |

**Edge Cases / Pitfalls**

- A factory with dozens of `if-else` or `switch` branches becomes a maintenance burden; combine with a registry (`Map<String, Supplier<T>>`) to keep it open for extension.
- Returning `null` instead of throwing on unknown input leads to `NullPointerException` downstream. Always throw or return an `Optional`.

**Best Practices**

Favour a `Map<String, Supplier<T>>` registry inside the factory for extensibility. Keep the factory stateless so it can be shared safely.

**Common Follow-Up Questions**

1. **What is the difference between Factory Method and Abstract Factory?** -- Factory Method uses a single method (or class) to create one product; Abstract Factory provides a family of related products (see Intermediate section).
2. **How does Spring's `FactoryBean` differ from `BeanFactory`?** -- `FactoryBean<T>` is a special bean whose `getObject()` creates and returns another bean; `BeanFactory` is the IoC container interface itself.
3. **When would you prefer a constructor over a factory?** -- When there is only one concrete type and no creation logic; factories add indirection you may not need.

---

### What is the Builder pattern? When is it preferred over a constructor?

**Core Explanation**

The Builder pattern separates the construction of a complex object from its representation, allowing step-by-step construction with a fluent API. It solves the *telescoping constructor* problem (many overloaded constructors with different parameter combinations).

**Practical Example**

```java
public class HttpRequest {

    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;

    private HttpRequest(Builder builder) {
        this.url       = builder.url;
        this.method    = builder.method;
        this.headers   = Map.copyOf(builder.headers);   // immutable copy
        this.body      = builder.body;
        this.timeoutMs = builder.timeoutMs;
    }

    public static class Builder {
        // Required
        private final String url;

        // Optional with defaults
        private String method = "GET";
        private Map<String, String> headers = new LinkedHashMap<>();
        private String body;
        private int timeoutMs = 5000;

        public Builder(String url) {
            this.url = Objects.requireNonNull(url);
        }

        public Builder method(String method)              { this.method = method; return this; }
        public Builder header(String key, String value)    { this.headers.put(key, value); return this; }
        public Builder body(String body)                   { this.body = body; return this; }
        public Builder timeoutMs(int timeoutMs)            { this.timeoutMs = timeoutMs; return this; }

        public HttpRequest build() {
            // validation can happen here
            if (body != null && "GET".equalsIgnoreCase(method)) {
                throw new IllegalStateException("GET requests must not have a body");
            }
            return new HttpRequest(this);
        }
    }
}

// Usage
HttpRequest req = new HttpRequest.Builder("https://api.example.com/users")
        .method("POST")
        .header("Content-Type", "application/json")
        .body("{\"name\":\"Alice\"}")
        .timeoutMs(3000)
        .build();
```

**When to Prefer Builder over Constructor**

| Scenario | Use Builder? |
|---|---|
| More than 3-4 parameters | Yes |
| Many optional parameters | Yes |
| Object must be immutable after creation | Yes |
| Construction requires validation across multiple fields | Yes |
| Simple value object with 1-2 fields | No -- constructor or `record` is fine |

**Edge Cases / Pitfalls**

- A mutable Builder can be reused to build multiple objects; make sure shared mutable collections are defensively copied in `build()`.
- Lombok's `@Builder` generates builder code automatically but hides the API from javadoc; ensure the team understands what is generated.

**Best Practices**

Put required parameters in the Builder constructor and optional ones in fluent setters. Validate invariants inside `build()`, not in the outer class constructor.

**Common Follow-Up Questions**

1. **How does Lombok `@Builder` work internally?** -- It generates a static inner `Builder` class with fluent setters and a `build()` method via annotation processing at compile time.
2. **Can you combine Builder with the Factory pattern?** -- Yes. A factory method can internally use a builder: `OrderFactory.createRushOrder()` returns a pre-configured builder result.
3. **How does `StringBuilder` relate to the Builder pattern?** -- It is a simplified builder that accumulates state (`append`) and produces a result (`toString`), though it does not follow the GoF structure exactly.

---

### Why are design patterns useful? Can you over-apply them?

**Core Explanation**

Design patterns provide *proven, named solutions* to recurring software design problems. Their value is threefold:

1. **Shared vocabulary** -- saying "we used a Strategy here" instantly communicates intent to the whole team.
2. **Reduced design risk** -- the trade-offs of each pattern are well-documented.
3. **Improved maintainability** -- patterns promote SOLID principles (single responsibility, open/closed, etc.).

**Can You Over-Apply Them?**

Absolutely. Over-application is one of the most common mistakes:

```java
// Over-engineered: AbstractSingletonProxyFactoryBean is a real Spring class name
// that became a meme for pattern abuse.

// Simple task: return a greeting based on time of day.

// OVER-ENGINEERED approach
interface GreetingStrategy { String greet(); }
class MorningGreeting implements GreetingStrategy { ... }
class EveningGreeting implements GreetingStrategy { ... }
class GreetingStrategyFactory { ... }
class GreetingService {
    GreetingService(GreetingStrategyFactory factory) { ... }
}

// APPROPRIATE approach
public String greet(LocalTime time) {
    return time.getHour() < 12 ? "Good morning" : "Good evening";
}
```

**Signs of Over-Application**

- A pattern is added "just in case" future requirements need it (speculative generality).
- The pattern introduces more classes than the problem has concepts.
- A simple `if-else` is replaced by five classes and two interfaces with no realistic prospect of new branches.
- Reading the code requires jumping through multiple indirection layers to understand a single flow.

**Best Practices**

Apply patterns when they solve a *current* pain point, not a hypothetical future one. Revisit and refactor: introduce a pattern when the code starts violating SOLID, not before.

**Common Follow-Up Questions**

1. **Which patterns do you use most often?** -- Strategy, Factory, Builder, and Observer cover the vast majority of real-world needs.
2. **How do you decide which pattern to apply?** -- Identify the design problem first (e.g., "too many conditional branches" -> Strategy; "complex construction" -> Builder), then evaluate trade-offs.
3. **Are there anti-patterns?** -- Yes. God Object, Service Locator (misused), and Singleton abuse are common anti-patterns that superficially resemble legitimate patterns.

---

## Intermediate

---

### What is the difference between Factory, Abstract Factory, and Builder patterns?

**Core Explanation**

| Aspect | Factory Method | Abstract Factory | Builder |
|---|---|---|---|
| **Purpose** | Create *one* product | Create a *family* of related products | Construct a complex object step-by-step |
| **Abstraction** | A single method decides the concrete type | An interface with multiple factory methods, one per product type | A fluent API accumulating configuration |
| **Result** | One object | Multiple related objects (e.g., Button + Checkbox for a UI theme) | One fully-configured object |
| **When to use** | Type decision based on input/config | Families that must be used together | Many optional parameters, immutable objects |

**Practical Example**

```java
// --- Factory Method ---
public static Connection createConnection(String dbType) {
    return switch (dbType) {
        case "mysql"    -> new MySqlConnection();
        case "postgres" -> new PostgresConnection();
        default -> throw new IllegalArgumentException(dbType);
    };
}

// --- Abstract Factory ---
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}
public class DarkThemeFactory implements UIFactory {
    public Button createButton()     { return new DarkButton(); }
    public Checkbox createCheckbox() { return new DarkCheckbox(); }
}
public class LightThemeFactory implements UIFactory {
    public Button createButton()     { return new LightButton(); }
    public Checkbox createCheckbox() { return new LightCheckbox(); }
}

// --- Builder ---
Pizza pizza = new Pizza.Builder("thin")
        .cheese(true)
        .pepperoni(true)
        .size(Size.LARGE)
        .build();
```

**Edge Cases / Pitfalls**

- Abstract Factory becomes unwieldy if the product family keeps growing (every new product type requires a method in every factory).
- Builder is sometimes confused with Abstract Factory; the key difference is *step-by-step configuration* vs. *family-of-products*.

**Best Practices**

Use Factory Method for simple single-type decisions, Abstract Factory when multiple related objects must be consistent, and Builder when construction complexity is the primary concern.

**Common Follow-Up Questions**

1. **Can you combine these patterns?** -- Yes. A factory can return a pre-configured builder, or an abstract factory can internally use builders for complex products.
2. **Where does Spring use Abstract Factory?** -- `BeanFactory` is conceptually an abstract factory; `FactoryBean` implementations serve as specialised factories for individual beans.

---

### What is the Strategy pattern? Give an example where you replaced `if-else` chains with it.

**Core Explanation**

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable at runtime. The client delegates behaviour to a strategy interface rather than implementing conditional logic internally.

**Practical Example -- Before (if-else chain)**

```java
public class PricingService {

    public BigDecimal calculateDiscount(String customerType, BigDecimal price) {
        if ("REGULAR".equals(customerType)) {
            return price.multiply(new BigDecimal("0.05"));
        } else if ("PREMIUM".equals(customerType)) {
            return price.multiply(new BigDecimal("0.15"));
        } else if ("VIP".equals(customerType)) {
            return price.multiply(new BigDecimal("0.25"));
        } else {
            return BigDecimal.ZERO;
        }
    }
}
```

**After (Strategy pattern)**

```java
// Strategy interface
@FunctionalInterface
public interface DiscountStrategy {
    BigDecimal apply(BigDecimal price);
}

// Concrete strategies (can also be lambdas)
public class RegularDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) {
        return price.multiply(new BigDecimal("0.05"));
    }
}

public class PremiumDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) {
        return price.multiply(new BigDecimal("0.15"));
    }
}

// Registry + Factory to resolve the right strategy
public class DiscountStrategyFactory {

    private static final Map<String, DiscountStrategy> STRATEGIES = Map.of(
        "REGULAR", new RegularDiscount(),
        "PREMIUM", new PremiumDiscount(),
        "VIP",     price -> price.multiply(new BigDecimal("0.25"))   // lambda
    );

    public static DiscountStrategy forType(String customerType) {
        return Optional.ofNullable(STRATEGIES.get(customerType))
                .orElseThrow(() -> new IllegalArgumentException(
                        "No strategy for: " + customerType));
    }
}

// Client
public class PricingService {

    public BigDecimal calculateDiscount(String customerType, BigDecimal price) {
        return DiscountStrategyFactory.forType(customerType).apply(price);
    }
}
```

**Edge Cases / Pitfalls**

- If you only have 2-3 branches that will never grow, a simple `switch` is clearer than introducing a full Strategy hierarchy.
- Stateful strategies must not be shared across threads unless they are thread-safe.

**Best Practices**

Leverage Java's functional interfaces (`Function`, `Predicate`, `Comparator`) as lightweight strategies before creating dedicated interfaces. Combine with a Map-based factory for open/closed compliance.

**Common Follow-Up Questions**

1. **How does Strategy differ from State?** -- Strategy selects an algorithm; State changes the object's behaviour as its internal state changes. Strategies are usually stateless; State objects often trigger transitions.
2. **How does `Comparator` relate to Strategy?** -- `Comparator` is the JDK's classic Strategy example: `Collections.sort(list, comparator)` delegates the comparison algorithm.
3. **Can Spring inject strategies automatically?** -- Yes. Define strategies as `@Component` beans and inject them as a `Map<String, DiscountStrategy>` or a `List<DiscountStrategy>`, then resolve at runtime.

---

### What is the Observer pattern? How does Spring's event system (`ApplicationEvent`) use it?

**Core Explanation**

The Observer pattern defines a one-to-many dependency: when one object (the *subject* / *publisher*) changes state, all its dependents (*observers* / *subscribers*) are notified automatically. This decouples the producer of events from the consumers.

**Practical Example -- Plain Java**

```java
// Observer interface
public interface OrderEventListener {
    void onOrderPlaced(Order order);
}

// Subject
public class OrderService {

    private final List<OrderEventListener> listeners = new CopyOnWriteArrayList<>();

    public void addListener(OrderEventListener listener) {
        listeners.add(listener);
    }

    public void placeOrder(Order order) {
        // ... business logic ...
        listeners.forEach(l -> l.onOrderPlaced(order));
    }
}

// Concrete observers
public class InventoryUpdater implements OrderEventListener {
    public void onOrderPlaced(Order order) {
        // reduce stock
    }
}

public class EmailNotifier implements OrderEventListener {
    public void onOrderPlaced(Order order) {
        // send confirmation email
    }
}
```

**Spring's Event System**

Spring implements the Observer pattern through `ApplicationEventPublisher` and `@EventListener`:

```java
// Event (POJO since Spring 4.2; no longer must extend ApplicationEvent)
public record OrderPlacedEvent(String orderId, BigDecimal total) {}

// Publisher
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ApplicationEventPublisher publisher;

    public void placeOrder(Order order) {
        // ... save order ...
        publisher.publishEvent(new OrderPlacedEvent(order.getId(), order.getTotal()));
    }
}

// Listeners (observers)
@Component
public class InventoryListener {

    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // reduce stock for event.orderId()
    }
}

@Component
public class NotificationListener {

    @Async
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // send email asynchronously
    }
}
```

**Edge Cases / Pitfalls**

- By default, Spring events are **synchronous** -- a slow listener blocks the publisher. Use `@Async` (with `@EnableAsync`) for fire-and-forget.
- An exception in one listener prevents subsequent listeners from executing unless you configure a custom `ApplicationEventMulticaster` with error handling.
- `@TransactionalEventListener(phase = AFTER_COMMIT)` ensures the listener only fires after the transaction commits.

**Best Practices**

Use Spring's built-in event system instead of hand-rolling observer lists. Prefer `@TransactionalEventListener` for side effects that must not execute if the transaction rolls back.

**Common Follow-Up Questions**

1. **How do you order listeners?** -- Annotate with `@Order(priority)` or implement `Ordered`.
2. **What is the difference between `ApplicationEvent` and domain events in DDD?** -- `ApplicationEvent` is a Spring infrastructure concern; domain events are part of the business model and are often collected on the aggregate root, then published after persistence.
3. **How does reactive `Flux`/`Sinks` relate to Observer?** -- Reactive streams are a push-based Observer with backpressure; `Sinks.Many` acts as the subject, `Flux` subscribers act as observers.

---

### What is the Proxy pattern? How does Spring AOP leverage proxies?

**Core Explanation**

The Proxy pattern provides a surrogate or placeholder for another object to control access to it. The proxy implements the same interface as the real subject, intercepting calls to add behaviour (logging, security, lazy loading, transaction management) without modifying the real object.

**Practical Example -- Static Proxy**

```java
public interface PaymentService {
    void pay(BigDecimal amount);
}

public class RealPaymentService implements PaymentService {
    public void pay(BigDecimal amount) {
        System.out.println("Processing payment: " + amount);
    }
}

public class PaymentServiceProxy implements PaymentService {

    private final PaymentService delegate;

    public PaymentServiceProxy(PaymentService delegate) {
        this.delegate = delegate;
    }

    @Override
    public void pay(BigDecimal amount) {
        System.out.println("LOG: Payment request for " + amount);
        long start = System.nanoTime();
        delegate.pay(amount);
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        System.out.println("LOG: Payment completed in " + elapsed + " ms");
    }
}
```

**Spring AOP Proxies**

Spring creates proxies at runtime to implement cross-cutting concerns (`@Transactional`, `@Cacheable`, `@Async`, custom aspects):

| Proxy Type | Mechanism | When Used |
|---|---|---|
| **JDK Dynamic Proxy** | `java.lang.reflect.Proxy` | Bean implements at least one interface (default) |
| **CGLIB Proxy** | Subclass generation via bytecode | Bean has no interface, or `proxyTargetClass=true` |

```java
@Aspect
@Component
public class TimingAspect {

    @Around("@annotation(Timed)")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            System.out.println(pjp.getSignature() + " took " + ms + " ms");
        }
    }
}
```

**Edge Cases / Pitfalls**

- **Self-invocation trap**: calling a `@Transactional` method from another method *in the same class* bypasses the proxy. The call goes directly to `this`, not through the proxy. Fix: inject the bean into itself, use `AopContext.currentProxy()`, or extract the method to another bean.
- CGLIB cannot proxy `final` classes or `final` methods.
- Proxied beans have a different class at runtime, which can break `instanceof` checks or direct field access.

**Best Practices**

Program to interfaces so Spring can use the lighter JDK Dynamic Proxy. Be aware of the self-invocation limitation and design bean boundaries accordingly.

**Common Follow-Up Questions**

1. **What is the difference between JDK and CGLIB proxies?** -- JDK proxies require an interface and use `InvocationHandler`; CGLIB generates a subclass via bytecode and does not require an interface.
2. **How does `@Transactional` work under the hood?** -- Spring wraps the bean in a proxy that opens a transaction before the method, commits on success, and rolls back on `RuntimeException`.
3. **What is a dynamic proxy in plain Java?** -- `java.lang.reflect.Proxy.newProxyInstance(classLoader, interfaces, invocationHandler)` creates a proxy at runtime that delegates every call to the `InvocationHandler`.

---

### What is the Template Method pattern? Where is it used in the Spring framework?

**Core Explanation**

The Template Method pattern defines the skeleton of an algorithm in a superclass method, deferring certain steps to subclasses. The superclass controls the overall flow (`final` template method), while subclasses override specific *hook* methods.

**Practical Example**

```java
public abstract class DataImporter {

    // Template method -- defines the algorithm skeleton
    public final void importData() {
        String raw  = readSource();
        List<?> parsed = parse(raw);
        validate(parsed);
        save(parsed);
    }

    protected abstract String readSource();
    protected abstract List<?> parse(String raw);

    // Hook with default behaviour -- subclasses can override
    protected void validate(List<?> data) {
        // no-op by default
    }

    protected abstract void save(List<?> data);
}

public class CsvImporter extends DataImporter {
    protected String readSource()        { /* read CSV file */ return ""; }
    protected List<?> parse(String raw)  { /* parse CSV */ return List.of(); }
    protected void save(List<?> data)    { /* insert into DB */ }
}
```

**Spring Framework Usage**

| Spring Class | Template Method |
|---|---|
| `JdbcTemplate` | Manages connection, statement, result set lifecycle; user provides `RowMapper` or `PreparedStatementSetter` callbacks |
| `RestTemplate` | Handles HTTP connection setup, error handling; user provides `ResponseExtractor` |
| `JmsTemplate` | Manages JMS session and producer lifecycle |
| `TransactionTemplate` | Wraps `TransactionCallback` with begin/commit/rollback logic |
| `AbstractController` (Spring MVC legacy) | `handleRequestInternal()` hook |

```java
// JdbcTemplate example -- user provides the RowMapper (hook/callback)
List<User> users = jdbcTemplate.query(
    "SELECT id, name, email FROM users WHERE active = ?",
    (rs, rowNum) -> new User(
        rs.getLong("id"),
        rs.getString("name"),
        rs.getString("email")
    ),
    true
);
```

> Note: Spring often uses *callbacks* (composition) instead of *subclassing* for the hook steps, which is a more flexible variant of the Template Method idea.

**Edge Cases / Pitfalls**

- Deep inheritance hierarchies make the flow hard to trace. Prefer composition-based template methods (callbacks/lambdas) over subclassing when possible.
- Forgetting to make the template method `final` lets subclasses accidentally override the algorithm skeleton.

**Best Practices**

Limit the template to one level of inheritance. Favour the callback/lambda approach (as Spring's `JdbcTemplate` does) over forcing users to subclass.

**Common Follow-Up Questions**

1. **How does Template Method differ from Strategy?** -- Template Method uses inheritance (subclass overrides steps); Strategy uses composition (inject a strategy object). Strategy is generally more flexible.
2. **Why does Spring prefer callbacks over abstract classes?** -- Callbacks avoid coupling to an inheritance hierarchy, work with lambdas, and allow more granular reuse.

---

### What is the Adapter pattern? Give a real-world Java example.

**Core Explanation**

The Adapter pattern converts the interface of one class into another interface that clients expect. It allows classes with incompatible interfaces to work together. Think of it as a "translator" between two systems.

**Practical Example**

```java
// Legacy system returns XML strings
public class LegacyUserService {
    public String getUserXml(int id) {
        return "<user><id>" + id + "</id><name>Alice</name></user>";
    }
}

// Modern system expects a POJO
public record UserDto(int id, String name) {}

// Target interface
public interface UserProvider {
    UserDto getUser(int id);
}

// Adapter
public class LegacyUserAdapter implements UserProvider {

    private final LegacyUserService legacy;

    public LegacyUserAdapter(LegacyUserService legacy) {
        this.legacy = legacy;
    }

    @Override
    public UserDto getUser(int id) {
        String xml = legacy.getUserXml(id);
        // Parse XML and map to DTO
        int parsedId = extractInt(xml, "id");
        String name  = extractString(xml, "name");
        return new UserDto(parsedId, name);
    }

    private int extractInt(String xml, String tag) { /* parsing logic */ return 0; }
    private String extractString(String xml, String tag) { /* parsing logic */ return ""; }
}
```

**Real-World JDK Examples**

| Adapter | Adapts From | Adapts To |
|---|---|---|
| `Arrays.asList(T[])` | Array | `List<T>` |
| `Collections.enumeration(Collection)` | `Collection` | `Enumeration` |
| `InputStreamReader` | `InputStream` (bytes) | `Reader` (chars) |
| `OutputStreamWriter` | `OutputStream` (bytes) | `Writer` (chars) |

```java
// InputStreamReader adapts byte stream to character stream
try (Reader reader = new InputStreamReader(
        new FileInputStream("data.csv"), StandardCharsets.UTF_8)) {
    // now we can read characters instead of raw bytes
}
```

**Edge Cases / Pitfalls**

- `Arrays.asList()` returns a fixed-size list backed by the array; calling `add()` or `remove()` throws `UnsupportedOperationException`. This is a common interview trick.
- An adapter that does too much transformation becomes a facade or translator; keep adapters thin.

**Best Practices**

Use adapters at system boundaries (legacy integration, third-party libraries) to isolate external API changes from your domain model.

**Common Follow-Up Questions**

1. **What is the difference between Adapter and Facade?** -- Adapter makes one existing interface conform to another; Facade provides a simplified interface over a complex subsystem.
2. **Object adapter vs. class adapter?** -- Object adapter uses composition (wraps the adaptee); class adapter uses multiple inheritance (not common in Java since Java lacks multiple class inheritance).
3. **How does `HandlerAdapter` work in Spring MVC?** -- It adapts different handler types (annotated controllers, `HttpRequestHandler`, `Controller` interface) to a uniform `handle()` method that the `DispatcherServlet` can call.

---

### How do design patterns improve testability?

**Core Explanation**

Design patterns improve testability by enforcing **loose coupling** and **programming to interfaces**, which makes it easy to substitute real dependencies with test doubles (mocks, stubs, fakes).

**Key Patterns and Their Testing Benefits**

| Pattern | Testability Benefit |
|---|---|
| **Strategy** | Inject a mock strategy to verify behaviour in isolation |
| **Factory** | Replace the factory to return test doubles |
| **Observer** | Test publisher and subscriber independently |
| **Proxy / Decorator** | Wrap a real object with a spy to verify interactions |
| **Template Method** | Test the skeleton independently; mock the hook methods |
| **Dependency Injection** (not GoF, but a pattern) | The foundation of testability -- inject mocks via constructor |

**Practical Example**

```java
// Strategy makes the payment gateway swappable
public class OrderService {

    private final PaymentStrategy paymentStrategy;

    public OrderService(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;     // injected -- testable
    }

    public OrderResult checkout(Order order) {
        boolean paid = paymentStrategy.charge(order.getTotal());
        return paid ? OrderResult.SUCCESS : OrderResult.FAILED;
    }
}

// Test
@Test
void checkout_success() {
    PaymentStrategy mockPayment = mock(PaymentStrategy.class);
    when(mockPayment.charge(any())).thenReturn(true);

    OrderService service = new OrderService(mockPayment);
    assertEquals(OrderResult.SUCCESS, service.checkout(testOrder));
}
```

**Contrast with Untestable Code**

```java
// Hard to test -- creates its own dependency
public class OrderService {
    public OrderResult checkout(Order order) {
        StripeGateway gateway = new StripeGateway();   // tight coupling
        return gateway.charge(order.getTotal())
                ? OrderResult.SUCCESS : OrderResult.FAILED;
    }
}
// Cannot mock StripeGateway without PowerMock or refactoring.
```

**Best Practices**

Design for injection: accept dependencies through constructors (Strategy, Factory). If a class creates its collaborators internally, it is hard to test in isolation.

**Common Follow-Up Questions**

1. **Does using patterns guarantee testability?** -- No. A badly implemented Singleton or a God-class using patterns internally can still be untestable. Patterns must be combined with SOLID principles.
2. **How does the Factory pattern help testing?** -- You can inject a test factory that returns stubs/mocks, controlling the objects your code receives without changing its logic.
3. **What role does DI play in testable design?** -- DI externalises object creation, letting you swap in test doubles at construction time. Spring and constructor injection make this trivial.

---

## Advanced

---

### How do you make a Singleton thread-safe? Compare eager initialization, double-checked locking, enum, and Bill Pugh approaches.

**Core Explanation**

The naive lazy Singleton is not thread-safe because two threads can simultaneously see `instance == null` and each create a new object. Four common thread-safe approaches exist, each with different trade-offs.

**1. Eager Initialization**

```java
public class EagerSingleton {

    private static final EagerSingleton INSTANCE = new EagerSingleton();

    private EagerSingleton() {}

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}
```

- **Thread safety**: Guaranteed by the JVM class-loading mechanism (static fields are initialized once).
- **Downside**: Instance is created even if never used; no lazy loading. Wastes resources if construction is expensive.

**2. Double-Checked Locking (DCL)**

```java
public class DclSingleton {

    private static volatile DclSingleton instance;   // volatile is critical

    private DclSingleton() {}

    public static DclSingleton getInstance() {
        if (instance == null) {                     // 1st check (no lock)
            synchronized (DclSingleton.class) {
                if (instance == null) {             // 2nd check (under lock)
                    instance = new DclSingleton();
                }
            }
        }
        return instance;
    }
}
```

- **Thread safety**: `volatile` prevents instruction reordering; `synchronized` prevents race conditions.
- **Downside**: Verbose, error-prone (forgetting `volatile` breaks it on some JVMs). Not recommended for new code.

**3. Bill Pugh (Static Inner Class Holder)**

```java
public class BillPughSingleton {

    private BillPughSingleton() {}

    private static class Holder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

- **Thread safety**: The JVM loads the inner class only when `getInstance()` is first called; class loading is inherently thread-safe.
- **Advantage**: Lazy + thread-safe + no synchronization overhead.
- **Downside**: Still vulnerable to reflection and serialization attacks.

**4. Enum Singleton**

```java
public enum EnumSingleton {

    INSTANCE;

    private final Map<String, String> config = new ConcurrentHashMap<>();

    public void put(String key, String value) { config.put(key, value); }
    public String get(String key) { return config.get(key); }
}
```

- **Thread safety**: Guaranteed by the JVM enum instantiation mechanism.
- **Advantage**: Immune to reflection, serialization, and cloning attacks by design.
- **Downside**: Cannot extend a class (enums implicitly extend `java.lang.Enum`); cannot lazy-load (created at class-load time).

**Comparison Summary**

| Approach | Lazy | Thread-Safe | Reflection-Proof | Serialization-Proof | Complexity |
|---|---|---|---|---|---|
| Eager | No | Yes | No | No | Low |
| DCL | Yes | Yes (if `volatile`) | No | No | High |
| Bill Pugh | Yes | Yes | No | No | Low |
| Enum | No | Yes | Yes | Yes | Lowest |

**Best Practices**

Use **Enum Singleton** when the singleton does not need to extend a class. Use **Bill Pugh** when lazy loading is required and you can tolerate the reflection/serialization risk. Avoid DCL in new code.

**Common Follow-Up Questions**

1. **Why is `volatile` required in DCL?** -- Without it, the JVM may reorder the writes during object construction, allowing another thread to see a partially constructed object.
2. **Can you lazy-load with Enum?** -- Not the enum constant itself, but you can lazy-load heavy resources inside it using a holder or `Supplier`.

---

### How can Singleton be broken via reflection, serialization, or cloning? How do you prevent each?

**Core Explanation**

Even a carefully written Singleton can be broken in three ways:

**1. Reflection Attack**

```java
// Breaking
Constructor<BillPughSingleton> constructor =
        BillPughSingleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
BillPughSingleton second = constructor.newInstance();   // new instance!

System.out.println(BillPughSingleton.getInstance() == second);  // false
```

**Prevention**: Throw from the private constructor if an instance already exists.

```java
private BillPughSingleton() {
    if (Holder.INSTANCE != null) {
        throw new IllegalStateException("Singleton already initialized");
    }
}
```

**2. Serialization Attack**

```java
// Breaking
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("singleton.ser"));
out.writeObject(BillPughSingleton.getInstance());
out.close();

ObjectInputStream in = new ObjectInputStream(new FileInputStream("singleton.ser"));
BillPughSingleton deserialized = (BillPughSingleton) in.readObject();   // new instance!
in.close();

System.out.println(BillPughSingleton.getInstance() == deserialized);    // false
```

**Prevention**: Implement `readResolve()`.

```java
public class BillPughSingleton implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    // ... existing code ...

    @Serial
    private Object readResolve() {
        return Holder.INSTANCE;   // return the real singleton
    }
}
```

**3. Clone Attack**

```java
// If Singleton implements Cloneable (bad idea, but possible via inheritance)
BillPughSingleton cloned = (BillPughSingleton) singleton.clone();   // new instance
```

**Prevention**: Override `clone()` to throw or return the same instance.

```java
@Override
protected Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException("Singleton cannot be cloned");
}
```

**Edge Cases / Pitfalls**

- The reflection guard in the constructor introduces a chicken-and-egg timing issue if `Holder.INSTANCE` has not been initialized yet. Carefully order the check.
- `Enum` singletons handle all three attacks automatically -- the JVM prevents enum reflection instantiation, serialization returns the same constant, and `clone()` is final.

**Best Practices**

Use Enum Singleton to avoid having to defend against these attacks manually. If you must use a class-based singleton, implement all three defenses.

**Common Follow-Up Questions**

1. **Can `Unsafe.allocateInstance()` bypass the constructor guard?** -- Yes. `sun.misc.Unsafe` can create instances without calling any constructor, bypassing all guards. This is an extreme edge case only relevant in security-sensitive contexts.
2. **Does Spring protect its singletons from reflection?** -- No. Spring singleton scope simply means one bean per context stored in a map. If someone uses reflection to create another instance of the class, Spring cannot prevent it.

---

### Why is Enum Singleton considered the safest approach?

**Core Explanation**

The JVM provides three guarantees for enum constants that make Enum Singleton inherently safe:

1. **Thread safety**: Enum constants are instantiated during class loading, which is thread-safe by the JLS (Java Language Specification, section 12.4.2).
2. **Serialization safety**: The `ObjectInputStream` deserializes enum constants by calling `Enum.valueOf()`, which returns the existing constant rather than creating a new instance. The `readResolve()` mechanism is not needed.
3. **Reflection safety**: The `Constructor.newInstance()` method explicitly checks if the class is an enum and throws `IllegalArgumentException("Cannot reflectively create enum objects")`.

```java
public enum ConfigManager {

    INSTANCE;

    private final Map<String, String> settings = new ConcurrentHashMap<>();

    public void set(String key, String value) { settings.put(key, value); }
    public String get(String key) { return settings.get(key); }
}

// Usage
ConfigManager.INSTANCE.set("timeout", "5000");
String timeout = ConfigManager.INSTANCE.get("timeout");
```

**Limitations of Enum Singleton**

- Cannot extend a class (already extends `java.lang.Enum`).
- Eagerly initialized (no lazy loading of the instance itself).
- Looks unusual if the singleton has many methods -- reads more like a utility enum than a service.
- Cannot be used when the singleton must be created by a DI framework (Spring cannot instantiate enums with `@Component`).

**Best Practices**

Use Enum Singleton for infrastructure-level singletons (registries, caches, utility holders). For application-layer services, prefer Spring-managed singleton scope with constructor injection.

**Common Follow-Up Questions**

1. **Who recommended Enum Singleton?** -- Joshua Bloch, in *Effective Java* (Item 3): "A single-element enum type is often the best way to implement a singleton."
2. **Can an Enum Singleton implement interfaces?** -- Yes. Enums can implement any number of interfaces, which enables programming to an interface and improves testability.
3. **Can you use Enum Singleton with dependency injection?** -- Not directly with Spring `@Component`. You can register it as a bean via `@Bean` in a configuration class, but this is unusual.

---

### What is the difference between the Decorator and Proxy patterns?

**Core Explanation**

Both Decorator and Proxy wrap an object and implement the same interface, but their **intent** differs:

| Aspect | Decorator | Proxy |
|---|---|---|
| **Intent** | Add new behaviour / responsibilities dynamically | Control access to the wrapped object |
| **Relationship** | The client typically knows it is decorating | The proxy is often transparent to the client |
| **Wrapping** | Multiple decorators can be stacked | Usually a single proxy |
| **Creation** | Client creates and composes decorators explicitly | Proxy is often created by a framework or factory |
| **Examples** | `BufferedInputStream(new FileInputStream(...))` | Spring AOP proxy, `java.lang.reflect.Proxy` |

**Practical Example -- Decorator**

```java
// Stacking decorators -- each adds behaviour
InputStream in = new BufferedInputStream(           // buffering decorator
                    new GZIPInputStream(            // decompression decorator
                        new FileInputStream("data.gz")));

// Custom decorator
public class LoggingList<E> implements List<E> {

    private final List<E> delegate;

    public LoggingList(List<E> delegate) { this.delegate = delegate; }

    @Override
    public boolean add(E e) {
        System.out.println("Adding: " + e);
        return delegate.add(e);             // adds logging, delegates actual work
    }
    // ... delegate remaining methods ...
}
```

**Practical Example -- Proxy**

```java
// Access control proxy
public class SecuredPaymentService implements PaymentService {

    private final PaymentService real;
    private final SecurityContext security;

    public SecuredPaymentService(PaymentService real, SecurityContext security) {
        this.real = real;
        this.security = security;
    }

    @Override
    public void pay(BigDecimal amount) {
        if (!security.hasRole("FINANCE")) {
            throw new AccessDeniedException("Insufficient privileges");
        }
        real.pay(amount);                   // controls access, then delegates
    }
}
```

**Key Distinction**

- **Decorator**: "I do what you do, *plus* something extra." (enhancement)
- **Proxy**: "I decide *whether and when* you do it." (control)

**Edge Cases / Pitfalls**

- In practice, the structural code looks nearly identical. The distinction is purely about intent. Interviewers want you to articulate this.
- Spring AOP proxies behave more like decorators when adding `@Transactional` behaviour, but they are called proxies because the client does not know about them.

**Best Practices**

Name your wrapper classes to express intent: `CachingProxy`, `LoggingDecorator`. This makes the design self-documenting.

**Common Follow-Up Questions**

1. **Can you give a JDK example of each?** -- Decorator: `java.io` stream wrappers. Proxy: `java.lang.reflect.Proxy`, `Collections.unmodifiableList()`.
2. **Is `Collections.unmodifiableList()` a proxy or decorator?** -- It is a proxy: it controls access by throwing `UnsupportedOperationException` on mutation methods.
3. **How does the Proxy pattern relate to the Facade?** -- Proxy wraps a single object with the same interface; Facade wraps an entire subsystem behind a simplified interface.

---

### When should you deliberately avoid using a design pattern?

**Core Explanation**

Patterns should be *discovered* through refactoring, not *imposed* upfront. Deliberately avoid a pattern when:

1. **The problem is simple** -- a two-branch `if-else` does not need a Strategy hierarchy.
2. **YAGNI (You Aren't Gonna Need It)** -- adding a pattern for hypothetical future requirements increases complexity now for uncertain future benefit.
3. **The pattern obscures intent** -- if a new team member needs to trace through five classes to understand "send an email," the pattern has failed.
4. **Performance is critical** -- patterns add indirection (virtual method calls, object creation). In tight loops or latency-sensitive paths, a direct approach may be warranted.
5. **The team does not know the pattern** -- a pattern only works as a shared vocabulary if the team shares the vocabulary.

**Code Example -- Avoidable Pattern**

```java
// OVER-ENGINEERED: Strategy for two static options that will never change
interface TaxStrategy { BigDecimal calculate(BigDecimal amount); }
class DomesticTaxStrategy implements TaxStrategy { ... }
class InternationalTaxStrategy implements TaxStrategy { ... }
class TaxStrategyFactory { ... }
class TaxService { TaxService(TaxStrategyFactory factory) { ... } }

// SUFFICIENT: Simple method
public BigDecimal calculateTax(BigDecimal amount, boolean domestic) {
    return domestic
        ? amount.multiply(new BigDecimal("0.10"))
        : amount.multiply(new BigDecimal("0.20"));
}
```

**Decision Framework**

| Question | If Yes | If No |
|---|---|---|
| Are there 3+ variants likely to grow? | Use Strategy/Factory | Keep it simple |
| Is the object constructed with many optional params? | Use Builder | Use constructor |
| Do multiple unrelated consumers need notifications? | Use Observer | Direct method call |
| Is the construction logic complex and reusable? | Use Factory | `new` is fine |

**Best Practices**

Follow the Rule of Three: introduce a pattern when you see the same structural problem for the third time, not the first.

**Common Follow-Up Questions**

1. **How do you balance "clean code" with simplicity?** -- Clean code *is* simple. Patterns that add complexity without solving a real problem are not clean.
2. **What is "Resume-Driven Development"?** -- Choosing technologies or patterns because they look good on a resume rather than because they solve a problem. A common source of over-engineering.
3. **How do you refactor toward a pattern?** -- Start with the simplest solution. When you notice duplication or violation of SOLID principles, refactor toward the appropriate pattern incrementally.

---

### How would you refactor a large `switch` statement using the Strategy + Factory pattern combination?

**Core Explanation**

A large `switch` (or `if-else` chain) that selects behaviour based on a type discriminator violates the Open/Closed Principle: every new type requires modifying the switch. The Strategy + Factory combination replaces the switch with a registry of strategy objects, making the system open for extension without modification.

**Before -- Monolithic Switch**

```java
public class NotificationService {

    public void send(String channel, String recipient, String message) {
        switch (channel) {
            case "EMAIL":
                // 30 lines of email sending logic
                SmtpClient smtp = new SmtpClient();
                smtp.connect();
                smtp.send(recipient, message);
                smtp.disconnect();
                break;
            case "SMS":
                // 20 lines of SMS logic
                SmsGateway gw = new SmsGateway();
                gw.sendText(recipient, message);
                break;
            case "PUSH":
                // 25 lines of push notification logic
                PushService push = PushService.getInstance();
                push.notify(recipient, message);
                break;
            case "SLACK":
                // 15 lines of Slack webhook logic
                SlackClient slack = new SlackClient();
                slack.postMessage(recipient, message);
                break;
            default:
                throw new IllegalArgumentException("Unknown channel: " + channel);
        }
    }
}
```

**After -- Strategy + Factory**

**Step 1: Define the Strategy interface**

```java
public interface NotificationStrategy {
    void send(String recipient, String message);
    String getChannel();    // self-identifying for auto-registration
}
```

**Step 2: Implement concrete strategies**

```java
@Component
public class EmailNotificationStrategy implements NotificationStrategy {

    public String getChannel() { return "EMAIL"; }

    public void send(String recipient, String message) {
        SmtpClient smtp = new SmtpClient();
        smtp.connect();
        smtp.send(recipient, message);
        smtp.disconnect();
    }
}

@Component
public class SmsNotificationStrategy implements NotificationStrategy {

    public String getChannel() { return "SMS"; }

    public void send(String recipient, String message) {
        new SmsGateway().sendText(recipient, message);
    }
}

// ... PushNotificationStrategy, SlackNotificationStrategy ...
```

**Step 3: Create the Factory (auto-registering with Spring)**

```java
@Component
public class NotificationStrategyFactory {

    private final Map<String, NotificationStrategy> strategies;

    // Spring injects all NotificationStrategy beans automatically
    public NotificationStrategyFactory(List<NotificationStrategy> strategyList) {
        this.strategies = strategyList.stream()
                .collect(Collectors.toUnmodifiableMap(
                        NotificationStrategy::getChannel,
                        Function.identity()
                ));
    }

    public NotificationStrategy getStrategy(String channel) {
        return Optional.ofNullable(strategies.get(channel))
                .orElseThrow(() -> new IllegalArgumentException(
                        "No strategy for channel: " + channel));
    }
}
```

**Step 4: Simplified Service**

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final NotificationStrategyFactory factory;

    public void send(String channel, String recipient, String message) {
        factory.getStrategy(channel).send(recipient, message);
    }
}
```

**Adding a new channel now requires only**:
1. Create a new `@Component` implementing `NotificationStrategy`.
2. That is it. No changes to `NotificationService` or `NotificationStrategyFactory`.

**Without Spring (plain Java)**

```java
public class NotificationStrategyFactory {

    private static final Map<String, NotificationStrategy> STRATEGIES = Map.of(
        "EMAIL", new EmailNotificationStrategy(),
        "SMS",   new SmsNotificationStrategy(),
        "PUSH",  new PushNotificationStrategy(),
        "SLACK", new SlackNotificationStrategy()
    );

    public static NotificationStrategy getStrategy(String channel) {
        NotificationStrategy strategy = STRATEGIES.get(channel);
        if (strategy == null) {
            throw new IllegalArgumentException("Unknown channel: " + channel);
        }
        return strategy;
    }
}
```

**Edge Cases / Pitfalls**

- If strategies have state, a shared `Map` can cause thread-safety issues. Use `Supplier<NotificationStrategy>` in the map and create fresh instances per call, or ensure strategies are stateless.
- If there are only 2-3 branches and no realistic chance of growth, the refactoring adds complexity without benefit. Apply judgment.
- ServiceLoader (SPI) is another option for registration without a DI framework: `META-INF/services/com.example.NotificationStrategy`.

**Best Practices**

Let Spring auto-discover strategies via `List<T>` injection. Have each strategy self-identify its discriminator (`getChannel()`) to avoid a separate registration step.

**Common Follow-Up Questions**

1. **How would you handle a default/fallback strategy?** -- Add a `DefaultNotificationStrategy` with a lower `@Order` priority, or use `Map.getOrDefault()`.
2. **How does this compare to using an enum with behaviour?** -- Enums with abstract methods work well for a closed set of types. Strategy + Factory is better when the set is open (plugins, new modules).
3. **How would you test this?** -- Test each strategy in isolation with unit tests. Test the factory separately to verify correct resolution. Test the service with a mock factory. Integration tests verify the wiring.
# 3. Core Java — Modern Java (8–21)

---

## Basic

---

### What are the key features introduced in Java 8?

**Core Explanation**

Java 8 was the most transformative release in the language's history. The key features are:

1. **Lambda Expressions** — concise anonymous function syntax enabling functional-style programming.
2. **Functional Interfaces** — interfaces with a single abstract method (SAM), annotated with `@FunctionalInterface`.
3. **Stream API** — declarative pipeline for processing collections (filter, map, reduce).
4. **Optional** — a container type to represent the presence or absence of a value.
5. **Default Methods** — methods with implementations in interfaces, enabling interface evolution.
6. **Method References** — shorthand for lambdas that call a single existing method.
7. **New Date/Time API (`java.time`)** — immutable, thread-safe replacement for `java.util.Date` and `Calendar`.
8. **`CompletableFuture`** — composable asynchronous programming primitive.
9. **Nashorn JavaScript Engine** (deprecated in 11, removed in 15).
10. **`forEach()`** on `Iterable`, `StringJoiner`, `Base64` utilities.

**Practical Example**

```java
// Before Java 8
List<String> filtered = new ArrayList<>();
for (String s : names) {
    if (s.startsWith("A")) {
        filtered.add(s.toUpperCase());
    }
}
Collections.sort(filtered);

// Java 8
List<String> filtered = names.stream()
    .filter(s -> s.startsWith("A"))
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());
```

**Edge Cases / Pitfalls**
- Lambda expressions capture effectively-final variables only; modifying an outer variable inside a lambda is a compile error.
- Streams are single-use; calling a terminal operation twice on the same stream throws `IllegalStateException`.

**Best Practices**
- Adopt `java.time` over legacy `Date`/`Calendar` unconditionally. Use streams for declarative transformations but keep side-effect logic in plain loops.

**Common Follow-Up Questions**
1. *Why was Java 8 backward-compatible despite adding default methods?* — Default methods avoid breaking existing implementations; if a class implements two interfaces with the same default method, the compiler forces an explicit override.
2. *What is the difference between `Iterable.forEach()` and `Stream.forEach()`?* — `Iterable.forEach()` iterates the collection directly; `Stream.forEach()` is a terminal stream operation with no guaranteed order in parallel streams.
3. *Has anything from Java 8 been deprecated since?* — Nashorn was deprecated in Java 11 and removed in Java 15. The `Unsafe` methods used internally are being replaced incrementally.

---

### What is a functional interface? Name three from `java.util.function`.

**Core Explanation**

A functional interface is an interface with exactly **one abstract method** (SAM — Single Abstract Method). It may contain any number of default or static methods. The `@FunctionalInterface` annotation is optional but recommended because it causes a compile error if the contract is violated.

Functional interfaces are the **type targets** for lambda expressions and method references.

Three key functional interfaces from `java.util.function`:

| Interface | SAM Signature | Purpose |
|---|---|---|
| `Predicate<T>` | `boolean test(T t)` | Evaluate a condition |
| `Function<T, R>` | `R apply(T t)` | Transform input to output |
| `Consumer<T>` | `void accept(T t)` | Perform side-effect on input |

Other notable ones: `Supplier<T>`, `UnaryOperator<T>`, `BiFunction<T, U, R>`, `BiConsumer<T, U>`.

**Practical Example**

```java
@FunctionalInterface
interface Validator<T> {
    boolean validate(T t);
}

// Usage with lambda
Predicate<String> nonEmpty = s -> s != null && !s.isBlank();
Function<String, Integer> toLength = String::length;
Consumer<String> printer = System.out::println;

// Composing predicates
Predicate<String> validInput = nonEmpty.and(s -> s.length() <= 100);
```

**Edge Cases / Pitfalls**
- Interfaces that extend another functional interface and add no new abstract method are still functional interfaces (e.g., `UnaryOperator<T> extends Function<T, T>`).
- An interface with one abstract method plus methods from `Object` (like `equals()`) is still a functional interface — `Object` methods do not count.
- Forgetting `@FunctionalInterface` means accidental addition of a second abstract method will not cause a compile error.

**Best Practices**
- Always annotate custom functional interfaces with `@FunctionalInterface`. Prefer the built-in `java.util.function` types over creating custom ones unless the name adds significant clarity.

**Common Follow-Up Questions**
1. *Is `Comparable` a functional interface?* — Yes, it has a single abstract method `compareTo()`, so it can be a lambda target, though it is not annotated with `@FunctionalInterface`.
2. *Why does `Runnable` qualify?* — It has one abstract method (`run()`). It predates Java 8 but was retrofitted as a functional interface.
3. *What are primitive specializations?* — Types like `IntPredicate`, `LongFunction<R>`, `DoubleConsumer` avoid autoboxing overhead.

---

### What is a lambda expression? How does it relate to functional interfaces?

**Core Explanation**

A lambda expression is an anonymous function — a concise way to represent a single-method implementation. It provides the **body** for the single abstract method of a functional interface.

Syntax: `(parameters) -> expression` or `(parameters) -> { statements; }`

The **type** of a lambda is always a functional interface. The compiler infers which functional interface based on the target type context (assignment, method argument, return).

Under the hood, lambdas are **not** anonymous inner classes. The JVM uses `invokedynamic` and `LambdaMetafactory` to generate implementation classes at runtime, which is more efficient than creating `.class` files for inner classes.

**Practical Example**

```java
// Full form
Comparator<String> byLength = (String a, String b) -> {
    return Integer.compare(a.length(), b.length());
};

// Concise form (type inference, expression body)
Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());

// Single parameter — parentheses optional
Consumer<String> print = s -> System.out.println(s);

// No parameters
Runnable task = () -> System.out.println("Running");

// Capturing effectively-final variables
String prefix = "Hello";
Function<String, String> greeter = name -> prefix + " " + name;
// prefix = "Hi"; // Compile error — prefix must be effectively final
```

**Edge Cases / Pitfalls**
- Lambdas capture **values** of local variables, not references. The variable must be effectively final.
- `this` inside a lambda refers to the **enclosing class**, not the lambda itself (unlike anonymous inner classes where `this` refers to the inner class instance).
- Lambdas cannot have their own state (fields). If you need state, use an anonymous inner class or a named class.
- Serializable lambdas (casting to `Serializable`) are fragile and implementation-dependent.

**Best Practices**
- Keep lambdas short (1-3 lines). If the logic is complex, extract it into a named method and use a method reference.

**Common Follow-Up Questions**
1. *How are lambdas different from anonymous inner classes at the bytecode level?* — Lambdas use `invokedynamic` which defers class generation to runtime via `LambdaMetafactory`; anonymous inner classes create a separate `.class` file at compile time.
2. *Can lambdas throw checked exceptions?* — Only if the target functional interface's SAM declares the exception. Standard interfaces like `Function` do not, so you must wrap or use a custom interface.
3. *What is a closure in Java?* — A lambda that captures variables from its enclosing scope. Java closures capture only effectively-final values, unlike true closures in other languages that can capture mutable references.

---

### What is a method reference? What are the four types?

**Core Explanation**

A method reference is a shorthand syntax for a lambda that calls a single existing method. It uses the `::` operator. It improves readability when the lambda does nothing more than delegate to an existing method.

The four types:

| Type | Syntax | Lambda Equivalent |
|---|---|---|
| Static method | `ClassName::staticMethod` | `x -> ClassName.staticMethod(x)` |
| Instance method on a specific object | `instance::method` | `x -> instance.method(x)` |
| Instance method on an arbitrary object of a type | `ClassName::instanceMethod` | `(obj, x) -> obj.instanceMethod(x)` |
| Constructor | `ClassName::new` | `x -> new ClassName(x)` |

**Practical Example**

```java
// 1. Static method reference
Function<String, Integer> parse = Integer::parseInt;
// equivalent to: s -> Integer.parseInt(s)

// 2. Instance method on a specific object
String prefix = "Hello";
Predicate<String> startCheck = prefix::startsWith;
// equivalent to: s -> prefix.startsWith(s)  [but note: this is "Hello".startsWith(s)]

// 3. Instance method on arbitrary object of a type
Function<String, String> upper = String::toUpperCase;
// equivalent to: s -> s.toUpperCase()

// Used commonly in streams:
List<String> uppercased = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// 4. Constructor reference
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<String, File> fileCreator = File::new;
```

**Edge Cases / Pitfalls**
- Type 3 (arbitrary instance) can be confusing: `String::compareTo` becomes `(a, b) -> a.compareTo(b)` — the first parameter becomes the receiver.
- Overloaded methods may cause ambiguity; the compiler resolves based on the target functional interface, but sometimes explicit casting is needed.
- Constructor references with generics: `ArrayList::new` works as `Supplier<List<String>>` because of type inference, but complex generic scenarios may require explicit type witnesses.

**Best Practices**
- Prefer method references over lambdas when the lambda body is a direct method call with no transformation. Avoid them when they reduce clarity (e.g., deeply nested or overloaded methods).

**Common Follow-Up Questions**
1. *Can you reference a method that takes no arguments?* — Yes, e.g., `String::isEmpty` used as `Predicate<String>`.
2. *Can you use method references with multiple parameters?* — Yes, e.g., `String::compareTo` maps to `BiFunction<String, String, Integer>` or `Comparator<String>`.
3. *How does the compiler decide between type 2 and type 3?* — If the left side of `::` is a type name, it is type 3 (or type 1 if static). If it is a variable/expression, it is type 2.

---

### What are default methods in interfaces? Why were they introduced?

**Core Explanation**

Default methods are methods in interfaces that have a body (implementation), declared with the `default` keyword. They were introduced in Java 8 primarily to **evolve existing interfaces** without breaking all implementing classes.

The immediate motivation was adding `stream()`, `forEach()`, and other methods to `Collection` and `Iterable` — without default methods, every existing collection class would have broken.

They also enable a limited form of **multiple inheritance of behavior** (but not state).

**Practical Example**

```java
public interface Loggable {
    default void log(String message) {
        System.out.println("[" + getClass().getSimpleName() + "] " + message);
    }
}

public interface Auditable {
    default void log(String message) {
        System.out.println("[AUDIT] " + message);
    }
}

// Diamond problem — compiler forces explicit override
public class Service implements Loggable, Auditable {
    @Override
    public void log(String message) {
        Loggable.super.log(message); // explicitly choose
    }
}

// Real-world example: interface evolution
public interface Collection<E> {
    // existing abstract methods ...

    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }

    default void forEach(Consumer<? super E> action) {
        for (E e : this) {
            action.accept(e);
        }
    }
}
```

**Edge Cases / Pitfalls**
- If a class inherits the same default method from two interfaces, it **must** override it or the code will not compile.
- A class method always wins over a default method (class wins rule).
- A more specific interface's default method wins over a less specific one (sub-interface wins rule).
- Default methods **cannot** access instance state unless the interface defines abstract getters. They have no fields.

**Best Practices**
- Use default methods for interface evolution and for providing convenience overloads. Avoid using them to simulate abstract classes with state — that leads to fragile designs.

**Common Follow-Up Questions**
1. *Can default methods be final?* — No. Interface methods cannot be `final`. Any implementing class can override them.
2. *What is the difference between default methods and abstract class methods?* — Abstract classes can have constructors, state (fields), and access modifiers; default methods cannot. A class can implement multiple interfaces but extend only one abstract class.
3. *Can you call a super default method?* — Yes, using `InterfaceName.super.methodName()`, but only from a direct implementor.

---

### What is the Stream API? What problem does it solve over traditional loops?

**Core Explanation**

The Stream API (`java.util.stream`) provides a **declarative, pipeline-based** approach to processing sequences of elements. A stream is not a data structure — it is a lazy pipeline of operations that draws elements from a source (collection, array, generator, I/O channel).

Problems it solves over traditional loops:
1. **Declarative intent** — describes *what* to compute, not *how* (no manual index management, temporary lists).
2. **Composability** — operations chain naturally (`filter().map().sorted().collect()`).
3. **Parallelism** — switch from sequential to parallel with `.parallelStream()` with no structural code changes.
4. **Lazy evaluation** — intermediate operations execute only when a terminal operation is invoked.

**Practical Example**

```java
// Traditional loop: find distinct cities of employees earning > 100K, sorted
Set<String> seen = new HashSet<>();
List<String> cities = new ArrayList<>();
for (Employee e : employees) {
    if (e.getSalary() > 100_000) {
        if (seen.add(e.getCity())) {
            cities.add(e.getCity());
        }
    }
}
Collections.sort(cities);

// Stream API
List<String> cities = employees.stream()
    .filter(e -> e.getSalary() > 100_000)
    .map(Employee::getCity)
    .distinct()
    .sorted()
    .collect(Collectors.toList()); // or .toList() in Java 16+
```

**Edge Cases / Pitfalls**
- Streams are **single-use**. Storing a stream in a variable and reusing it after a terminal operation throws `IllegalStateException`.
- `Stream.forEach()` does not guarantee order in parallel streams. Use `forEachOrdered()` if order matters.
- Side effects in intermediate operations (like `peek()` modifying state) are a design smell and can be unreliable in parallel streams.
- Streams of autoboxed primitives (`Stream<Integer>`) are slower than primitive streams (`IntStream`).

**Best Practices**
- Use streams for data transformation pipelines. Stick to traditional loops for simple iterations with side effects or when performance on tiny collections is critical.

**Common Follow-Up Questions**
1. *Can you create a stream from something other than a collection?* — Yes: `Stream.of()`, `Arrays.stream()`, `Stream.generate()`, `Stream.iterate()`, `BufferedReader.lines()`, `Files.lines()`, `Pattern.splitAsStream()`.
2. *What is a short-circuiting operation?* — Operations like `findFirst()`, `anyMatch()`, and `limit()` that do not need to process all elements.
3. *Are streams always better than loops?* — No. For simple iterations, small collections, or code that requires index access and mutation, loops are clearer and faster.

---

### What is `Optional`? How does it help avoid `NullPointerException`?

**Core Explanation**

`Optional<T>` is a container that may or may not contain a non-null value. It was introduced in Java 8 to provide a **type-level signal** that a value might be absent, forcing the caller to handle the absence explicitly rather than returning `null` and hoping the caller checks.

It shifts the burden from runtime `NullPointerException` to compile-time API design — a method returning `Optional<User>` clearly communicates "this might not return a user."

**Practical Example**

```java
// Without Optional — caller may forget null check
public User findUser(String id) {
    return userMap.get(id); // may return null
}
User user = findUser("123");
System.out.println(user.getName()); // NPE if user is null

// With Optional
public Optional<User> findUser(String id) {
    return Optional.ofNullable(userMap.get(id));
}

// Safe usage
String name = findUser("123")
    .map(User::getName)
    .orElse("Unknown");

// Chaining
String city = findUser("123")
    .flatMap(User::getAddress)       // Address is Optional
    .map(Address::getCity)
    .orElseThrow(() -> new UserNotFoundException("123"));
```

**Edge Cases / Pitfalls**
- `Optional.of(null)` throws `NullPointerException`. Use `Optional.ofNullable()` when the value may be null.
- `Optional.get()` without checking `isPresent()` defeats the purpose and still throws `NoSuchElementException`. Prefer `orElse()`, `orElseGet()`, or `orElseThrow()`.
- `orElse()` evaluates its argument eagerly. If the default is expensive, use `orElseGet(Supplier)` instead.
- `Optional` is not `Serializable`. Using it as a field or in a class that must be serialized will fail.

**Best Practices**
- Use `Optional` only as a return type for methods that might not produce a result. Never use `Optional` as a method parameter, field, or collection element.

**Common Follow-Up Questions**
1. *What is the difference between `orElse()` and `orElseGet()`?* — `orElse(default)` evaluates `default` eagerly every time; `orElseGet(() -> computeDefault())` evaluates the supplier lazily only when the Optional is empty.
2. *Can you use `Optional` with primitive types?* — Yes, use `OptionalInt`, `OptionalLong`, `OptionalDouble` to avoid autoboxing.
3. *Why not use `Optional` for fields?* — It adds memory overhead (extra object allocation), is not serializable, and was explicitly designed for return types. Use `@Nullable` annotation or null-object pattern for fields.

---

## Intermediate

---

### What is the difference between `map()` and `flatMap()` in Streams?

**Core Explanation**

Both are intermediate operations that transform elements, but they differ in how they handle the result:

- **`map(Function<T, R>)`** — applies the function to each element and wraps each result as a single element in the output stream. One input element produces exactly one output element. The result is `Stream<R>`.

- **`flatMap(Function<T, Stream<R>>)`** — applies the function to each element, where the function returns a stream for each element, and then **flattens** all those streams into a single stream. One input element can produce zero, one, or many output elements.

Think of it as: `map` is one-to-one; `flatMap` is one-to-many (then flattened).

**Practical Example**

```java
List<List<String>> nested = List.of(
    List.of("a", "b"),
    List.of("c", "d"),
    List.of("e")
);

// map — produces Stream<Stream<String>> (not useful)
Stream<Stream<String>> mapped = nested.stream()
    .map(Collection::stream);

// flatMap — produces Stream<String> (flattened)
List<String> flat = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList());
// Result: ["a", "b", "c", "d", "e"]

// Real-world: get all orders from all customers
List<Order> allOrders = customers.stream()
    .flatMap(c -> c.getOrders().stream())
    .collect(Collectors.toList());

// map would give Stream<List<Order>> — each element is still a list
```

**Edge Cases / Pitfalls**
- Using `map` when you need `flatMap` results in `Stream<Stream<T>>` or `Stream<List<T>>` — a nested structure you did not want.
- `flatMap` with `null`-returning functions causes `NullPointerException`. Return `Stream.empty()` instead of `null`.
- The function passed to `flatMap` should return a stream. To flatten `Optional` results, use `.flatMap(x -> optionalResult.stream())` (Java 9+ added `Optional.stream()`).

**Best Practices**
- Use `map` for one-to-one transformation. Use `flatMap` whenever the mapping function returns a collection, array, or optional that you want to unwrap/flatten.

**Common Follow-Up Questions**
1. *Can `flatMap` reduce the number of elements?* — Yes. If the function returns `Stream.empty()` for some elements, those elements are effectively filtered out.
2. *What about `flatMap` with `Optional.stream()` in Java 9+?* — `optionals.stream().flatMap(Optional::stream)` extracts present values and discards empties in one step.
3. *Is `flatMap` lazy?* — Yes, but each sub-stream is consumed eagerly when reached. Overall the pipeline remains lazy with respect to terminal operations.

---

### What is the difference between intermediate and terminal operations? Give examples of each.

**Core Explanation**

| Aspect | Intermediate | Terminal |
|---|---|---|
| **Returns** | A new `Stream` | A non-stream result (value, collection, void) |
| **Evaluation** | Lazy — nothing happens until a terminal operation | Eager — triggers pipeline execution |
| **Chaining** | Can chain multiple | Only one per pipeline (ends the stream) |

**Intermediate operations:** `filter()`, `map()`, `flatMap()`, `sorted()`, `distinct()`, `peek()`, `limit()`, `skip()`, `mapToInt()`

**Terminal operations:** `collect()`, `forEach()`, `reduce()`, `count()`, `findFirst()`, `findAny()`, `anyMatch()`, `allMatch()`, `noneMatch()`, `toArray()`, `min()`, `max()`, `toList()` (Java 16+)

**Short-circuiting** operations stop early: `limit()` (intermediate), `findFirst()`, `anyMatch()` (terminal).

**Practical Example**

```java
// This does NOTHING — no terminal operation
Stream<String> stream = names.stream()
    .filter(n -> {
        System.out.println("Filtering: " + n); // never printed
        return n.startsWith("A");
    })
    .map(String::toUpperCase);

// Adding a terminal operation triggers execution
List<String> result = names.stream()
    .filter(n -> n.startsWith("A"))  // intermediate
    .map(String::toUpperCase)        // intermediate
    .sorted()                        // intermediate (stateful)
    .collect(Collectors.toList());   // terminal — NOW it runs

// Short-circuiting example
boolean hasAdmin = users.stream()
    .filter(u -> u.getRole() == Role.ADMIN) // intermediate
    .findAny()                              // terminal, short-circuit
    .isPresent();
// Stops as soon as the first ADMIN is found
```

**Edge Cases / Pitfalls**
- Stateful intermediate operations (`sorted()`, `distinct()`) require buffering all elements internally, breaking the pure lazy streaming model.
- Calling only intermediate operations on a stream and never a terminal operation means the pipeline never executes and the stream silently leaks.
- `peek()` is an intermediate operation intended for debugging. It will not execute if the pipeline has no terminal operation.

**Best Practices**
- Always terminate a stream pipeline. Be aware of the distinction between stateless intermediates (`filter`, `map`) and stateful ones (`sorted`, `distinct`) for performance reasoning.

**Common Follow-Up Questions**
1. *Is `forEach()` intermediate or terminal?* — Terminal. It consumes the stream. There is no `forEach` that returns a stream; use `peek` for mid-pipeline side effects.
2. *What happens if you call two terminal operations on the same stream?* — `IllegalStateException: stream has already been operated upon or closed`.
3. *Can an intermediate operation cause eager evaluation?* — `sorted()` must consume all upstream elements before emitting, so it acts as a barrier, but it is still technically lazy until the terminal operation triggers it.

---

### What is the difference between `Optional.map()` and `Optional.flatMap()`?

**Core Explanation**

- **`Optional.map(Function<T, R>)`** — if the Optional contains a value, applies the function and wraps the result in a new `Optional`. If the result is `null`, returns `Optional.empty()`.

- **`Optional.flatMap(Function<T, Optional<R>>)`** — if the Optional contains a value, applies the function which **itself returns an Optional**, and returns that Optional directly (no double wrapping).

The distinction mirrors `Stream.map()` vs `Stream.flatMap()`. Use `flatMap` when the mapping function already returns an `Optional`.

**Practical Example**

```java
Optional<User> user = findUser("123");

// map — getName() returns String, result is Optional<String>
Optional<String> name = user.map(User::getName);

// map with a function returning Optional — BAD: gives Optional<Optional<Address>>
Optional<Optional<Address>> doubleWrapped = user.map(User::getAddress);

// flatMap — getAddress() returns Optional<Address>, result is Optional<Address>
Optional<Address> address = user.flatMap(User::getAddress);

// Chaining
String city = findUser("123")
    .flatMap(User::getAddress)   // getAddress returns Optional<Address>
    .map(Address::getCity)       // getCity returns String
    .orElse("Unknown");
```

**Edge Cases / Pitfalls**
- If the function passed to `flatMap` returns `null` (instead of `Optional.empty()`), a `NullPointerException` is thrown. The function **must** return an `Optional`.
- With `map`, if the function returns `null`, the result is `Optional.empty()` — it does not throw.
- Confusing `map` and `flatMap` leads to `Optional<Optional<T>>` nesting that is awkward to unwrap.

**Best Practices**
- Use `map` when the function returns a plain value. Use `flatMap` when the function returns an `Optional`. This keeps the chain flat and readable.

**Common Follow-Up Questions**
1. *Can you chain multiple `flatMap` calls?* — Yes, it is common: `opt.flatMap(A::getB).flatMap(B::getC).map(C::getValue)`.
2. *What does `Optional.map()` do if the Optional is empty?* — Returns `Optional.empty()` immediately without invoking the function.
3. *Is there `Optional.flatMap` for primitive optionals?* — No. `OptionalInt`, `OptionalLong`, `OptionalDouble` do not have `map` or `flatMap` methods.

---

### Why is `Optional` not recommended for fields, method parameters, or collections?

**Core Explanation**

`Optional` was designed as a **return type** for methods to signal "no result possible." Using it elsewhere is an anti-pattern for several reasons:

1. **Fields**: `Optional` is not `Serializable`. It adds an extra object allocation per field. It makes every access verbose (`field.orElse(...)`) without adding safety — the field itself could be `null`, giving you the same problem plus an extra layer.

2. **Method parameters**: It forces every caller to wrap arguments in `Optional.of()` or `Optional.empty()`, adding ceremony. The method should instead use overloading or `@Nullable` annotation. A parameter that is `Optional<String>` can itself be `null`, creating a paradox.

3. **Collections**: `List<Optional<String>>` is semantically confused. A collection already handles absence by simply not containing the element. Use `List<String>` and filter out nulls, or use `Map` where absence is indicated by key non-existence.

**Practical Example**

```java
// BAD — Optional as field
public class User {
    private Optional<String> nickname; // not serializable, extra allocation
}

// GOOD — nullable field with annotation
public class User {
    @Nullable
    private String nickname;

    public Optional<String> getNickname() { // Optional as return type is fine
        return Optional.ofNullable(nickname);
    }
}

// BAD — Optional as parameter
public void sendEmail(Optional<String> cc) { ... }
sendEmail(Optional.of("bob@co.com"));  // ugly for every caller
sendEmail(Optional.empty());

// GOOD — overloaded or nullable parameter
public void sendEmail(@Nullable String cc) { ... }
public void sendEmail() { sendEmail(null); } // overload

// BAD — Optional in collection
List<Optional<String>> names; // absurd

// GOOD
List<String> names; // just exclude nulls
```

**Edge Cases / Pitfalls**
- Some frameworks (e.g., Jackson, JPA) do not handle `Optional` fields well by default and require special modules or configuration.
- Using `Optional` as a field in a `record` works syntactically but still carries serialization and design issues.
- An `Optional` parameter that is itself `null` creates a bizarre "null Optional" scenario that defeats the purpose entirely.

**Best Practices**
- Return `Optional` from methods. Store plain values (with `@Nullable` annotations) in fields. Accept plain values (with overloads or null-checks) as parameters.

**Common Follow-Up Questions**
1. *What did Brian Goetz (Java architect) say about this?* — He explicitly stated that `Optional` was designed for return types only and should not be used for fields or parameters.
2. *Can `Optional` be used in DTOs for JSON APIs?* — It is possible with Jackson's `Jdk8Module`, but it adds coupling to `Optional` in your API contract. Plain nullable fields are simpler.
3. *What about `Optional` in records?* — Technically valid, but the record becomes non-serializable and the canonical constructor must handle both `null` and `Optional.empty()`.

---

### When should you avoid parallel streams? What are the performance pitfalls?

**Core Explanation**

Parallel streams use the common `ForkJoinPool` to split work across multiple threads. They are **not** a universal performance improvement. Avoid them when:

1. **Small collections** — Thread scheduling overhead dominates. Below ~10,000 elements for simple operations, sequential is often faster.
2. **Order-dependent operations** — `forEachOrdered()`, `limit()`, `findFirst()` in parallel streams negate parallelism benefits.
3. **Shared mutable state** — Accessing or modifying shared state from parallel stream operations causes race conditions.
4. **Blocking I/O in pipeline** — Blocks common `ForkJoinPool` threads, starving other parallel operations across the entire JVM.
5. **Linked data structures** — `LinkedList`, `TreeSet` have poor spliterator decomposition. `ArrayList` and arrays split well.
6. **Expensive merge (combiner)** — Operations like `Collectors.toList()` must merge partial results; this overhead can exceed the gains.
7. **Non-associative operations** — `reduce()` with non-associative operations gives wrong results in parallel.

**Practical Example**

```java
// BAD — small collection, overhead dominates
List<Integer> small = List.of(1, 2, 3, 4, 5);
int sum = small.parallelStream().mapToInt(i -> i).sum(); // slower than sequential

// BAD — shared mutable state
List<String> results = new ArrayList<>(); // not thread-safe
data.parallelStream()
    .filter(predicate)
    .forEach(results::add); // race condition!

// GOOD — use thread-safe collector instead
List<String> results = data.parallelStream()
    .filter(predicate)
    .collect(Collectors.toList()); // safe

// BAD — blocking I/O
urls.parallelStream()
    .map(url -> httpClient.get(url)) // blocks ForkJoinPool threads
    .collect(Collectors.toList());

// GOOD — use CompletableFuture with dedicated thread pool
ExecutorService pool = Executors.newFixedThreadPool(10);
List<CompletableFuture<Response>> futures = urls.stream()
    .map(url -> CompletableFuture.supplyAsync(() -> httpClient.get(url), pool))
    .collect(Collectors.toList());

// BAD — LinkedList splits poorly
LinkedList<Integer> linked = new LinkedList<>(data);
linked.parallelStream().map(...); // O(n) splitting
```

**Edge Cases / Pitfalls**
- The common `ForkJoinPool` is JVM-wide with `Runtime.getRuntime().availableProcessors() - 1` threads by default. One misbehaving parallel stream can starve all others.
- `Collectors.toMap()` in parallel streams can throw `ConcurrentModificationException` with some implementations.
- Benchmark before and after. The JMH microbenchmark harness is essential for reliable measurement.

**Best Practices**
- Default to sequential streams. Only switch to parallel after benchmarking proves a measurable benefit with large datasets (>10K elements) and CPU-bound, stateless operations on well-splitting data structures.

**Common Follow-Up Questions**
1. *Can you use a custom thread pool with parallel streams?* — Not directly via the API, but a workaround is to submit the stream operation to a custom `ForkJoinPool`: `customPool.submit(() -> data.parallelStream()...).get()`.
2. *What is a good data structure for parallel streams?* — `ArrayList`, arrays, `IntStream.range()`, and `HashMap` have excellent spliterator characteristics (known size, easy to split).
3. *Does `unordered()` help?* — Yes, calling `.unordered()` on an ordered stream (like from a `List`) allows optimizations for `distinct()`, `limit()`, and collection in parallel.

---

### What is the difference between `parallelStream()` and manual multithreading with `ExecutorService`?

**Core Explanation**

| Aspect | `parallelStream()` | `ExecutorService` |
|---|---|---|
| **Thread pool** | Uses common `ForkJoinPool` (shared JVM-wide) | Uses a dedicated, configurable pool |
| **Use case** | CPU-bound data-parallel transformations | I/O-bound tasks, async orchestration, mixed workloads |
| **Control** | Minimal — cannot easily set pool size, priorities, or timeouts per stream | Full control over pool size, queue type, rejection policy, shutdown |
| **Error handling** | Exceptions propagate to the calling thread | `Future.get()` wraps in `ExecutionException`; more explicit handling |
| **Work model** | Fork/join recursive decomposition | Task submission model |
| **Best for** | Stateless, CPU-heavy operations on large collections | I/O, heterogeneous tasks, fine-grained concurrency control |

**Practical Example**

```java
// parallelStream — CPU-bound, simple
long count = largeList.parallelStream()
    .filter(item -> expensiveCpuComputation(item))
    .count();

// ExecutorService — I/O-bound, controlled
ExecutorService pool = Executors.newFixedThreadPool(20);
List<Future<Response>> futures = urls.stream()
    .map(url -> pool.submit(() -> httpClient.get(url)))
    .collect(Collectors.toList());

for (Future<Response> f : futures) {
    try {
        Response r = f.get(5, TimeUnit.SECONDS); // timeout control
        process(r);
    } catch (TimeoutException e) {
        f.cancel(true);
    }
}
pool.shutdown();

// CompletableFuture with custom executor — best of both worlds
ExecutorService ioPool = Executors.newFixedThreadPool(20);
CompletableFuture<List<Response>> all = CompletableFuture.allOf(
    urls.stream()
        .map(url -> CompletableFuture.supplyAsync(() -> httpClient.get(url), ioPool))
        .toArray(CompletableFuture[]::new)
).thenApply(v -> /* collect results */);
```

**Edge Cases / Pitfalls**
- Mixing `parallelStream` and `ExecutorService` in the same application can lead to thread pool exhaustion if parallel streams block on I/O.
- `ExecutorService` requires explicit `shutdown()` to avoid thread leaks. Use try-with-resources in Java 19+ (`ExecutorService` implements `AutoCloseable`).
- `parallelStream` has no built-in timeout mechanism. Stuck operations block `ForkJoinPool` threads indefinitely.

**Best Practices**
- Use `parallelStream` only for CPU-bound, stateless data processing. Use `ExecutorService` (or `CompletableFuture` with a custom pool) for I/O-bound or mixed workloads where you need control over threading.

**Common Follow-Up Questions**
1. *Can you combine both?* — You can submit a parallel stream to a custom `ForkJoinPool`, but this is a workaround, not a designed feature.
2. *What about virtual threads (Java 21)?* — Virtual threads make `ExecutorService` ideal for I/O-heavy tasks with millions of concurrent tasks at minimal cost, making parallel streams even less relevant for I/O.
3. *Which is easier to debug?* — `ExecutorService` with named threads and explicit task boundaries is significantly easier to debug than parallel streams.

---

### How do lambdas affect debugging and stack traces?

**Core Explanation**

Lambdas produce **cryptic stack traces** because the JVM generates synthetic classes and methods at runtime. Instead of meaningful class/method names, you see entries like:

```
at com.example.Service.lambda$process$0(Service.java:45)
at java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:197)
```

The `lambda$methodName$index` naming makes it hard to identify which lambda in a method caused the error, especially when there are multiple lambdas on the same line or in chained stream pipelines.

**Practical Example**

```java
// Deeply chained stream — nightmare stack trace
List<String> result = employees.stream()
    .filter(e -> e.getDepartment() != null)
    .map(e -> e.getDepartment().getManager().getName()) // NPE here
    .filter(name -> name.length() > 3)
    .collect(Collectors.toList());

// Stack trace:
// java.lang.NullPointerException
//   at com.example.EmployeeService.lambda$process$1(EmployeeService.java:42)
//   at java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:197)
//   at java.util.ArrayList$ArrayListSpliterator.forEachRemaining(ArrayList.java:1625)
//   at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:509)
//   ... 15 more stream internals

// BETTER — extract to a named method
private String getManagerName(Employee e) {
    return e.getDepartment().getManager().getName();
}

List<String> result = employees.stream()
    .filter(e -> e.getDepartment() != null)
    .map(this::getManagerName) // stack trace shows "getManagerName"
    .filter(name -> name.length() > 3)
    .collect(Collectors.toList());
```

**Edge Cases / Pitfalls**
- Parallel streams make stack traces worse — frames from `ForkJoinPool` worker threads obscure the calling context.
- Lambda-generated class names vary across JVM implementations and versions. Do not rely on them for logging or reflection.
- Breakpoints inside lambdas work in modern IDEs (IntelliJ, Eclipse) but stepping through stream pipelines is non-linear — the debugger jumps between stream internals and your lambdas.

**Best Practices**
- Extract complex lambda bodies into named methods for clearer stack traces and easier debugging. Use `peek()` for temporary debugging but remove it before committing.

**Common Follow-Up Questions**
1. *Do method references have better stack traces than lambdas?* — Slightly, because the referenced method name appears directly in the trace.
2. *How can you debug stream pipelines effectively?* — Use `peek()` to inspect elements, or break the stream into multiple statements with intermediate variables. IntelliJ's "Trace Current Stream Chain" feature visualizes each step.
3. *Does Java 14+ Helpful NullPointerExceptions help?* — Yes, JEP 358 shows exactly which expression was null (e.g., "Cannot invoke getName() because the return value of getManager() is null"), which significantly helps even within lambdas.

---

### What are `Collectors` and how does `Collectors.groupingBy()` work?

**Core Explanation**

`Collectors` is a utility class providing factory methods for `Collector` instances used with `Stream.collect()`. A `Collector` defines how to accumulate stream elements into a mutable result container (list, set, map, string, etc.).

`Collectors.groupingBy()` partitions elements into groups based on a **classifier function**, producing a `Map<K, List<T>>` by default. It supports a downstream collector to transform each group further.

Three overloads:
1. `groupingBy(classifier)` — groups into `Map<K, List<T>>`
2. `groupingBy(classifier, downstream)` — applies downstream collector per group
3. `groupingBy(classifier, mapFactory, downstream)` — specifies map implementation

**Practical Example**

```java
List<Employee> employees = getEmployees();

// Basic grouping
Map<Department, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// With downstream: count per department
Map<Department, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));

// With downstream: average salary per department
Map<Department, Double> avgSalary = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

// Nested grouping: by department, then by city
Map<Department, Map<String, List<Employee>>> nested = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getCity)
    ));

// Group names by first letter, joining with comma
Map<Character, String> namesByLetter = names.stream()
    .collect(Collectors.groupingBy(
        n -> n.charAt(0),
        Collectors.joining(", ")
    ));

// Other useful collectors
String csv = names.stream().collect(Collectors.joining(", "));
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 100_000));
Map<String, Employee> byId = employees.stream()
    .collect(Collectors.toMap(Employee::getId, Function.identity()));
```

**Edge Cases / Pitfalls**
- `Collectors.toMap()` throws `IllegalStateException` on duplicate keys unless you provide a merge function.
- `groupingBy` does not allow `null` classifier results. If the classifier returns `null`, a `NullPointerException` is thrown. Pre-filter or map nulls to a sentinel value.
- The default `groupingBy` uses `HashMap`. Use the three-argument overload with `TreeMap::new` for sorted keys.

**Best Practices**
- Use `groupingBy` with downstream collectors for multi-level aggregation. For simple key-to-element maps, prefer `toMap` with explicit merge and map-factory parameters.

**Common Follow-Up Questions**
1. *What is the difference between `groupingBy` and `partitioningBy`?* — `partitioningBy` is a special case that groups into `Map<Boolean, List<T>>` using a predicate. It always has two keys (`true` and `false`).
2. *Can you write a custom Collector?* — Yes, implement `Collector<T, A, R>` with `supplier`, `accumulator`, `combiner`, `finisher`, and `characteristics`.
3. *What is `groupingByConcurrent`?* — A concurrent variant that produces a `ConcurrentMap` and works efficiently with parallel streams.

---

### Explain `reduce()` — how does the identity, accumulator, and combiner work in parallel streams?

**Core Explanation**

`reduce()` combines stream elements into a single result. It has three overloads:

1. **`reduce(BinaryOperator<T>)`** — no identity, returns `Optional<T>`
2. **`reduce(T identity, BinaryOperator<T>)`** — with identity value, returns `T`
3. **`reduce(U identity, BiFunction<U, T, U> accumulator, BinaryOperator<U> combiner)`** — for type-changing reductions, required for parallel correctness

**Key concepts:**
- **Identity**: The starting value that, when combined with any element, yields that element. E.g., `0` for addition, `1` for multiplication, `""` for concatenation.
- **Accumulator**: Combines the running result with the next element.
- **Combiner**: Merges two partial results (used only in parallel streams). Must be compatible with the accumulator.

**Practical Example**

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

// Simple reduce — sum
int sum = numbers.stream()
    .reduce(0, Integer::sum);
// 0 + 1 + 2 + 3 + 4 + 5 = 15

// Without identity — returns Optional
Optional<Integer> sum2 = numbers.stream()
    .reduce(Integer::sum);

// Three-argument reduce — type-changing (int to string length sum)
int totalLength = words.stream()
    .reduce(
        0,                              // identity
        (len, word) -> len + word.length(), // accumulator: int + String -> int
        Integer::sum                     // combiner: int + int -> int (parallel)
    );

// How parallel works:
// Thread 1: 0 + "hello".length() + "world".length() = 10
// Thread 2: 0 + "foo".length() + "bar".length() = 6
// Combiner: 10 + 6 = 16

// BAD identity — wrong results in parallel
int wrong = numbers.parallelStream()
    .reduce(10, Integer::sum);
// Sequential: 10+1+2+3+4+5 = 25
// Parallel: (10+1+2) + (10+3) + (10+4+5) = 45 — WRONG!
// Identity must be neutral: 0 for sum, not 10
```

**Edge Cases / Pitfalls**
- A wrong identity value silently produces incorrect results in parallel streams. The identity must satisfy: `accumulator(identity, t) == t` for all `t`.
- The accumulator must be **associative**: `(a op b) op c == a op (b op c)`. Non-associative operations (like subtraction) give wrong results in parallel.
- The combiner must be consistent with the accumulator: `combiner(u, accumulator(identity, t))` must equal `accumulator(u, t)`.
- For mutable reductions (building collections), use `collect()` with `Collectors` rather than `reduce()`.

**Best Practices**
- Use `reduce()` for immutable combining operations (sum, max, concatenation). Use `collect()` for mutable aggregation (building lists, maps). Always verify the identity is truly neutral.

**Common Follow-Up Questions**
1. *When is the combiner actually invoked?* — Only in parallel streams. In sequential streams, only the accumulator is used.
2. *Why not use `reduce()` to build a list?* — `reduce()` with list concatenation creates a new list on every step (O(n^2)). `collect(Collectors.toList())` uses a mutable accumulator (O(n)).
3. *What is the difference between `reduce()` and `collect()`?* — `reduce()` is for immutable folding; `collect()` is for mutable reduction with a mutable container (supplier + accumulator + combiner).

---

### What are text blocks (Java 13+)? Where are they most useful?

**Core Explanation**

Text blocks (preview in Java 13, final in Java 15) are multi-line string literals delimited by triple quotes (`"""`). They preserve formatting, handle line breaks automatically, and eliminate the need for escape sequences for quotes and newlines.

The compiler:
1. Normalizes line endings to `\n`
2. Strips common leading whitespace (incidental indentation)
3. Removes the trailing newline if the closing `"""` is on its own line

**Practical Example**

```java
// Before text blocks
String json = "{\n" +
    "  \"name\": \"John\",\n" +
    "  \"age\": 30,\n" +
    "  \"city\": \"New York\"\n" +
    "}";

// With text blocks
String json = """
        {
          "name": "John",
          "age": 30,
          "city": "New York"
        }
        """;

// SQL
String sql = """
        SELECT e.name, d.department_name
        FROM employees e
        JOIN departments d ON e.dept_id = d.id
        WHERE e.salary > ?
        ORDER BY e.name
        """;

// HTML
String html = """
        <html>
            <body>
                <p>Hello, %s</p>
            </body>
        </html>
        """.formatted(userName);

// Indentation control — closing """ position sets the baseline
String s = """
        indented""";  // no trailing newline, "indented" has no leading spaces

// Escape sequences still work
String escaped = """
        line1\
        continued on same line
        tab\there
        """;
// "line1continued on same line\ntab\there\n"
```

**Edge Cases / Pitfalls**
- The opening `"""` must be followed by a line break — no content on the same line.
- Trailing whitespace on each line is stripped by default. Use `\s` (Java escape) to preserve a trailing space.
- The `\` escape at the end of a line (Java 14+) continues the line without a newline.
- Moving the closing `"""` to the left adds leading indentation; moving it right has no effect (indentation is based on the minimum leading whitespace across all lines).

**Best Practices**
- Use text blocks for embedded SQL, JSON, HTML, XML, regex patterns, and multi-line messages. Use `.formatted()` or `String.format()` for interpolation.

**Common Follow-Up Questions**
1. *Does Java have string interpolation like Kotlin?* — Not yet. String templates (JEP 430) were previewed in Java 21 but withdrawn. Use `.formatted()` or `String.format()` for now.
2. *How does indentation stripping work?* — The compiler finds the minimum leading whitespace across all content lines and the closing `"""` line, then strips that many spaces from each line.
3. *Can text blocks be used with `+` concatenation?* — Yes, they are regular `String` instances and support all `String` operations.

---

### How does `var` (Java 10) help with type inference? What are its limitations and when should you avoid it?

**Core Explanation**

`var` (Java 10) enables **local variable type inference**. The compiler infers the variable's type from the right-hand side initializer. It is syntactic sugar — the bytecode is identical to explicit typing. It is **not** dynamic typing; the type is fixed at compile time.

Allowed contexts: local variables with initializers, `for`-loop variables, try-with-resources.

**Not allowed**: fields, method parameters, return types, `null` initializers, lambda parameters (though Java 11 added `var` in lambda parameters for annotation purposes).

**Practical Example**

```java
// Reduces boilerplate with verbose generic types
var map = new HashMap<String, List<Map<Integer, String>>>();
// instead of
HashMap<String, List<Map<Integer, String>>> map = new HashMap<>();

// For-loop
for (var entry : map.entrySet()) {
    var key = entry.getKey();
    var values = entry.getValue();
}

// Try-with-resources
try (var reader = new BufferedReader(new FileReader("data.txt"))) {
    var line = reader.readLine();
}

// Java 11: var in lambda parameters (enables annotations)
list.stream()
    .filter((@NotNull var s) -> s.length() > 3)
    .collect(Collectors.toList());

// AVOID — hides the type, reduces readability
var result = service.process(data);  // What type is result?
var x = condition ? getA() : getB(); // Type depends on common supertype

// GOOD — type is obvious from RHS
var users = new ArrayList<User>();
var name = "John";
var count = users.size();
```

**Edge Cases / Pitfalls**
- `var x = null;` does not compile — the type cannot be inferred.
- `var arr = {1, 2, 3};` does not compile — array initializers need an explicit type.
- `var x = condition ? 1 : "string";` — inferred type is `Object & Serializable & Comparable<...>`, which is rarely useful.
- `var` captures the **concrete** type, not the interface: `var list = new ArrayList<String>()` infers `ArrayList<String>`, not `List<String>`. This can leak implementation details.
- Diamond operator with `var`: `var list = new ArrayList<>()` infers `ArrayList<Object>` — not helpful.

**Best Practices**
- Use `var` when the type is obvious from the right-hand side (constructor calls, literals, factory methods). Avoid it when the type is not immediately clear or when programming to an interface is important.

**Common Follow-Up Questions**
1. *Is `var` a keyword?* — No, it is a **reserved type name**. You can still use `var` as a variable name (but please do not).
2. *Does `var` work with anonymous classes?* — Yes, and it infers the anonymous class type, which allows accessing members not defined in the supertype — a unique capability.
3. *Should teams standardize on `var` usage?* — Yes, establish guidelines. Common policy: use `var` when the type is clear from the RHS, and prefer explicit types for return values of method calls where the type is not obvious.

---

## Advanced

---

### What are sealed classes (Java 17)? How do they enable exhaustive pattern matching?

**Core Explanation**

Sealed classes restrict which classes can extend or implement them. Declared with the `sealed` keyword and a `permits` clause, they define a **closed set of subtypes**. Each permitted subtype must be `final`, `sealed`, or `non-sealed`.

This enables the compiler to know **all possible subtypes** at compile time, which allows:
- **Exhaustive `switch` expressions** — no `default` branch needed when all permitted subtypes are covered.
- **Pattern matching** — the compiler verifies completeness.
- **Domain modeling** — algebraic data types (sum types) like in Kotlin, Scala, or Rust.

**Practical Example**

```java
// Sealed hierarchy
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}

public record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
}

public final class Triangle implements Shape {
    private final double base, height;
    // constructor, area(), etc.
    public double area() { return 0.5 * base * height; }
}

// Exhaustive pattern matching (Java 21)
public String describe(Shape shape) {
    return switch (shape) {
        case Circle c    -> "Circle with radius " + c.radius();
        case Rectangle r -> "Rectangle " + r.width() + "x" + r.height();
        case Triangle t  -> "Triangle with area " + t.area();
        // No default needed — compiler knows all subtypes
    };
}

// If you add a new permitted subtype, ALL switch expressions break
// at compile time — forcing you to handle the new case.
// This is the key advantage over unsealed hierarchies.

// Sealed with non-sealed escape hatch
public sealed interface Payment permits CreditCard, BankTransfer, OtherPayment {}
public record CreditCard(String number) implements Payment {}
public record BankTransfer(String iban) implements Payment {}
public non-sealed interface OtherPayment extends Payment {}
// Anyone can implement OtherPayment — but switch must have a default
```

**Edge Cases / Pitfalls**
- All permitted subtypes must be in the same module (or same package if no modules).
- If subtypes are in the same file, the `permits` clause can be omitted — the compiler infers it.
- `non-sealed` breaks the closed hierarchy, requiring `default` in switch again.
- Sealed classes work with `instanceof` pattern matching but exhaustiveness checking is only in `switch` expressions.

**Best Practices**
- Use sealed classes to model closed domain types (state machines, AST nodes, result types). Combine with records for concise algebraic data types.

**Common Follow-Up Questions**
1. *Can abstract classes be sealed?* — Yes, both classes and interfaces can be sealed.
2. *What is the relationship between sealed classes and records?* — Records are implicitly final, making them ideal as permitted subtypes of sealed interfaces.
3. *How does this compare to enums?* — Enums are fixed instances with no state variation per instance. Sealed classes allow different data shapes per subtype (e.g., `Circle` has `radius`, `Rectangle` has `width` and `height`).

---

### How does pattern matching for `switch` (Java 17+) improve code over traditional `instanceof` chains?

**Core Explanation**

Pattern matching for `switch` (preview in Java 17, final in Java 21) replaces verbose `if-else instanceof` chains with concise, type-safe, exhaustive `switch` expressions. It combines type checking, casting, and binding into a single construct.

Key improvements:
1. **No manual casting** — the pattern variable is automatically typed.
2. **Exhaustiveness** — compiler ensures all cases are handled (with sealed types).
3. **Guarded patterns** — `when` clause adds conditions.
4. **Null handling** — explicit `case null` instead of separate null check.
5. **Dominance ordering** — compiler enforces correct case ordering.

**Practical Example**

```java
// BEFORE — instanceof chains
public String format(Object obj) {
    if (obj == null) {
        return "null";
    } else if (obj instanceof Integer i) {
        return String.format("int %d", i);
    } else if (obj instanceof Long l) {
        return String.format("long %d", l);
    } else if (obj instanceof String s) {
        return String.format("String \"%s\"", s);
    } else if (obj instanceof Collection<?> c && c.isEmpty()) {
        return "empty collection";
    } else if (obj instanceof Collection<?> c) {
        return String.format("collection of %d elements", c.size());
    } else {
        return obj.toString();
    }
}

// AFTER — pattern matching switch
public String format(Object obj) {
    return switch (obj) {
        case null                          -> "null";
        case Integer i                     -> String.format("int %d", i);
        case Long l                        -> String.format("long %d", l);
        case String s                      -> String.format("String \"%s\"", s);
        case Collection<?> c when c.isEmpty() -> "empty collection";
        case Collection<?> c               -> "collection of %d elements".formatted(c.size());
        default                            -> obj.toString();
    };
}

// With sealed classes — exhaustive, no default needed
sealed interface Result<T> permits Success, Failure {}
record Success<T>(T value) implements Result<T> {}
record Failure<T>(Exception error) implements Result<T> {}

public <T> String describe(Result<T> result) {
    return switch (result) {
        case Success<T> s -> "OK: " + s.value();
        case Failure<T> f -> "ERROR: " + f.error().getMessage();
    };
}

// Record patterns (Java 21) — destructure records in switch
record Point(int x, int y) {}

String describePoint(Object obj) {
    return switch (obj) {
        case Point(int x, int y) when x == 0 && y == 0 -> "origin";
        case Point(int x, int y) when x == 0 -> "on y-axis at " + y;
        case Point(int x, int y) -> "(%d, %d)".formatted(x, y);
        default -> "not a point";
    };
}
```

**Edge Cases / Pitfalls**
- Dominance: `case CharSequence cs` must come after `case String s`, not before — the compiler enforces this ordering.
- `case null` must be handled explicitly; without it, a null value throws `NullPointerException` (traditional behavior).
- The `when` clause replaced the earlier `&&` guard syntax from preview versions — older preview code may not compile.
- Pattern variables are scoped to their case branch. They cannot be accessed outside.

**Best Practices**
- Prefer pattern matching `switch` over `if-else instanceof` chains for more than two type checks. Combine with sealed types for compile-time exhaustiveness guarantees.

**Common Follow-Up Questions**
1. *Can you fall through in pattern switches?* — Arrow-form cases (`->`) do not fall through. Colon-form (`:`) can, but pattern variables do not scope across fall-through, so it is restricted.
2. *What are record patterns?* — Java 21 allows destructuring record components directly in the pattern: `case Point(int x, int y)`.
3. *Does this work with traditional enums?* — Yes, enum constants can be switch case labels and benefit from exhaustiveness checking.

---

### What are virtual threads (Java 21)? How do they differ from platform threads architecturally?

**Core Explanation**

Virtual threads (JEP 444, Java 21) are lightweight threads managed by the JVM, not the OS. They are **cheap to create** (a few hundred bytes of stack) and **cheap to block** — when a virtual thread blocks on I/O or `sleep()`, the JVM unmounts it from the carrier (platform) thread, freeing the carrier to run other virtual threads.

| Aspect | Platform Threads | Virtual Threads |
|---|---|---|
| **Managed by** | OS kernel | JVM scheduler |
| **Memory** | ~1MB stack each | ~few hundred bytes (grows on demand) |
| **Cost to create** | Expensive (syscall) | Cheap (Java object) |
| **Scalability** | Thousands | Millions |
| **Blocking cost** | Ties up OS thread | Unmounts from carrier, carrier reused |
| **Scheduling** | OS preemptive | JVM cooperative (yield on blocking) |
| **Carrier threads** | N/A | Platform threads in a `ForkJoinPool` |

**Practical Example**

```java
// Creating virtual threads
Thread vThread = Thread.startVirtualThread(() -> {
    System.out.println("Running on: " + Thread.currentThread());
});

// Using the builder
Thread vThread = Thread.ofVirtual()
    .name("worker-", 0)
    .start(() -> doWork());

// ExecutorService with virtual threads — one thread per task
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // Submit 100,000 tasks — each gets its own virtual thread
    List<Future<String>> futures = IntStream.range(0, 100_000)
        .mapToObj(i -> executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1)); // blocks, but cheap
            return fetchFromDb(i);
        }))
        .toList();

    for (var future : futures) {
        System.out.println(future.get());
    }
} // executor.close() waits for all tasks

// Structured concurrency (preview in Java 21)
// Ensures child tasks are bounded to parent scope
```

**Edge Cases / Pitfalls**
- Virtual threads are **not faster for CPU-bound work**. They shine for I/O-bound workloads where threads spend most of their time waiting.
- Thread-local variables work but can be expensive at scale (millions of threads). Consider scoped values (`ScopedValue`, preview) instead.
- `Thread.currentThread().isVirtual()` returns `true` for virtual threads.
- Virtual threads always have daemon status and `NORM_PRIORITY`. Setting priority has no effect.

**Best Practices**
- Use virtual threads for I/O-heavy server applications (HTTP handlers, database calls). Do not pool them — create one per task and let the JVM manage scheduling.

**Common Follow-Up Questions**
1. *Should you pool virtual threads?* — No. The whole point is that they are cheap. Create one per task with `Executors.newVirtualThreadPerTaskExecutor()`.
2. *Do existing libraries work with virtual threads?* — Most do, as long as they do not use `synchronized` on I/O-bound code paths (which pins the carrier thread).
3. *How do virtual threads interact with `ForkJoinPool`?* — Virtual threads use a dedicated `ForkJoinPool` as their carrier pool, separate from the common pool used by parallel streams.

---

### When would virtual threads NOT help? What are their limitations with `synchronized` and native calls?

**Core Explanation**

Virtual threads do not help — and can even hurt — in these scenarios:

1. **CPU-bound work**: Virtual threads yield only on blocking operations. CPU-intensive code monopolizes the carrier thread, negating the benefit. Use platform threads or parallel streams for CPU-bound tasks.

2. **`synchronized` blocks (pinning)**: When a virtual thread holds a `synchronized` monitor and then blocks inside it, the virtual thread is **pinned** to its carrier thread — the carrier cannot be reused. This effectively turns the virtual thread into a platform thread during that block.

3. **Native calls / JNI**: Native code runs on the carrier thread and cannot be unmounted. Blocking in native code pins the carrier.

4. **Thread-local abuse**: Creating millions of virtual threads, each with thread-local state, can cause massive memory consumption.

**Practical Example**

```java
// PROBLEM: synchronized pins the carrier thread
public class LegacyDao {
    private final Object lock = new Object();

    public String query(String sql) {
        synchronized (lock) {                // acquires monitor
            return jdbcTemplate.query(sql);  // blocks I/O WHILE holding monitor
            // Virtual thread is PINNED — carrier thread is stuck
        }
    }
}

// SOLUTION: Replace synchronized with ReentrantLock
public class ModernDao {
    private final ReentrantLock lock = new ReentrantLock();

    public String query(String sql) {
        lock.lock();                         // does NOT pin
        try {
            return jdbcTemplate.query(sql);  // virtual thread can unmount
        } finally {
            lock.unlock();
        }
    }
}

// PROBLEM: CPU-bound work — no benefit from virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // Each task does heavy CPU computation
    executor.submit(() -> computePrimes(1_000_000)); // no I/O, no yielding
    // Virtual threads just add overhead vs platform threads
}

// PROBLEM: Thread-local proliferation
ThreadLocal<byte[]> buffer = ThreadLocal.withInitial(() -> new byte[1024 * 1024]);
// With 1 million virtual threads = 1TB of thread-local buffers!

// Detect pinning with JFR or system property
// -Djdk.tracePinnedThreads=full  (prints stack trace on pinning)
// -Djdk.tracePinnedThreads=short (prints summary)
```

**Edge Cases / Pitfalls**
- Even short `synchronized` blocks are fine if they do not contain blocking I/O inside. The issue is specifically `synchronized` + blocking inside the block.
- `ReentrantLock` does not cause pinning and is the recommended replacement.
- Third-party libraries (JDBC drivers, HTTP clients) may internally use `synchronized`, causing hidden pinning. Check with `-Djdk.tracePinnedThreads=short`.
- File I/O does not currently unmount the virtual thread in all cases (implementation-dependent in early versions).

**Best Practices**
- Audit code for `synchronized` blocks that contain I/O before migrating to virtual threads. Use `-Djdk.tracePinnedThreads=short` in testing to detect pinning.

**Common Follow-Up Questions**
1. *Will `synchronized` eventually work without pinning?* — It is a goal for future JDK releases (Project Loom roadmap), but not guaranteed.
2. *What is `ScopedValue`?* — A preview API intended to replace `ThreadLocal` for virtual threads, with automatic lifecycle scoping.
3. *How do you monitor virtual thread performance?* — Use JFR (Java Flight Recorder) events: `jdk.VirtualThreadPinned`, `jdk.VirtualThreadStart`, `jdk.VirtualThreadEnd`.

---

### What is the difference between `record`, sealed class, and traditional POJO? When would you use each?

**Core Explanation**

| Aspect | Record | Sealed Class | Traditional POJO |
|---|---|---|---|
| **Purpose** | Immutable data carrier | Restricted type hierarchy | General-purpose mutable/immutable object |
| **Mutability** | Immutable (final fields) | Depends on implementation | Typically mutable |
| **Boilerplate** | Minimal (auto: constructor, getters, equals, hashCode, toString) | Normal class boilerplate | Full boilerplate (or Lombok) |
| **Inheritance** | Cannot extend classes, can implement interfaces | Controls which classes can extend | Full inheritance |
| **Pattern matching** | Destructurable in switch (Java 21) | Enables exhaustive switch | No special support |
| **When to use** | DTOs, value objects, events, messages | Closed type hierarchies, sum types | Entities, mutable state, complex behavior |

**Practical Example**

```java
// RECORD — immutable data carrier
public record Money(BigDecimal amount, Currency currency) {
    // Compact constructor for validation
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount must be non-negative");
        }
        Objects.requireNonNull(currency);
    }

    // Custom method
    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(amount.add(other.amount), currency);
    }
}

// SEALED CLASS — closed hierarchy
public sealed interface PaymentResult permits Approved, Declined, PendingReview {
}
public record Approved(String transactionId, Money amount) implements PaymentResult {}
public record Declined(String reason, String errorCode) implements PaymentResult {}
public record PendingReview(String reviewId) implements PaymentResult {}

// Exhaustive handling
String message = switch (result) {
    case Approved a    -> "Paid " + a.amount();
    case Declined d    -> "Failed: " + d.reason();
    case PendingReview p -> "Under review: " + p.reviewId();
};

// TRADITIONAL POJO — mutable JPA entity
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private String email;

    // JPA requires no-arg constructor
    protected User() {}

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    // Getters and setters (mutable)
    public void setEmail(String email) { this.email = email; }
    // equals, hashCode based on id
}
```

**Edge Cases / Pitfalls**
- Records cannot extend other classes (they implicitly extend `java.lang.Record`). They can implement interfaces.
- Record fields are `private final`, but the objects they reference can be mutable (`record Holder(List<String> items)` — the list is mutable). Defend with copies in the compact constructor.
- Records are not suitable for JPA entities (JPA requires mutable fields, no-arg constructors, proxying).
- Sealed classes require all subtypes in the same module/package.

**Best Practices**
- Use records for value objects and DTOs. Use sealed interfaces + records for domain-specific algebraic types. Use traditional POJOs when mutability, JPA, or framework proxying is required.

**Common Follow-Up Questions**
1. *Can records have additional fields?* — No. All fields must be declared in the record header (component list). You can add static fields and instance methods, but not instance fields.
2. *Can you override the canonical constructor?* — Yes, either with a compact constructor (no parameter list) or a full constructor.
3. *Do records work with Jackson/JSON?* — Yes, Jackson 2.12+ supports records out of the box, using the canonical constructor for deserialization.

---

### What is `CompletableFuture`? How does it differ from `Future`?

**Core Explanation**

`CompletableFuture<T>` (Java 8) is a composable, asynchronous programming primitive. It extends `Future<T>` with the ability to **chain callbacks**, **compose multiple async operations**, and **complete programmatically**.

| Aspect | `Future<T>` | `CompletableFuture<T>` |
|---|---|---|
| **Completion** | Set only by the `ExecutorService` | Can be completed manually (`complete()`, `completeExceptionally()`) |
| **Get result** | Only `get()` — blocking | `get()`, plus callbacks: `thenApply()`, `thenAccept()`, `thenRun()` |
| **Chaining** | Not possible | Full pipeline: `thenCompose()`, `thenCombine()`, `handle()` |
| **Error handling** | `ExecutionException` from `get()` | `exceptionally()`, `handle()`, `whenComplete()` |
| **Combining** | Manual | `allOf()`, `anyOf()` |

**Practical Example**

```java
// Future — blocking
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<String> future = pool.submit(() -> fetchFromApi());
String result = future.get(); // blocks calling thread

// CompletableFuture — non-blocking composition
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> fetchUserFromDb(userId), dbPool)
    .thenApply(user -> user.getEmail())
    .thenCompose(email -> sendEmailAsync(email))
    .exceptionally(ex -> {
        log.error("Failed", ex);
        return "fallback";
    });

// Combine multiple
CompletableFuture<String> userCf = CompletableFuture.supplyAsync(() -> fetchUser());
CompletableFuture<String> orderCf = CompletableFuture.supplyAsync(() -> fetchOrders());

CompletableFuture<String> combined = userCf.thenCombine(orderCf,
    (user, orders) -> user + " has " + orders);

// Wait for all
CompletableFuture.allOf(cf1, cf2, cf3).join();

// Manual completion
CompletableFuture<String> promise = new CompletableFuture<>();
// ... some callback later:
promise.complete("done");
// or: promise.completeExceptionally(new RuntimeException("oops"));
```

**Edge Cases / Pitfalls**
- `get()` wraps exceptions in `ExecutionException`; `join()` wraps in `CompletionException` (unchecked). Prefer `join()` in non-interruptible contexts.
- Without specifying an executor, `*Async` methods use the common `ForkJoinPool` — dangerous for blocking operations.
- A `CompletableFuture` that never completes causes `get()` to hang forever. Always use `get(timeout, unit)` in production or `orTimeout()` (Java 9+).
- Chaining many stages can create deep callback chains that are hard to debug.

**Best Practices**
- Always provide an explicit `Executor` to `*Async` methods for production code. Use `orTimeout()` and `completeOnTimeout()` (Java 9+) to prevent hanging futures.

**Common Follow-Up Questions**
1. *What is the difference between `join()` and `get()`?* — `join()` throws unchecked `CompletionException`; `get()` throws checked `ExecutionException` and `InterruptedException`.
2. *Can you cancel a CompletableFuture?* — `cancel()` exists but does not interrupt the running task. It only completes the future exceptionally with `CancellationException`.
3. *Is CompletableFuture still relevant with virtual threads?* — Yes, for composing async pipelines and non-blocking callbacks. But virtual threads reduce the need for callback-based async patterns since blocking becomes cheap.

---

### What is the difference between `thenApply()`, `thenCompose()`, and `thenCombine()`?

**Core Explanation**

| Method | Signature | Purpose | Analogy |
|---|---|---|---|
| `thenApply` | `CF<T> -> Function<T,U> -> CF<U>` | Transform the result synchronously | `Stream.map()` / `Optional.map()` |
| `thenCompose` | `CF<T> -> Function<T, CF<U>> -> CF<U>` | Chain another async operation (flatMap) | `Stream.flatMap()` / `Optional.flatMap()` |
| `thenCombine` | `CF<T> + CF<U> -> BiFunction<T,U,V> -> CF<V>` | Combine two independent futures | No direct stream equivalent |

- **`thenApply`**: One-to-one synchronous transformation. The function returns a plain value.
- **`thenCompose`**: One-to-one asynchronous transformation. The function returns a `CompletableFuture`. Avoids `CompletableFuture<CompletableFuture<T>>` nesting.
- **`thenCombine`**: Combines two independent futures that can run in parallel. The BiFunction runs when both complete.

**Practical Example**

```java
CompletableFuture<User> userFuture = fetchUserAsync(userId);

// thenApply — synchronous transformation (map)
CompletableFuture<String> nameFuture = userFuture
    .thenApply(user -> user.getName()); // User -> String

// thenCompose — async transformation (flatMap)
CompletableFuture<List<Order>> ordersFuture = userFuture
    .thenCompose(user -> fetchOrdersAsync(user.getId())); // User -> CF<List<Order>>

// BAD: using thenApply when the function returns a CompletableFuture
CompletableFuture<CompletableFuture<List<Order>>> nested = userFuture
    .thenApply(user -> fetchOrdersAsync(user.getId())); // CF<CF<List<Order>>> — wrong!

// thenCombine — two independent futures
CompletableFuture<User> userCf = fetchUserAsync(userId);
CompletableFuture<UserPrefs> prefsCf = fetchPreferencesAsync(userId);

CompletableFuture<ProfilePage> profileCf = userCf.thenCombine(prefsCf,
    (user, prefs) -> new ProfilePage(user, prefs));

// All three together in a pipeline
CompletableFuture<String> result = fetchUserAsync(userId)       // CF<User>
    .thenApply(User::getEmail)                                   // CF<String>
    .thenCompose(email -> sendVerificationAsync(email))          // CF<Token>
    .thenCombine(fetchConfigAsync(), (token, config) ->          // CF<String>
        buildResponse(token, config));
```

**Edge Cases / Pitfalls**
- Using `thenApply` with a function that returns `CompletableFuture` gives `CompletableFuture<CompletableFuture<T>>` — use `thenCompose` instead.
- `thenCombine` runs both futures concurrently, but the BiFunction runs after both complete. If one fails, the combined future fails.
- Each method has an `*Async` variant (`thenApplyAsync`, `thenComposeAsync`, `thenCombineAsync`) that runs the function on a different thread.

**Best Practices**
- Think of `thenApply` as `map` and `thenCompose` as `flatMap`. Use `thenCombine` to parallelize independent async operations that need to be merged.

**Common Follow-Up Questions**
1. *When should I use the `*Async` variants?* — When the transformation is expensive and you do not want to block the completing thread. Always pass a custom executor in production.
2. *Can I chain more than two futures?* — Use `allOf()` with a collection of futures, or chain `thenCombine` calls. For many futures, use `allOf` + stream collection.
3. *What happens if one future in `thenCombine` fails?* — The combined future completes exceptionally with the first exception. The other future continues running (no cancellation).

---

### Why can `CompletableFuture` silently swallow exceptions? How do you handle errors properly?

**Core Explanation**

`CompletableFuture` can silently swallow exceptions because:

1. **No one calls `get()` or `join()`** — if the future completes exceptionally but nothing observes the result, the exception is lost.
2. **Intermediate stages fail silently** — in a chain like `cf.thenApply(f1).thenApply(f2)`, if `f1` throws, `f2` is never called, but unless someone handles the final future, the exception vanishes.
3. **`thenAccept` / `thenRun` do not propagate** — they return `CompletableFuture<Void>`. If the resulting future is not observed, exceptions are lost.

Unlike `Thread.UncaughtExceptionHandler` for regular threads, there is **no global handler** for `CompletableFuture` exceptions.

**Practical Example**

```java
// SILENT SWALLOWING — exception lost
CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("Boom!"); // nobody observes this
});
// No error logged, no crash. Just silently gone.

// WRONG — incomplete error handling
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(result -> transform(result)); // exception from riskyOperation is swallowed

// CORRECT — handle errors at the end of the chain
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(result -> transform(result))
    .exceptionally(ex -> {
        log.error("Pipeline failed", ex);
        return fallbackValue;
    });

// CORRECT — handle() for both success and failure
CompletableFuture.supplyAsync(() -> riskyOperation())
    .handle((result, ex) -> {
        if (ex != null) {
            log.error("Failed", ex);
            return fallbackValue;
        }
        return transform(result);
    });

// CORRECT — whenComplete() for side effects (logging) without changing the result
CompletableFuture.supplyAsync(() -> riskyOperation())
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("Failed", ex);
            metrics.increment("pipeline.failure");
        }
    })
    .exceptionally(ex -> fallbackValue);

// PATTERN — always terminate chains with error handling
public CompletableFuture<Response> processRequest(Request req) {
    return validateAsync(req)
        .thenCompose(this::enrichAsync)
        .thenCompose(this::persistAsync)
        .thenApply(this::toResponse)
        .exceptionally(ex -> {
            log.error("Request processing failed for {}", req.getId(), ex);
            return Response.error(ex.getMessage());
        });
}
```

**Edge Cases / Pitfalls**
- `exceptionally()` receives the wrapped exception. For `supplyAsync` failures, it is a `CompletionException` wrapping the real cause — use `ex.getCause()`.
- `handle()` is called for both success and failure. It replaces the result, so a successful handle after a failure "recovers" the future.
- `whenComplete()` does not alter the result. If you throw inside `whenComplete()`, it replaces the exception but not a successful result.
- `allOf()` completes exceptionally if **any** future fails, but only with the first exception. Other exceptions are lost unless you check each future individually.

**Best Practices**
- Always add `exceptionally()` or `handle()` at the end of every `CompletableFuture` chain. Use `whenComplete()` for logging without altering the pipeline result.

**Common Follow-Up Questions**
1. *How do you propagate exceptions to callers?* — Return the `CompletableFuture` to the caller and let them handle it, or use `join()` which throws `CompletionException`.
2. *How do you handle exceptions from `allOf()`?* — After `allOf().join()`, iterate over each future and check `isCompletedExceptionally()` or call `join()` on each.
3. *Is there a global exception handler for CompletableFuture?* — No. Each chain must handle its own exceptions. You can create a utility method like `withErrorLogging(CompletableFuture<T>)` to standardize this.

---

### What are common production mistakes with `CompletableFuture` (wrong thread pool, blocking calls)?

**Core Explanation**

Common production mistakes:

1. **Using the common `ForkJoinPool` for blocking I/O** — the default executor for `*Async` methods is the common pool, which has limited threads. Blocking calls starve the pool.

2. **Blocking in async chains** — calling `future.get()` or `Thread.sleep()` inside `thenApply()` blocks the executing thread.

3. **Not specifying an executor** — relying on the default pool leads to unpredictable behavior under load.

4. **Not handling exceptions** — as discussed, silently swallowed exceptions.

5. **Creating too many stages** — long chains are hard to debug and maintain.

6. **Forgetting `orTimeout()`** — futures that never complete cause resource leaks.

7. **Using `thenApply` instead of `thenCompose`** — leading to nested futures.

**Practical Example**

```java
// MISTAKE 1: Blocking I/O on common ForkJoinPool
CompletableFuture.supplyAsync(() -> {
    return restTemplate.getForObject(url, String.class); // blocks FJP thread
});

// FIX: Use dedicated I/O pool
ExecutorService ioPool = Executors.newFixedThreadPool(20);
CompletableFuture.supplyAsync(() -> {
    return restTemplate.getForObject(url, String.class);
}, ioPool); // uses dedicated pool

// MISTAKE 2: Blocking inside thenApply
CompletableFuture.supplyAsync(() -> getUserId())
    .thenApply(id -> {
        return fetchUser(id).get(); // BLOCKING inside async chain!
    });

// FIX: Use thenCompose for async chaining
CompletableFuture.supplyAsync(() -> getUserId())
    .thenCompose(id -> fetchUserAsync(id)); // non-blocking

// MISTAKE 3: No timeout
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> slowService.call());
String result = cf.join(); // hangs forever if slowService never returns

// FIX: Add timeout (Java 9+)
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> slowService.call())
    .orTimeout(5, TimeUnit.SECONDS)           // throws TimeoutException
    .completeOnTimeout("default", 5, TimeUnit.SECONDS); // or provide default

// MISTAKE 4: Unbounded concurrency
List<CompletableFuture<Void>> futures = orders.stream()  // 100,000 orders
    .map(o -> CompletableFuture.runAsync(() -> process(o)))
    .toList();
// Floods common ForkJoinPool with 100K tasks

// FIX: Use bounded executor or semaphore
Semaphore limiter = new Semaphore(50);
ExecutorService pool = Executors.newFixedThreadPool(50);
orders.stream()
    .map(o -> CompletableFuture.runAsync(() -> {
        limiter.acquireUninterruptibly();
        try { process(o); } finally { limiter.release(); }
    }, pool))
    .toList();

// MISTAKE 5: Fire-and-forget without error handling
CompletableFuture.runAsync(() -> auditLog(event)); // exception? who knows.

// FIX
CompletableFuture.runAsync(() -> auditLog(event))
    .exceptionally(ex -> {
        log.error("Audit logging failed", ex);
        return null;
    });
```

**Edge Cases / Pitfalls**
- Thread pool sizing: I/O-bound pools should be larger (20-200 threads); CPU-bound pools should match core count.
- The common `ForkJoinPool` default size is `Runtime.getRuntime().availableProcessors() - 1`. On a 4-core machine, that is 3 threads for the entire JVM.
- `ExecutorService` created with `Executors.newFixedThreadPool()` has an unbounded queue — tasks pile up in memory under high load. Use `ThreadPoolExecutor` with bounded queues in production.

**Best Practices**
- Always pass a dedicated `Executor` to `*Async` methods. Size thread pools based on workload type (I/O vs CPU). Add timeouts and error handling to every chain.

**Common Follow-Up Questions**
1. *How do you size the I/O thread pool?* — A common formula: `threads = numCores * (1 + waitTime/computeTime)`. For high-latency I/O, this can be 10-100x the core count.
2. *Should you reuse one pool or have multiple?* — Separate pools for different workloads (DB, HTTP, messaging) prevents one slow service from starving others (bulkhead pattern).
3. *Does Spring's `@Async` have the same issues?* — Yes. By default it uses a `SimpleAsyncTaskExecutor` (no pooling). Always configure a `TaskExecutor` bean.

---

### How does the common `ForkJoinPool` impact async APIs when used with `CompletableFuture`?

**Core Explanation**

The common `ForkJoinPool` (`ForkJoinPool.commonPool()`) is a JVM-wide shared pool used by:
- `CompletableFuture.*Async()` methods (when no executor is specified)
- Parallel streams (`.parallelStream()`)
- `Arrays.parallelSort()`

It has `Runtime.getRuntime().availableProcessors() - 1` threads by default. Because it is **shared globally**, abuse in one part of the application affects all other users of the pool.

**Impact on async APIs:**
1. **Thread starvation**: Blocking I/O in the common pool (DB calls, HTTP requests) blocks all threads, causing parallel streams and other async operations to stall.
2. **Latency spikes**: Under load, tasks queue up behind blocked threads, causing unpredictable latency.
3. **Cascading failures**: One misbehaving library using the common pool can bring down unrelated features.

**Practical Example**

```java
// Scenario: three independent systems share the common pool

// System A: parallel stream (CPU-bound, legitimate)
employees.parallelStream()
    .filter(e -> expensiveValidation(e))
    .collect(Collectors.toList());

// System B: CompletableFuture without explicit executor (BAD)
CompletableFuture.supplyAsync(() -> {
    return httpClient.get("https://slow-api.com/data"); // blocks 2+ seconds
});

// System C: Another CompletableFuture
CompletableFuture.supplyAsync(() -> {
    return database.query("SELECT ..."); // also blocks
});

// On a 4-core machine: common pool has 3 threads
// If Systems B and C block all 3 threads on I/O,
// System A's parallel stream STALLS until a thread frees up.

// FIX: Dedicated pools per workload
ExecutorService httpPool = Executors.newFixedThreadPool(10,
    new ThreadFactory() {
        private final AtomicInteger counter = new AtomicInteger();
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "http-pool-" + counter.incrementAndGet());
            t.setDaemon(true);
            return t;
        }
    });

ExecutorService dbPool = Executors.newFixedThreadPool(20);

// Now each system uses its own pool
CompletableFuture.supplyAsync(() -> httpClient.get(url), httpPool);
CompletableFuture.supplyAsync(() -> database.query(sql), dbPool);
// Parallel streams still use the common pool unimpeded

// You CAN adjust the common pool size (but this is a global JVM setting)
// -Djava.util.concurrent.ForkJoinPool.common.parallelism=16
// This affects ALL users of the common pool
```

**Edge Cases / Pitfalls**
- The common pool parallelism is `max(1, availableProcessors() - 1)`. On a single-core machine (or container with 1 CPU), it is 1 thread.
- In containerized environments (Docker, Kubernetes), `availableProcessors()` may return the host's CPU count, not the container's limit. Use `-XX:ActiveProcessorCount=N` to fix this.
- `ForkJoinPool.commonPool()` uses daemon threads. If the main thread exits, pending tasks are killed.
- Setting system property `java.util.concurrent.ForkJoinPool.common.parallelism` affects the entire JVM and cannot be changed at runtime.

**Best Practices**
- Never perform blocking I/O on the common `ForkJoinPool`. Reserve it for CPU-bound work (parallel streams). Use dedicated thread pools for all `CompletableFuture` I/O operations.

**Common Follow-Up Questions**
1. *How do you monitor the common pool?* — `ForkJoinPool.commonPool().getActiveThreadCount()`, `.getQueuedTaskCount()`, `.getStealCount()`. Expose these as metrics.
2. *Can you replace the common pool?* — Not directly. You can set its parallelism and thread factory via system properties but cannot swap the implementation.
3. *Does this problem go away with virtual threads?* — Partially. Virtual threads eliminate the I/O blocking problem since blocking is cheap, but `CompletableFuture` still uses the common pool for callbacks unless you specify otherwise.

---

### How do Streams work internally (lazy evaluation, spliterator, short-circuiting)?

**Core Explanation**

Streams are implemented as a **pipeline of stages**, evaluated lazily. The key internal components are:

1. **Spliterator**: The source of elements. Wraps the underlying data structure and provides `tryAdvance()`, `trySplit()`, `estimateSize()`, and `characteristics()`. The characteristics (ORDERED, SIZED, SORTED, DISTINCT, etc.) enable optimizations.

2. **Lazy evaluation**: Intermediate operations build a linked list of pipeline stages (`ReferencePipeline`). No computation occurs until a terminal operation is invoked. Each element flows through the entire pipeline before the next element is processed (loop fusion).

3. **Short-circuiting**: Operations like `findFirst()`, `anyMatch()`, and `limit()` can terminate the pipeline early by canceling the spliterator traversal.

4. **Sink chain**: Terminal operations create a chain of `Sink` objects (one per stage). Each sink's `accept()` method processes the element and passes it downstream. This is the internal push-based execution model.

**Practical Example**

```java
// Demonstrating laziness
List<String> names = List.of("Alice", "Bob", "Charlie", "Diana");

names.stream()
    .filter(n -> {
        System.out.println("filter: " + n);
        return n.length() > 3;
    })
    .map(n -> {
        System.out.println("map: " + n);
        return n.toUpperCase();
    })
    .findFirst();

// Output (lazy, element-by-element, short-circuits):
// filter: Alice
// map: Alice
// Result: Optional[ALICE]
// Bob, Charlie, Diana are NEVER processed

// Spliterator example
List<Integer> list = List.of(1, 2, 3, 4, 5, 6, 7, 8);
Spliterator<Integer> spliterator = list.spliterator();

System.out.println(spliterator.estimateSize());       // 8
System.out.println(spliterator.characteristics());    // ORDERED | SIZED | SUBSIZED

Spliterator<Integer> secondHalf = spliterator.trySplit(); // splits for parallelism
// spliterator now covers [5,6,7,8], secondHalf covers [1,2,3,4]

// How parallel streams use spliterators:
// 1. ForkJoinPool calls trySplit() recursively until chunks are small enough
// 2. Each chunk is processed by a separate thread through the sink chain
// 3. Results are combined using the collector's combiner

// Stateful operations break loop fusion
names.stream()
    .filter(n -> n.length() > 3)   // stateless — fused
    .sorted()                       // STATEFUL — must buffer ALL upstream elements
    .map(String::toUpperCase)       // stateless — fused with downstream
    .forEach(System.out::println);
// sorted() acts as a barrier: everything before it runs first,
// then everything after it runs on the sorted buffer.
```

**Edge Cases / Pitfalls**
- `sorted()` and `distinct()` are stateful and break pure laziness. `sorted()` must consume all elements before producing any output.
- `flatMap()` eagerly consumes each sub-stream before moving to the next element (a known JDK implementation detail).
- Spliterators with `SIZED` characteristic enable `toArray()` to allocate the right-sized array upfront. Without it, the stream must grow arrays dynamically.
- Infinite streams (`Stream.generate()`, `Stream.iterate()`) work fine with short-circuiting operations but hang with terminal operations like `count()` or `collect()`.

**Best Practices**
- Understand that lazy evaluation means side effects in intermediate operations may never execute. Place side effects only in terminal operations. Be aware that `sorted()` is a full barrier in the pipeline.

**Common Follow-Up Questions**
1. *What is loop fusion?* — The JVM processes each element through the entire pipeline (filter-map-collect) before fetching the next element, rather than running filter on all elements, then map on all elements, etc.
2. *Can you create a custom Spliterator?* — Yes, implement `Spliterator<T>` for custom data sources (database cursors, file readers, network streams).
3. *What characteristics does the stream maintain?* — Characteristics flow from the source spliterator and can be added/removed by operations. `filter()` clears SIZED; `sorted()` adds SORTED; `distinct()` adds DISTINCT.

---

### What is the performance impact of Streams vs loops for small collections?

**Core Explanation**

For small collections (fewer than ~1,000 elements), traditional `for` loops are **measurably faster** than streams due to:

1. **Object allocation overhead**: Each stream operation creates pipeline objects (`ReferencePipeline`, `Sink` instances, lambdas). For a 10-element list, this overhead dominates the actual computation.

2. **No inlining**: The JIT compiler can inline simple loop bodies easily but struggles with the indirection layers in stream pipelines (virtual dispatch through `Sink.accept()`).

3. **Autoboxing**: `Stream<Integer>` boxes/unboxes constantly. Primitive loops avoid this entirely. `IntStream` helps but still has pipeline overhead.

4. **Megamorphic call sites**: Stream internals use polymorphic dispatch that can prevent JIT optimizations.

Benchmark data (approximate, varies by JVM and hardware):
- **Summing 10 integers**: loop ~5ns, stream ~50-100ns (10-20x slower)
- **Summing 10,000 integers**: loop ~5us, stream ~6-8us (1.2-1.6x slower)
- **Summing 1,000,000 integers**: loop ~500us, stream ~500-600us (negligible difference)

**Practical Example**

```java
// For small collections — loop is significantly faster
int[] small = {1, 2, 3, 4, 5};

// Loop: ~5ns
int sum = 0;
for (int i : small) {
    sum += i;
}

// Stream: ~50-100ns (pipeline setup dominates)
int sum = Arrays.stream(small).sum();

// For large collections — difference is negligible
int[] large = new int[1_000_000];

// Loop: ~500us
int sum = 0;
for (int i : large) sum += i;

// IntStream: ~500-600us
int sum = Arrays.stream(large).sum();

// Autoboxing penalty with Stream<Integer>
List<Integer> boxed = List.of(1, 2, 3, 4, 5);

// BAD — autoboxing in stream
int sum = boxed.stream()
    .reduce(0, Integer::sum); // unboxes each Integer

// BETTER — use mapToInt to avoid boxing in reduction
int sum = boxed.stream()
    .mapToInt(Integer::intValue)
    .sum();

// JMH benchmark setup (the right way to measure)
@Benchmark
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public int loopSum(MyState state) {
    int sum = 0;
    for (int i : state.array) sum += i;
    return sum;
}

@Benchmark
public int streamSum(MyState state) {
    return Arrays.stream(state.array).sum();
}
```

**Edge Cases / Pitfalls**
- **Microbenchmark traps**: Never use `System.nanoTime()` in a loop to benchmark. Use JMH (Java Microbenchmark Harness) to account for JIT warmup, dead code elimination, and GC pauses.
- The JIT compiler improves stream performance over time (warmup). First invocations are slower; after JIT compilation, the gap narrows.
- Parallel streams on small collections are **always slower** — thread scheduling overhead far exceeds any parallelism benefit.
- The readability benefit of streams often outweighs the nanosecond-level performance difference in application code that is not on a hot path.

**Best Practices**
- Optimize for readability first. Only switch from streams to loops in proven hot paths (identified by profiling). For primitive-heavy computations on small arrays, prefer plain loops or `IntStream`.

**Common Follow-Up Questions**
1. *At what collection size do streams become competitive?* — Generally around 1,000-10,000 elements for simple operations. For complex multi-step pipelines, streams may be competitive earlier because they avoid intermediate collections.
2. *Does GraalVM change this?* — GraalVM's compiler can sometimes optimize stream pipelines better than HotSpot C2, reducing the gap further.
3. *Should I refactor all small-collection streams to loops?* — No. Only refactor if profiling shows the stream is on a hot path and the overhead is measurable in context. Readability matters more in 99% of code.
# 4. Core Java — JVM Internals & Memory Management

---

## Basic

---

### What is the difference between JDK, JRE, and JVM?

**Core Explanation**

| Component | What It Is | Contains |
|-----------|-----------|----------|
| **JVM** (Java Virtual Machine) | Abstract specification + runtime engine that executes bytecode | Interpreter, JIT compiler, GC, runtime data areas |
| **JRE** (Java Runtime Environment) | JVM + standard class libraries needed to *run* Java programs | JVM + `rt.jar` / modules, `java` launcher |
| **JDK** (Java Development Kit) | JRE + development tools needed to *build* Java programs | JRE + `javac`, `jdb`, `jlink`, `jpackage`, JFR, JMC |

Relationship: **JDK ⊃ JRE ⊃ JVM**

Since Java 11, Oracle no longer ships a standalone JRE. The JDK is the single deliverable, and `jlink` can create custom minimal runtimes.

**Practical Example**

```java
// Compile (requires JDK)
// javac Hello.java

// Run (requires JRE, which includes JVM)
// java Hello

// At runtime the JVM:
//   1. Loads Hello.class via ClassLoader
//   2. Verifies bytecode
//   3. Interprets / JIT-compiles
//   4. Manages memory via GC
```

**Edge Cases / Pitfalls**
- Deploying a JDK in production is common but exposes tools like `jstack`, `jmap`, etc. This is actually beneficial for diagnostics but increases image size in containers.
- `JAVA_HOME` must point to the JDK for build tools (Maven/Gradle) to work; pointing to a JRE causes `javac` not-found errors.

**Best Practices**
Use the JDK in development and CI; for production containers use `jlink` to create a minimal custom runtime image containing only the modules your application needs.

**Common Follow-Up Questions**
1. **What changed with the JRE in Java 11+?** Oracle stopped shipping a separate JRE. Use `jlink` to build custom runtimes.
2. **Is the JVM specification platform-dependent?** The specification is platform-independent; the *implementation* (HotSpot, OpenJ9, GraalVM) is platform-specific.
3. **Can you run Java code without a JDK?** Yes, with a JRE (pre-Java 11) or a custom runtime. You only need the JDK to compile `.java` to `.class`.

---

### What are the main memory areas in JVM (Heap, Stack, Metaspace, Code Cache)?

**Core Explanation**

| Area | Scope | Stores | Thread Safety |
|------|-------|--------|---------------|
| **Heap** | Shared across all threads | Object instances, arrays | Shared; GC-managed |
| **Stack** (per thread) | Private to each thread | Frames: local variables, operand stack, return address | Thread-private; no GC |
| **Metaspace** (off-heap, native) | Shared | Class metadata, method bytecode, constant pool | Shared; grows dynamically |
| **Code Cache** (native) | Shared | JIT-compiled native code | Shared; managed by JVM |

Additional areas:
- **Program Counter (PC) Register** — per thread, points to the current bytecode instruction.
- **Native Method Stack** — per thread, for JNI calls.
- **Direct Memory** — off-heap buffers allocated via `ByteBuffer.allocateDirect()`.

**Practical Example**

```java
public class MemoryDemo {
    // Class metadata -> Metaspace
    private static final String CONSTANT = "hello"; // String literal -> String Pool (Heap since Java 7)

    public void process() {
        int x = 42;                     // x -> Stack frame
        Object obj = new Object();      // reference 'obj' -> Stack; Object instance -> Heap
        byte[] buf = new byte[1024];    // reference -> Stack; byte array -> Heap
    }
}
```

**Edge Cases / Pitfalls**
- Metaspace grows unboundedly by default. Without `-XX:MaxMetaspaceSize`, dynamically generated classes (e.g., heavy reflection, CGLIB proxies) can exhaust native memory.
- Code Cache exhaustion silently disables JIT compilation, causing severe performance degradation. Monitor with `-XX:+PrintCodeCache`.
- Direct `ByteBuffer` memory is NOT part of the heap and is NOT controlled by `-Xmx`.

**Best Practices**
Always set `-XX:MaxMetaspaceSize` and `-XX:ReservedCodeCacheSize` in production; monitor direct memory with `-XX:MaxDirectMemorySize` when using NIO.

**Common Follow-Up Questions**
1. **Where does the String Pool live?** In the heap since Java 7 (it was in PermGen before).
2. **What is the default stack size?** Platform-dependent, typically 512 KB to 1 MB. Controlled by `-Xss`.
3. **Can you have an OOM from Metaspace?** Yes: `java.lang.OutOfMemoryError: Metaspace`.

---

### What is garbage collection? Why is manual memory management unnecessary in Java?

**Core Explanation**

Garbage Collection (GC) is the JVM's automatic process of identifying and reclaiming memory occupied by objects that are no longer reachable from any GC root. GC roots include: local variables on thread stacks, static fields, active threads, and JNI references.

Manual memory management (as in C/C++) is unnecessary because:
1. The JVM tracks object reachability automatically.
2. The GC reclaims unreachable objects without programmer intervention.
3. This eliminates entire classes of bugs: dangling pointers, double frees, use-after-free.

The trade-off is **GC pauses** (stop-the-world events) and reduced control over deallocation timing.

**Practical Example**

```java
public void example() {
    StringBuilder sb = new StringBuilder("data");
    // 'sb' is reachable here — the StringBuilder is alive.

    sb = null;
    // The StringBuilder object is now unreachable — eligible for GC.
    // The JVM will reclaim its memory at some future GC cycle.
    // No explicit free() or delete needed.
}
```

**Edge Cases / Pitfalls**
- GC does not guarantee *when* an object is collected, only that it will be collected before an `OutOfMemoryError` is thrown.
- `finalize()` is unreliable and deprecated (Java 9+). Use `Cleaner` or try-with-resources instead.
- Objects can be eligible for GC even while a method is still executing, if the JVM proves the reference is no longer used (reachability analysis, not scope-based).

**Best Practices**
Trust the GC but help it: minimize object allocation in hot loops, avoid retaining large collections longer than necessary, and prefer short-lived objects (they are cheaply collected in the Young Generation).

**Common Follow-Up Questions**
1. **Can you force garbage collection?** `System.gc()` is a *hint*; the JVM may ignore it. Never rely on it in production.
2. **What is a GC root?** A starting point for reachability analysis — stack locals, static fields, active threads, JNI references, class references from classloaders.
3. **Is Java immune to memory leaks?** No. Logical leaks happen when objects are reachable but no longer needed (e.g., entries left in a growing `HashMap`).

---

### What is the difference between `OutOfMemoryError` and `StackOverflowError`?

**Core Explanation**

| Aspect | `OutOfMemoryError` | `StackOverflowError` |
|--------|--------------------|-----------------------|
| Memory area | Heap, Metaspace, native, or direct memory | Thread stack |
| Cause | JVM cannot allocate more memory for objects/classes | Thread stack depth exceeds limit |
| Typical trigger | Large/numerous objects, memory leaks | Unbounded or deep recursion |
| Scope | Usually fatal for the JVM | Usually fatal for the thread only |
| Both extend | `java.lang.VirtualMachineError` (subclass of `Error`) | Same |

**Practical Example**

```java
// StackOverflowError — unbounded recursion
public static int factorial(int n) {
    return n * factorial(n - 1); // no base case -> StackOverflowError
}

// OutOfMemoryError — heap exhaustion
public static void oom() {
    List<byte[]> leak = new ArrayList<>();
    while (true) {
        leak.add(new byte[1_048_576]); // 1 MB each, never released
    }
}
```

**Edge Cases / Pitfalls**
- `OutOfMemoryError` can also occur for native memory (`unable to create new native thread`), Metaspace, or direct buffers — not just heap.
- Catching `OutOfMemoryError` is rarely useful because the JVM may be in an inconsistent state. If you do catch it, perform minimal cleanup (log and exit).
- `StackOverflowError` can happen in non-recursive code if the call chain is extremely deep (e.g., deeply nested library calls or expression evaluation engines).

**Best Practices**
Use `-Xmx` to set appropriate heap limits; use `-Xss` to increase stack size only when truly needed (deep recursion). Convert deep recursion to iteration when possible.

**Common Follow-Up Questions**
1. **Can you recover from an `OutOfMemoryError`?** Technically you can catch it, but the JVM state is unpredictable. Best to configure `-XX:+HeapDumpOnOutOfMemoryError` and restart.
2. **What is the default stack size per thread?** Depends on the OS and JVM, typically 512 KB to 1 MB.
3. **Does tail-call optimization prevent `StackOverflowError` in Java?** No. The JVM does not implement TCO. Use iteration instead.

---

### What is a ClassLoader? Name the three built-in ClassLoaders.

**Core Explanation**

A ClassLoader is a subsystem of the JVM responsible for loading `.class` files (bytecode) into memory at runtime. It follows the **delegation model**: before loading a class, a classloader delegates to its parent. Only if the parent fails does the child attempt to load the class itself.

**Three built-in ClassLoaders (Java 8):**

| ClassLoader | Loads From | Parent |
|-------------|-----------|--------|
| **Bootstrap ClassLoader** | `rt.jar`, core Java classes (`java.lang.*`, `java.util.*`) | None (native code) |
| **Extension (Platform) ClassLoader** | `jre/lib/ext` directory (Java 8) / platform modules (Java 9+) | Bootstrap |
| **Application (System) ClassLoader** | Classpath (`-cp`, `CLASSPATH`) | Extension / Platform |

In Java 9+ with JPMS, the Extension ClassLoader was renamed to **Platform ClassLoader**, and the Bootstrap ClassLoader loads modules from the module path.

**Practical Example**

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        // Application ClassLoader
        System.out.println(ClassLoaderDemo.class.getClassLoader());
        // sun.misc.Launcher$AppClassLoader (Java 8)
        // jdk.internal.loader.ClassLoaders$AppClassLoader (Java 9+)

        // Platform (Extension) ClassLoader
        System.out.println(ClassLoaderDemo.class.getClassLoader().getParent());

        // Bootstrap ClassLoader (returns null — implemented in native code)
        System.out.println(String.class.getClassLoader()); // null
    }
}
```

**Edge Cases / Pitfalls**
- Breaking the delegation model (e.g., Thread Context ClassLoader in JNDI/SPI) is intentional but can confuse developers. `Thread.currentThread().getContextClassLoader()` exists for this reason.
- Two classes loaded by *different* classloaders are considered different types by the JVM, even if the bytecode is identical. This causes unexpected `ClassCastException`.
- Custom classloaders that do not delegate properly can load the same class twice, creating subtle bugs.

**Best Practices**
Rely on the delegation model; only write custom classloaders when you have a clear need (plugin systems, hot-reloading). Always delegate to the parent first.

**Common Follow-Up Questions**
1. **What is the parent-delegation model?** A classloader delegates loading to its parent before attempting to load the class itself. This ensures core Java classes are always loaded by the Bootstrap ClassLoader.
2. **When would you write a custom ClassLoader?** Plugin architectures, application servers (isolating WARs), hot-reloading, loading classes from non-standard sources (database, network).
3. **What is `Thread.currentThread().getContextClassLoader()` for?** It lets frameworks (JNDI, SPI, JAXB) load classes using the caller's classloader rather than the framework's own classloader.

---

## Intermediate

---

### Explain JVM architecture: ClassLoader subsystem, runtime data areas, and execution engine.

**Core Explanation**

The JVM has three major subsystems:

```
┌─────────────────────────────────────────────────────────┐
│                     JVM Architecture                     │
├──────────────────┬──────────────────┬───────────────────┤
│  1. ClassLoader  │ 2. Runtime Data  │  3. Execution     │
│     Subsystem    │    Areas         │     Engine         │
├──────────────────┼──────────────────┼───────────────────┤
│ Loading          │ Method Area /    │ Interpreter        │
│ Linking:         │   Metaspace      │ JIT Compiler (C1,  │
│  - Verify        │ Heap             │   C2, Graal)       │
│  - Prepare       │ Stack (per thrd) │ Garbage Collector  │
│  - Resolve       │ PC Register      │                   │
│ Initialization   │ Native Method    │ JNI (Native       │
│                  │   Stack          │   Method Interface)│
│                  │ Code Cache       │                   │
└──────────────────┴──────────────────┴───────────────────┘
```

**1. ClassLoader Subsystem** performs three phases:
- **Loading** — reads `.class` bytecode (Bootstrap, Platform, Application classloaders).
- **Linking** — *Verification* (bytecode validity), *Preparation* (allocate static fields with defaults), *Resolution* (symbolic references to direct references).
- **Initialization** — execute `<clinit>` (static initializers and static blocks).

**2. Runtime Data Areas** (covered in detail in the memory areas question above).

**3. Execution Engine:**
- **Interpreter** — reads and executes bytecode line by line. Fast startup, slow sustained execution.
- **JIT Compiler** — compiles hot methods to native code. C1 (client, fast compilation) and C2 (server, aggressive optimization). Tiered compilation uses both.
- **Garbage Collector** — manages heap memory lifecycle.

**Practical Example**

```java
// What happens when you run: java com.example.App

// 1. ClassLoader Subsystem
//    - Bootstrap loads java.lang.Object, java.lang.String, etc.
//    - App ClassLoader loads com.example.App
//    - Linking: verify bytecode, prepare static fields, resolve references
//    - Initialization: run static blocks

// 2. Runtime Data Areas
//    - main thread stack is created
//    - App's class metadata stored in Metaspace
//    - Objects created during execution stored on Heap

// 3. Execution Engine
//    - Interpreter starts executing main() bytecode
//    - JIT monitors invocation counts
//    - After threshold, C1 compiles warm methods; C2 compiles hot methods
//    - GC reclaims unreachable objects periodically
```

**Edge Cases / Pitfalls**
- Class initialization (`<clinit>`) is guaranteed to be thread-safe by the JVM. This is why the *Initialization-on-Demand Holder* pattern works for lazy singletons.
- Resolution can be *lazy* — the JVM may not resolve symbolic references until first use, which is why `NoClassDefFoundError` can appear at runtime.

**Best Practices**
Understand the three subsystems conceptually for interviews; for production issues, focus on the Execution Engine (JIT compilation logs with `-XX:+PrintCompilation`) and GC behavior.

**Common Follow-Up Questions**
1. **What is tiered compilation?** The JVM uses 5 tiers (0=interpreter, 1-3=C1, 4=C2). Methods start interpreted and get promoted as they become hotter.
2. **When does the JVM deoptimize compiled code?** When an assumption made during compilation is invalidated (e.g., a class is loaded that changes an inheritance hierarchy, or an uncommon trap is hit).
3. **What is the difference between `<init>` and `<clinit>`?** `<init>` is the instance constructor; `<clinit>` is the class/static initializer.

---

### What is the difference between Metaspace and PermGen? Why was PermGen removed?

**Core Explanation**

| Aspect | PermGen (Java 7 and earlier) | Metaspace (Java 8+) |
|--------|------------------------------|----------------------|
| Location | Part of the Java heap (contiguous) | Native (off-heap) memory |
| Default size | Fixed (`-XX:MaxPermSize=256m` typical) | Unbounded (limited only by OS/physical memory) |
| GC | Collected during Full GC only | Collected when class metadata is unreachable |
| Resizing | Difficult, required JVM restart | Grows/shrinks dynamically |
| OOM message | `OutOfMemoryError: PermGen space` | `OutOfMemoryError: Metaspace` |

**Why was PermGen removed?**
1. **Fixed size was hard to tune** — too small caused OOM during deployments; too large wasted heap space.
2. **Fragmentation** — PermGen was subject to heap fragmentation, making Full GC expensive.
3. **Convergence with JRockit** — Oracle merged HotSpot and JRockit; JRockit never had PermGen.
4. **Dynamic class generation** — frameworks like Spring, Hibernate, and Groovy generate many classes dynamically, constantly stressing PermGen limits.

**Practical Example**

```java
// Java 7: frequent PermGen OOM with heavy CGLIB/Spring usage
// -XX:MaxPermSize=512m was a common "fix"

// Java 8+: Metaspace grows automatically
// But you should still set a cap to detect leaks early:
// -XX:MaxMetaspaceSize=512m
// -XX:MetaspaceSize=256m  (initial high-water mark before GC triggers)
```

**Edge Cases / Pitfalls**
- Unbounded Metaspace growth can silently consume all native memory if class metadata leaks (e.g., classloader leaks in application servers).
- The String Pool was moved from PermGen to the Heap in Java 7 (not Java 8). This is a common interview confusion point.
- Compressed class space (`-XX:CompressedClassSpaceSize`) is a subset of Metaspace for class pointers when compressed oops are enabled.

**Best Practices**
Always set `-XX:MaxMetaspaceSize` in production to fail fast on classloader leaks rather than letting the JVM consume all native memory.

**Common Follow-Up Questions**
1. **When was the String Pool moved?** Java 7 (from PermGen to Heap), not Java 8.
2. **What is Compressed Class Space?** A fixed-size region within Metaspace used for class pointers when `-XX:+UseCompressedOops` is enabled (default). Capped by `-XX:CompressedClassSpaceSize` (default 1 GB).
3. **How do you monitor Metaspace usage?** `jstat -gc <pid>`, JVisualVM, JFR, or GC logs with `-Xlog:gc*`.

---

### What is the difference between Minor GC (Young Gen) and Major/Full GC (Old Gen)?

**Core Explanation**

The heap is divided into generations based on the **generational hypothesis**: most objects die young.

```
Heap
├── Young Generation
│   ├── Eden        (new objects allocated here)
│   ├── Survivor 0  (S0)
│   └── Survivor 1  (S1)
└── Old Generation  (long-lived objects promoted here)
```

| Aspect | Minor GC | Major / Full GC |
|--------|----------|-----------------|
| Scope | Young Generation only | Old Generation (Major) or entire heap + Metaspace (Full) |
| Trigger | Eden space is full | Old Gen is full, or explicit `System.gc()`, or promotion failure |
| Duration | Fast (milliseconds) | Slow (hundreds of ms to seconds) |
| Frequency | Frequent | Infrequent |
| Algorithm | Copy collection (Eden + one Survivor -> other Survivor or Old) | Mark-Sweep-Compact (varies by collector) |

**Object lifecycle:**
1. New object allocated in Eden.
2. Minor GC: surviving objects copied to Survivor space, age incremented.
3. After reaching tenuring threshold (default 15), object promoted to Old Gen.
4. Old Gen collected during Major/Full GC.

**Practical Example**

```java
// GC log snippet (Unified Logging, Java 9+):
// -Xlog:gc*:file=gc.log:time,uptime,level,tags

// [0.234s] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
//   Eden: 24M -> 0M  Survivors: 0M -> 3M  Old: 0M -> 18M
//   Pause: 8.5ms     <-- Minor GC: fast

// [45.678s] GC(42) Pause Full (System.gc())
//   Eden: 0M -> 0M  Survivors: 0M -> 0M  Old: 480M -> 210M
//   Pause: 1450ms    <-- Full GC: slow, stop-the-world
```

**Edge Cases / Pitfalls**
- **Premature promotion**: if Survivor spaces are too small, objects are promoted to Old Gen before they die, increasing Full GC frequency. Tune with `-XX:SurvivorRatio` and `-XX:MaxTenuringThreshold`.
- A Minor GC can still cause a **promotion failure** if Old Gen does not have enough space, which triggers a Full GC.
- G1 GC blurs the line between Minor and Major GC with its region-based approach (mixed collections).

**Best Practices**
Aim for most objects to die in Young Gen (short-lived allocations). Monitor promotion rates; if Old Gen grows rapidly, investigate object lifetimes with heap profiling.

**Common Follow-Up Questions**
1. **What is a "mixed" GC in G1?** G1 collects Young regions plus some Old regions with the most garbage (highest reclamation efficiency).
2. **What is the default tenuring threshold?** 15 for most collectors. The JVM dynamically adjusts it based on Survivor space occupancy.
3. **Is Minor GC always stop-the-world?** Yes, but the pause is typically very short (single-digit milliseconds for reasonably sized Young Gens).

---

### What are the different garbage collectors (Serial, Parallel, G1, ZGC)? When would you choose each?

**Core Explanation**

| Collector | Flag | Algorithm | Threads | Pause Goal | Best For |
|-----------|------|-----------|---------|------------|----------|
| **Serial** | `-XX:+UseSerialGC` | Mark-Copy (Young), Mark-Compact (Old) | Single | None (simple) | Small heaps (<100 MB), containers with 1 CPU, client apps |
| **Parallel** (Throughput) | `-XX:+UseParallelGC` | Same as Serial but multi-threaded | Multi | Max throughput | Batch processing, high throughput, heap <4 GB |
| **G1** | `-XX:+UseG1GC` | Region-based, incremental compaction | Multi | `-XX:MaxGCPauseMillis=200` (default) | General purpose, heap 4-16 GB, balanced latency/throughput |
| **ZGC** | `-XX:+UseZGC` | Colored pointers, load barriers, concurrent | Multi | <1 ms (sub-millisecond) | Large heaps (multi-TB), ultra-low latency |
| **Shenandoah** | `-XX:+UseShenandoahGC` | Brooks pointers, concurrent compaction | Multi | <10 ms | Low latency, alternative to ZGC (Red Hat) |

**Defaults:**
- Java 8: Parallel GC
- Java 9+: G1 GC

**Practical Example**

```java
// High-throughput batch job (maximize CPU on GC work)
// java -XX:+UseParallelGC -XX:ParallelGCThreads=8 -Xmx4g BatchJob

// Web application (balanced latency)
// java -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -Xmx8g WebApp

// Trading system (ultra-low latency)
// java -XX:+UseZGC -Xmx32g -XX:+ZGenerational TradingEngine

// Tiny microservice in a 256MB container
// java -XX:+UseSerialGC -Xmx128m -Xss256k MicroService
```

**Edge Cases / Pitfalls**
- G1's pause-time goal is a *target*, not a guarantee. Very large heaps with many humongous objects can still cause long pauses.
- ZGC (Generational, Java 21+) significantly improves over non-generational ZGC. Always use `-XX:+ZGenerational` (default in Java 23+).
- Parallel GC maximizes throughput but can cause multi-second pauses on large heaps.

**Best Practices**
Start with G1 for general-purpose applications. Use ZGC when sub-millisecond pauses are required. Profile before switching collectors; measure, do not guess.

**Common Follow-Up Questions**
1. **What is the difference between G1 and ZGC?** G1 does incremental compaction with STW pauses (target ~200ms). ZGC does concurrent compaction with pauses <1ms regardless of heap size.
2. **What are humongous objects in G1?** Objects larger than half a G1 region. They are allocated directly in the Old Gen and can cause fragmentation.
3. **Can you mix GC options?** No. You pick one collector. The JVM will error if conflicting GC flags are set.

---

### What causes `OutOfMemoryError` even when heap size appears sufficient?

**Core Explanation**

Several scenarios cause OOM even when heap usage looks moderate:

1. **Native memory exhaustion** — the OS is out of memory for thread stacks, direct buffers, or JNI allocations.
2. **Metaspace exhaustion** — too many dynamically generated classes.
3. **GC overhead limit** — `OutOfMemoryError: GC overhead limit exceeded` when 98%+ of time is spent in GC recovering <2% heap.
4. **Fragmentation** — enough total free space exists but no single contiguous block for a large allocation (especially with CMS, less with G1/ZGC).
5. **Direct ByteBuffer exhaustion** — `OutOfMemoryError: Direct buffer memory`.
6. **Too many threads** — `OutOfMemoryError: unable to create new native thread` (each thread reserves ~1 MB of native stack).
7. **Humongous allocations in G1** — objects >50% of a region fail to find contiguous regions.
8. **Memory-mapped files** — extensive use of `MappedByteBuffer` consuming address space.

**Practical Example**

```java
// GC overhead limit exceeded
// Heap is 2 GB but almost full; GC runs constantly, reclaims almost nothing
List<Object> leak = new ArrayList<>();
while (true) {
    leak.add(new byte[100]); // slow leak; GC runs but can't free anything
}

// Native thread exhaustion
// Each thread uses ~1MB native stack
List<Thread> threads = new ArrayList<>();
while (true) {
    Thread t = new Thread(() -> {
        try { Thread.sleep(Long.MAX_VALUE); } catch (InterruptedException e) {}
    });
    t.start();
    threads.add(t);
    // Eventually: OutOfMemoryError: unable to create new native thread
}
```

**Edge Cases / Pitfalls**
- Container memory limits (cgroup) are separate from `-Xmx`. If heap is 4 GB but the container limit is 4.5 GB, native memory (thread stacks, Metaspace, code cache, direct memory) can push total usage over the container limit, causing an OOM kill (not a Java OOM).
- `-XX:+HeapDumpOnOutOfMemoryError` does NOT capture native memory or direct buffer issues.

**Best Practices**
Set `-Xmx` to ~75% of container memory, leaving room for non-heap usage. Always configure `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof` for post-mortem analysis.

**Common Follow-Up Questions**
1. **How do you diagnose native memory issues?** Use `-XX:NativeMemoryTracking=summary` and `jcmd <pid> VM.native_memory summary`.
2. **What is the GC overhead limit?** If the JVM spends >98% of time in GC and recovers <2% of heap, it throws OOM rather than thrashing indefinitely. Disable (not recommended) with `-XX:-UseGCOverheadLimit`.
3. **How does container memory differ from JVM heap?** Total JVM memory = Heap + Metaspace + Code Cache + Thread Stacks + Direct Memory + Native Allocations. All of this must fit within the container limit.

---

### What causes memory leaks in Java despite garbage collection? Give three real examples.

**Core Explanation**

A memory leak in Java occurs when objects are **reachable** (via GC roots) but **no longer needed** by the application. The GC cannot collect them because it can only reclaim unreachable objects. It has no concept of "needed" vs "not needed."

**Three Real Examples:**

**1. Listener/Callback Registration Without Deregistration**

```java
public class EventBus {
    private final List<EventListener> listeners = new ArrayList<>();

    public void register(EventListener listener) {
        listeners.add(listener);
    }
    // No unregister() called — listeners accumulate forever
}

// Every time a short-lived component registers, it stays referenced:
EventBus bus = getSingletonBus();
for (int i = 0; i < 1000; i++) {
    MyHandler handler = new MyHandler(); // Should be short-lived
    bus.register(handler);               // Now permanently referenced by the bus
    // handler is never deregistered
}
```

**2. Static Collections Growing Without Bound**

```java
public class CacheManager {
    // Static map — lives for the entire JVM lifetime
    private static final Map<String, byte[]> cache = new HashMap<>();

    public static void put(String key, byte[] data) {
        cache.put(key, data); // Entries never evicted
    }
    // Cache grows indefinitely — classic memory leak
}

// Fix: Use bounded caches (Caffeine, Guava) or WeakHashMap for metadata caches
```

**3. Unclosed Resources Holding Native Memory**

```java
public void readFiles(List<Path> paths) throws IOException {
    for (Path p : paths) {
        InputStream is = new FileInputStream(p.toFile());
        // process(is);
        // is.close() never called — file descriptors and buffers leak
        // Finalizer may eventually close, but it's unreliable and delayed
    }
}

// Fix: always use try-with-resources
public void readFilesSafe(List<Path> paths) throws IOException {
    for (Path p : paths) {
        try (InputStream is = Files.newInputStream(p)) {
            // process(is);
        } // auto-closed
    }
}
```

**Additional common leak patterns:**
- `ThreadLocal` not removed after use in thread pools (the thread stays alive, so the value is never collected).
- Inner classes holding implicit references to the outer class.
- Interned strings (`String.intern()`) accumulating in the String Pool.
- Custom classloader leaks in application servers (a single class reference pins the entire classloader and all its loaded classes).

**Edge Cases / Pitfalls**
- `WeakHashMap` keys are weakly referenced, but if the *value* strongly references the key, the entry is never collected.
- `ThreadLocal` leaks are insidious in servlet containers / thread pools because threads are reused, never terminated.

**Best Practices**
Use bounded caches (Caffeine with `maximumSize`), always close resources with try-with-resources, and call `ThreadLocal.remove()` in a `finally` block when using thread pools.

**Common Follow-Up Questions**
1. **How do you detect a memory leak?** Monitor heap usage over time (sawtooth pattern that trends upward). Take heap dumps and analyze with Eclipse MAT for dominator trees and leak suspects.
2. **What is a `ThreadLocal` leak?** In a thread pool, threads survive across requests. A `ThreadLocal` value set by request A persists to request B if not removed, and the object graph it references cannot be collected.
3. **What is the difference between a memory leak and high memory usage?** A leak means usage grows unboundedly over time. High usage can be legitimate (large caches, big datasets) but stable.

---

### How do you identify a memory leak in a running application (heap dump, MAT, VisualVM)?

**Core Explanation**

**Step-by-step process:**

1. **Detect symptoms**: monitor heap usage over time; a sawtooth pattern with an upward trend (baseline keeps growing) indicates a leak.
2. **Capture a heap dump**:
   - `jmap -dump:live,format=b,file=heap.hprof <pid>`
   - `jcmd <pid> GC.heap_dump heap.hprof`
   - Automatically on OOM: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/`
3. **Analyze with Eclipse MAT (Memory Analyzer Tool)**:
   - Open the `.hprof` file.
   - Run "Leak Suspects Report" for automated analysis.
   - Examine the **Dominator Tree** to find objects retaining the most memory.
   - Use **Histogram** to find classes with the most instances.
   - Trace **paths to GC roots** to understand *why* objects are retained.
4. **Compare heap dumps**: take two dumps at different times; compare histograms to find growing classes.

**Practical Example**

```bash
# Step 1: Monitor heap usage over time
jstat -gcutil <pid> 1000  # prints GC stats every second

# Step 2: Capture heap dump
jcmd <pid> GC.heap_dump /tmp/heap_before.hprof
# Wait, let the application run...
jcmd <pid> GC.heap_dump /tmp/heap_after.hprof

# Step 3: Analyze with MAT
# Open heap_after.hprof in Eclipse MAT
# Leak Suspects Report -> Suspect 1: 450MB retained by HashMap in CacheManager

# Step 4: Use VisualVM for live monitoring
# jvisualvm -> Monitor tab -> Heap chart
# Sampler tab -> Memory -> track allocations
```

```java
// JMX programmatic monitoring
MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
MemoryUsage heapUsage = memBean.getHeapMemoryUsage();
System.out.printf("Heap used: %d MB / %d MB%n",
    heapUsage.getUsed() / (1024 * 1024),
    heapUsage.getMax() / (1024 * 1024));
```

**Edge Cases / Pitfalls**
- Taking a heap dump causes a **stop-the-world pause** (can be seconds to minutes for large heaps). Do this during low-traffic windows or on a canary instance.
- `jmap -dump:live` forces a Full GC before dumping. This is usually what you want (to exclude garbage), but it adds to the pause.
- MAT's "shallow size" vs "retained size" distinction is critical. A small `HashMap` object can retain gigabytes via its entries.

**Best Practices**
Always configure `-XX:+HeapDumpOnOutOfMemoryError` in production. Regularly monitor heap trends with JFR or Prometheus + Micrometer, not just after incidents.

**Common Follow-Up Questions**
1. **What is a dominator tree in MAT?** It shows which objects, if garbage collected, would free the most memory. The dominator of an object is the last common gatekeeper on all paths from GC roots.
2. **Can you take a heap dump without pausing the application?** Not with standard tools. However, ZGC and Shenandoah minimize pause impact. Some APM tools (e.g., async-profiler) can do allocation profiling with minimal overhead.
3. **What is the difference between live and full heap dump?** `live` forces a GC first and includes only reachable objects. Full dump includes garbage too (useful for debugging finalizer issues).

---

### What are strong, weak, soft, and phantom references? Give a real use case for each.

**Core Explanation**

Java provides four reference strengths via `java.lang.ref`:

| Reference Type | GC Behavior | Collected When | Use Case |
|----------------|-------------|----------------|----------|
| **Strong** | Never collected while reachable | Only when unreachable from any GC root | Default; normal variables |
| **Soft** (`SoftReference`) | Collected before OOM | When memory pressure is high | Memory-sensitive caches |
| **Weak** (`WeakReference`) | Collected at next GC | As soon as no strong references exist | Metadata maps, canonicalizing maps |
| **Phantom** (`PhantomReference`) | Enqueued after finalization | After object is finalized but before memory is reclaimed | Resource cleanup, leak detection |

**Reachability order**: Strong > Soft > Weak > Phantom > Unreachable

**Practical Example**

```java
// 1. Strong reference — default
Object obj = new Object(); // Strong reference; GC will never collect while 'obj' is in scope

// 2. SoftReference — memory-sensitive cache
Map<String, SoftReference<byte[]>> imageCache = new HashMap<>();
public byte[] getImage(String key) {
    SoftReference<byte[]> ref = imageCache.get(key);
    byte[] data = (ref != null) ? ref.get() : null;
    if (data == null) {
        data = loadFromDisk(key);
        imageCache.put(key, new SoftReference<>(data));
    }
    return data;
}
// JVM keeps cached images as long as memory permits; evicts under pressure.

// 3. WeakReference — metadata/association map (WeakHashMap)
// Associates extra data with objects without preventing their collection
WeakHashMap<Thread, TransactionContext> threadContexts = new WeakHashMap<>();
// When a Thread is no longer strongly referenced, its entry is auto-removed.

// 4. PhantomReference — resource cleanup
ReferenceQueue<LargeNativeResource> queue = new ReferenceQueue<>();
PhantomReference<LargeNativeResource> phantom =
    new PhantomReference<>(resource, queue);

// Cleanup thread:
// while (true) {
//     Reference<?> ref = queue.remove(); // blocks until ref is enqueued
//     freeNativeMemory(ref);             // custom cleanup
// }
// java.lang.ref.Cleaner (Java 9+) encapsulates this pattern.
```

**Edge Cases / Pitfalls**
- `SoftReference` behavior is JVM-implementation-specific. HotSpot uses a formula based on free heap and time since last access. Do not rely on precise eviction behavior; prefer Caffeine/Guava caches with explicit size bounds.
- `PhantomReference.get()` always returns `null`. You cannot resurrect the object. This is by design — it only signals that cleanup should happen.
- `WeakHashMap` only weakly references *keys*. If the value strongly references the key, the entry is never collected.

**Best Practices**
Use `SoftReference` sparingly; prefer bounded caches (Caffeine). Use `Cleaner` (Java 9+) instead of raw `PhantomReference` for cleanup. `WeakReference` is useful in frameworks for observer patterns.

**Common Follow-Up Questions**
1. **Where is `WeakReference` used in the JDK?** `WeakHashMap`, `ThreadLocal` (each thread's `ThreadLocalMap` uses `WeakReference` to the `ThreadLocal` key).
2. **What is a `ReferenceQueue`?** A queue into which the GC enqueues `Reference` objects after the referent is collected. Allows cleanup code to react to collection events.
3. **Why not use `finalize()` instead of `PhantomReference`?** `finalize()` is deprecated, unpredictable, causes GC overhead, and can resurrect objects. `Cleaner`/`PhantomReference` is safer and more reliable.

---

### What is JIT compilation? How does the JVM decide which methods to compile?

**Core Explanation**

**JIT (Just-In-Time) compilation** converts frequently executed bytecode into optimized native machine code at runtime, rather than interpreting it instruction by instruction.

The JVM uses **tiered compilation** (default since Java 8):

| Tier | Compiler | Profile Data | Optimization Level |
|------|----------|-------------|-------------------|
| 0 | Interpreter | Full profiling | None |
| 1 | C1 | No profiling | Simple |
| 2 | C1 | Limited profiling | Simple |
| 3 | C1 | Full profiling | Simple + profiling |
| 4 | C2 (Server) | Uses Tier 3 profile | Aggressive (inlining, escape analysis, loop unrolling, vectorization) |

**How the JVM decides what to compile:**
1. **Invocation counters**: each method has a counter incremented on each call.
2. **Back-edge counters**: incremented on loop-back jumps (loop iterations).
3. When counters exceed thresholds, the method is queued for compilation.
4. Default threshold: ~10,000 invocations/iterations for C2 (`-XX:CompileThreshold`).

**Practical Example**

```java
// This loop body will be JIT-compiled after ~10,000 iterations
public long sum(int[] arr) {
    long total = 0;
    for (int val : arr) {   // Back-edge counter increments each iteration
        total += val;
    }
    return total;
    // After enough calls, C2 compiles this to SIMD vectorized native code
}
```

```bash
# Observe JIT compilation
java -XX:+PrintCompilation MyApp
#   Columns: timestamp compile_id tier method_name size deopt
#   112    1     3       java.lang.String::hashCode (55 bytes)
#   115    2     4       java.lang.String::hashCode (55 bytes)  <-- promoted to C2

# Detailed compilation log
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly MyApp
```

**Edge Cases / Pitfalls**
- **Deoptimization**: if a C2 assumption is invalidated (e.g., class hierarchy change, uncommon branch taken), the JVM deoptimizes back to the interpreter. This causes a temporary performance dip.
- **Code cache exhaustion**: compiled code lives in the Code Cache. If it fills up, the JVM stops compiling. Monitor with `-XX:ReservedCodeCacheSize`.
- Short-lived applications never benefit from JIT because methods never reach the compilation threshold. Consider AOT compilation (GraalVM Native Image) for such cases.

**Best Practices**
Use tiered compilation (the default). For latency-sensitive applications, consider a warm-up phase (pre-exercise hot paths) before accepting traffic.

**Common Follow-Up Questions**
1. **What is On-Stack Replacement (OSR)?** Compiling a method while it is still executing (mid-loop). Allows long-running loops to benefit from JIT without waiting for the method to be called again.
2. **What is the difference between C1 and C2?** C1 compiles quickly with modest optimizations (fast startup). C2 compiles slowly with aggressive optimizations (peak throughput).
3. **What is GraalVM's JIT compiler?** An alternative to C2, written in Java, that can serve as a drop-in JIT compiler with different optimization strategies.

---

### What is the difference between `ClassNotFoundException` and `NoClassDefFoundError`?

**Core Explanation**

| Aspect | `ClassNotFoundException` | `NoClassDefFoundError` |
|--------|-------------------------|------------------------|
| Type | Checked `Exception` | `Error` (unchecked) |
| Cause | Explicit class loading failed (`Class.forName()`, `ClassLoader.loadClass()`) | Class was available at compile time but missing at runtime |
| When | Runtime, during reflective loading | Runtime, during first active use of the class |
| Typical Fix | Add missing JAR to classpath | Fix classpath or failed static initializer |

**Practical Example**

```java
// ClassNotFoundException — reflective loading of a missing class
try {
    Class<?> clazz = Class.forName("com.mysql.cj.jdbc.Driver");
} catch (ClassNotFoundException e) {
    // mysql-connector-java JAR is not on the classpath
    System.err.println("Driver not found: " + e.getMessage());
}

// NoClassDefFoundError — class existed at compile time, missing at runtime
// Compile: javac -cp lib/utils.jar Main.java   (compiles fine)
// Run:     java Main                            (utils.jar not on classpath)
// Result:  NoClassDefFoundError: com/example/Utils

// Another common cause: static initializer failure
class Config {
    static {
        // If this throws, Config is marked as unusable
        String val = System.getenv("REQUIRED_VAR");
        if (val == null) throw new RuntimeException("Missing env var");
    }
}

// First access: ExceptionInInitializerError
// Second access: NoClassDefFoundError (class initialization already failed)
```

**Edge Cases / Pitfalls**
- `NoClassDefFoundError` wrapping `ExceptionInInitializerError`: if a class's `<clinit>` (static initializer) throws an exception, the first access throws `ExceptionInInitializerError`. All subsequent accesses throw `NoClassDefFoundError` because the class is permanently marked as failed.
- In application servers, `NoClassDefFoundError` often indicates a classloader visibility issue: the class exists in one WAR/EAR but not in the classloader hierarchy of the requesting code.

**Best Practices**
When you see `NoClassDefFoundError`, check: (1) is the class/JAR on the runtime classpath, (2) did the static initializer fail, (3) are there classloader isolation issues.

**Common Follow-Up Questions**
1. **Can `ClassNotFoundException` cause `NoClassDefFoundError`?** Yes. If the JVM internally calls `ClassLoader.loadClass()` and it fails, this can surface as `NoClassDefFoundError`.
2. **Which is recoverable?** `ClassNotFoundException` is a checked exception, so it is expected to be handled. `NoClassDefFoundError` is an `Error` and generally indicates a deployment issue.
3. **How does the module system (JPMS) affect this?** A class might exist on the module path but not be exported/opened to the requesting module, causing `IllegalAccessError` instead.

---

### What happens internally when you create a Java object (`new MyClass()`)?

**Core Explanation**

The sequence when executing `new MyClass()`:

1. **Class loading check**: JVM checks if `MyClass` is already loaded. If not, the ClassLoader subsystem loads, links, and initializes it.
2. **Memory allocation**: JVM allocates memory for the object on the heap.
   - **Bump-the-pointer**: if heap memory is contiguous (compacted), the JVM simply moves a pointer forward. Very fast.
   - **Free list**: if memory is fragmented, the JVM searches a free list for a suitable block.
   - **TLAB (Thread-Local Allocation Buffer)**: each thread has a pre-allocated chunk of Eden. Allocation is pointer-bump within the TLAB — no synchronization needed.
3. **Zero initialization**: all instance fields are set to their default values (`0`, `null`, `false`).
4. **Object header setup**: the JVM writes the object header:
   - **Mark word**: hash code, GC age, lock state, biased locking info.
   - **Class pointer (Klass pointer)**: reference to the class metadata in Metaspace.
   - **Array length** (if array).
5. **Constructor execution (`<init>`)**: the instance initializer runs — field initializers, instance initializer blocks, and the constructor body, following the chain up to `Object.<init>`.

**Practical Example**

```java
public class Employee {
    private String name;
    private int age = 25; // instance initializer

    public Employee(String name) {
        this.name = name;
    }
}

// new Employee("Alice") internally:
// 1. Check if Employee.class is loaded (yes, assume it is)
// 2. Allocate ~32 bytes on heap (header + fields, with alignment padding)
//    - TLAB bump-pointer: threadLocalPtr += 32
// 3. Zero memory: name = null, age = 0
// 4. Write object header:
//    - Mark word: identity hashcode=0 (lazy), age=0, lock=unlocked
//    - Klass pointer: -> Employee class metadata in Metaspace
// 5. Execute <init>:
//    - Call super() -> Object.<init>()
//    - age = 25 (field initializer)
//    - name = "Alice" (constructor body)
// 6. Return reference to the new object
```

```java
// Bytecode (javap -c Employee):
//   0: aload_0
//   1: invokespecial #1  // Object.<init>
//   4: aload_0
//   5: bipush 25
//   7: putfield #2       // age = 25
//  10: aload_0
//  11: aload_1
//  12: putfield #3       // name = argument
//  15: return
```

**Edge Cases / Pitfalls**
- **TLAB exhaustion**: when a thread's TLAB is full, it requests a new one from Eden. This is a fast-path/slow-path design.
- **Escape analysis** (JIT): if the JVM proves the object does not escape the method, it may allocate fields on the stack (scalar replacement) or eliminate the allocation entirely.
- The `new` bytecode and `invokespecial <init>` are separate operations. Between them, the object is allocated but uninitialized. This matters in bytecode manipulation and concurrent visibility.

**Best Practices**
Trust the JVM's allocation fast path (TLAB + bump pointer); it is extremely fast (~10 ns). Avoid premature optimization around object creation unless profiling shows allocation pressure.

**Common Follow-Up Questions**
1. **What is a TLAB?** Thread-Local Allocation Buffer — a thread-private region of Eden for lock-free allocation. Default size is dynamically tuned by the JVM.
2. **What is the object header size?** 12 bytes with compressed oops (8-byte mark word + 4-byte klass pointer), 16 bytes without. Arrays add 4 bytes for length.
3. **Can the constructor see partially constructed state?** Yes, if `this` reference escapes during construction (e.g., registering a listener in the constructor). Other threads may see default values for fields not yet assigned.

---

## Advanced

---

### What is escape analysis? How does the JVM use it to allocate objects on the stack?

**Core Explanation**

**Escape analysis** is a JIT compiler optimization (C2) that determines whether an object's reference escapes the scope of the method or thread:

| Escape Level | Description | Optimization |
|--------------|-------------|-------------|
| **NoEscape** | Object is used only within the method | Scalar replacement (stack allocation), lock elimination |
| **ArgEscape** | Object is passed to a method but does not escape the thread | Lock elimination possible |
| **GlobalEscape** | Object is stored in a static field, returned, or published to another thread | No optimization; heap allocation |

**Key optimizations enabled by escape analysis:**
1. **Scalar replacement**: the object is decomposed into its fields, which are placed in CPU registers or the thread stack. The object itself is never actually allocated on the heap.
2. **Lock elimination**: `synchronized` on a non-escaping object is removed.
3. **Allocation elimination**: if the object's fields are never read after the method, the allocation may be removed entirely.

**Practical Example**

```java
public int sumCoordinates(int x, int y) {
    Point p = new Point(x, y); // Does 'p' escape?
    return p.getX() + p.getY();
}

// Escape analysis determines: p is NoEscape (never stored, returned, or shared)
// JIT applies scalar replacement:
// Optimized to (conceptually):
public int sumCoordinates(int x, int y) {
    int p_x = x;  // 'fields' placed in registers
    int p_y = y;
    return p_x + p_y;
}
// No heap allocation occurs. No GC pressure. Zero-cost abstraction.
```

```java
// This PREVENTS escape analysis:
public Point createPoint(int x, int y) {
    Point p = new Point(x, y);
    return p;  // p escapes the method -> must be heap-allocated
}

// This also prevents it:
public void process(int x, int y) {
    Point p = new Point(x, y);
    externalList.add(p);  // p escapes to a shared data structure
}
```

**Edge Cases / Pitfalls**
- Escape analysis only works on **C2-compiled** methods (Tier 4). Interpreted code and C1-compiled code always allocate on the heap.
- Megamorphic call sites (method called on many different types) can prevent inlining, which in turn prevents escape analysis from proving NoEscape.
- The JVM does not literally "allocate on the stack." It performs scalar replacement — the object is never created; its fields are promoted to local variables/registers.
- `-XX:+DoEscapeAnalysis` is enabled by default. Disabling it (`-XX:-DoEscapeAnalysis`) is useful for benchmarking.

**Best Practices**
Write clean, idiomatic code with small, short-lived objects. Trust the JIT to eliminate allocations via escape analysis. Verify with `-XX:+PrintEscapeAnalysis` or JMH benchmarks.

**Common Follow-Up Questions**
1. **Does Java truly do stack allocation?** Not in the traditional sense. It performs scalar replacement, which is even better — fields go directly into registers. The object metadata (header, type info) is eliminated entirely.
2. **Does escape analysis work with `synchronized`?** Yes. If the lock object is NoEscape, the `synchronized` block is removed (lock elision).
3. **How do you verify escape analysis is working?** Use `-XX:+PrintEscapeAnalysis` (diagnostic), or benchmark with JMH comparing allocation rates with `-XX:+DoEscapeAnalysis` vs `-XX:-DoEscapeAnalysis`.

---

### What are safepoints? How do they cause stop-the-world pauses?

**Core Explanation**

A **safepoint** is a point in execution where the JVM can safely examine and modify a thread's state. At a safepoint, the thread's stack, registers, and object references are in a known, consistent state.

**When the JVM needs a safepoint (stop-the-world):**
1. Garbage collection (most common).
2. Deoptimization (invalidating compiled code).
3. Biased lock revocation (pre-Java 15).
4. Thread dump (`jstack`).
5. Class redefinition (JVMTI, hot-swap).
6. Heap dump.

**How STW happens:**
1. The JVM sets a global "safepoint requested" flag.
2. Each thread checks this flag at safepoint polls (loop back-edges, method returns, allocation sites).
3. Threads that are executing compiled code reach a safepoint poll and block.
4. Threads in interpreted code check at each bytecode boundary.
5. Threads blocked on I/O or monitors are already at a safepoint.
6. Once **all** threads have reached a safepoint, the JVM performs its operation.
7. Time-to-safepoint (TTSP) is the time waiting for the *slowest* thread.

**Practical Example**

```java
// Counted loop — JIT may NOT place safepoint polls inside:
public void countedLoop() {
    long sum = 0;
    for (int i = 0; i < 1_000_000_000; i++) { // Counted loop
        sum += i;
        // C2 may optimize away safepoint poll here for performance
        // This delays the entire JVM from reaching a safepoint!
    }
}

// Fix: use a long loop variable (uncounted loop has safepoint polls):
public void uncountedLoop() {
    long sum = 0;
    for (long i = 0; i < 1_000_000_000L; i++) { // 'long' makes it uncounted
        sum += i;
        // Safepoint poll IS inserted here
    }
}
```

```bash
# Diagnose safepoint pauses:
# Java 9+:
java -Xlog:safepoint=info MyApp
# Look for: "Total time for which application threads were stopped"
# and "Spinning" / "Sync" / "Cleanup" times
```

**Edge Cases / Pitfalls**
- **Counted loops with `int` iterators** are the most notorious TTSP issue. C2 omits safepoint polls for int-counted loops as an optimization. A billion-iteration counted loop can delay safepoint for seconds.
- JNI code does not poll for safepoints. A long-running JNI call delays STW until it returns. The JVM considers JNI threads as "in native" and can proceed if the thread does not access Java heap.
- **Thread.sleep(0)** and `Thread.yield()` are safepoint polls and can be inserted strategically to mitigate TTSP issues.
- Java 17+: `-XX:+UseCountedLoopSafepoints` (default on) mitigates the counted-loop issue by inserting polls at configurable intervals.

**Best Practices**
Monitor time-to-safepoint with `-Xlog:safepoint`. Avoid very large int-counted loops in latency-sensitive code. Use Java 17+ where counted-loop safepoints are enabled by default.

**Common Follow-Up Questions**
1. **What is the difference between "time at safepoint" and "time to safepoint"?** "Time at safepoint" is how long the STW operation took. "Time to safepoint" (TTSP) is how long the JVM waited for all threads to reach the safepoint.
2. **Do virtual threads (Project Loom) change safepoint behavior?** Virtual threads do not map 1:1 to OS threads, and they yield at blocking operations. This reduces the impact on carrier thread safepoint behavior.
3. **Can you have a STW pause without GC?** Yes — deoptimization, biased lock revocation, thread dumps, and class redefinition all require safepoints.

---

### What is false sharing? How does `@Contended` mitigate it?

**Core Explanation**

**False sharing** occurs when two threads write to different variables that reside on the same CPU cache line (typically 64 bytes). The CPU cache coherence protocol (MESI) invalidates the entire cache line on each write, causing both cores to constantly reload the line from main memory, destroying performance.

The threads are not sharing data, but they are sharing a cache line — hence "false" sharing.

**`@Contended`** (introduced in Java 8, `jdk.internal.misc.Contended` or `sun.misc.Contended`) adds padding around annotated fields to ensure they occupy separate cache lines.

**Practical Example**

```java
// FALSE SHARING: field1 and field2 are on the same cache line
class SharedData {
    volatile long field1; // Thread A writes this
    volatile long field2; // Thread B writes this
    // Both fields fit in one 64-byte cache line -> false sharing!
}

// FIXED: Manual padding
class PaddedData {
    volatile long field1;
    long p1, p2, p3, p4, p5, p6, p7; // 56 bytes of padding
    volatile long field2; // Now on a different cache line
}

// FIXED: Using @Contended (preferred)
// Requires: -XX:-RestrictContended (for user classes)
import jdk.internal.misc.Contended;

class ContendedData {
    @Contended
    volatile long field1;

    @Contended
    volatile long field2;
    // JVM automatically pads each field to separate cache lines (128 bytes default)
}
```

```java
// Real-world example: java.lang.Thread uses @Contended
// for its threadLocalRandomSeed to avoid false sharing between threads:
//
// @jdk.internal.misc.Contended("tlr")
// long threadLocalRandomSeed;
// @jdk.internal.misc.Contended("tlr")
// int threadLocalRandomProbe;

// LongAdder (java.util.concurrent.atomic) also uses @Contended
// internally in its Cell[] array to avoid false sharing between cells.
```

**Edge Cases / Pitfalls**
- `@Contended` padding defaults to 128 bytes (not 64) to account for hardware prefetching on some architectures. Configurable with `-XX:ContendedPaddingWidth`.
- For user-defined classes, you must add `-XX:-RestrictContended` or the annotation is silently ignored.
- False sharing is only a problem for **highly contended writes** across threads. Read-only data sharing across cache lines is fine.
- Array elements can also suffer from false sharing if different threads write to adjacent indices.

**Best Practices**
Use `@Contended` for fields that are written frequently by different threads (e.g., counters, RNG state). Prefer `LongAdder` over `AtomicLong` for high-contention counters — it uses `@Contended` cells internally.

**Common Follow-Up Questions**
1. **What is the typical cache line size?** 64 bytes on most x86/ARM64 processors. Some architectures use 128 bytes.
2. **Where is `@Contended` used in the JDK?** `Thread` (TLR fields), `Striped64` (parent of `LongAdder`/`LongAccumulator`), `ForkJoinPool`.
3. **How do you detect false sharing?** Use hardware performance counters (`perf stat` on Linux) to measure L1/L2 cache miss rates, or Intel VTune / `perf c2c` for cache-line contention analysis.

---

### What JVM flags have you used in production (`-Xms`, `-Xmx`, `-XX:+UseG1GC`, `-XX:MaxGCPauseMillis`)?

**Core Explanation**

Production JVM flags fall into categories:

**Heap Sizing:**

```bash
-Xms4g                        # Initial heap size (set equal to -Xmx to avoid resizing)
-Xmx4g                        # Maximum heap size
-XX:MaxMetaspaceSize=512m     # Cap Metaspace to detect leaks early
-XX:MaxDirectMemorySize=1g    # Cap direct (off-heap) ByteBuffer memory
```

**GC Selection and Tuning:**

```bash
-XX:+UseG1GC                   # G1 collector (default Java 9+)
-XX:MaxGCPauseMillis=200       # Target pause time (G1)
-XX:G1HeapRegionSize=16m       # Region size (1-32 MB, power of 2)
-XX:InitiatingHeapOccupancyPercent=45  # Start concurrent marking at 45% Old Gen
-XX:ParallelGCThreads=8        # GC threads for STW phases
-XX:ConcGCThreads=4            # GC threads for concurrent phases

# For ZGC (Java 17+):
-XX:+UseZGC -XX:+ZGenerational
```

**Diagnostics and Debugging:**

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heap.hprof
-XX:+ExitOnOutOfMemoryError      # Immediately exit; let orchestrator restart
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=100m
```

**Performance:**

```bash
-XX:+UseStringDeduplication      # G1 only; saves heap for String-heavy apps
-XX:+AlwaysPreTouch              # Touch all heap pages at startup (faster allocation, predictable RSS)
-XX:+UseCompressedOops           # Default for heaps <32 GB
-XX:ReservedCodeCacheSize=256m   # Increase if Code Cache fills up
```

**Practical Example — Production Startup Script:**

```bash
java \
  -Xms8g -Xmx8g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=150 \
  -XX:+AlwaysPreTouch \
  -XX:MaxMetaspaceSize=512m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/app/heap.hprof \
  -XX:+ExitOnOutOfMemoryError \
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=100m \
  -jar myapp.jar
```

**Edge Cases / Pitfalls**
- Setting `-Xms` != `-Xmx` causes heap resizing pauses. In production, set them equal.
- `-XX:+AlwaysPreTouch` increases startup time (touches every page) but prevents page faults during runtime.
- Container-aware JVM (Java 10+): the JVM reads cgroup limits automatically. Use `-XX:MaxRAMPercentage=75.0` instead of `-Xmx` for container-proportional sizing.

**Best Practices**
Set `-Xms` = `-Xmx`, always enable heap dump on OOM, use GC logging, and set `-XX:+ExitOnOutOfMemoryError` to let Kubernetes/systemd restart the process cleanly.

**Common Follow-Up Questions**
1. **What does `-XX:+AlwaysPreTouch` do?** Touches every heap page at JVM startup, forcing the OS to allocate physical memory. Avoids page faults later but increases startup time.
2. **What is `-XX:MaxRAMPercentage`?** Sets heap as a percentage of container memory. `75.0` means 75% of container RAM for heap, leaving 25% for non-heap (Metaspace, threads, direct memory).
3. **Should you use `-XX:+ExitOnOutOfMemoryError` or `-XX:+CrashOnOutOfMemoryError`?** `ExitOnOutOfMemoryError` calls `System.exit(1)` (graceful). `CrashOnOutOfMemoryError` calls `abort()` (generates core dump). Choose based on whether you need a core dump.

---

### How do you tune JVM for high-throughput vs low-latency applications? What are the trade-offs?

**Core Explanation**

| Aspect | High Throughput | Low Latency |
|--------|----------------|-------------|
| Goal | Maximize work done per unit time | Minimize worst-case pause time |
| GC choice | Parallel GC or G1 | ZGC or Shenandoah |
| Heap size | Large (maximize throughput before GC) | Large (reduce GC frequency) but with concurrent collection |
| GC threads | Max out STW parallelism | Concurrent GC threads, fewer STW threads |
| Young Gen | Larger (fewer Minor GCs) | Tuned to keep pauses short |
| Trade-off | Accepts long STW pauses | Accepts lower throughput (concurrent GC uses CPU) |

**Practical Example**

```bash
# HIGH THROUGHPUT (batch processing, ETL, MapReduce)
java \
  -XX:+UseParallelGC \
  -Xms16g -Xmx16g \
  -XX:ParallelGCThreads=16 \
  -XX:+UseAdaptiveSizePolicy \
  -XX:GCTimeRatio=99 \         # Spend <1% of time in GC
  -jar batch-job.jar

# LOW LATENCY (trading system, real-time API)
java \
  -XX:+UseZGC -XX:+ZGenerational \
  -Xms16g -Xmx16g \
  -XX:+AlwaysPreTouch \
  -XX:ConcGCThreads=4 \
  -jar trading-engine.jar

# BALANCED (web application)
java \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \
  -Xms8g -Xmx8g \
  -XX:+AlwaysPreTouch \
  -jar webapp.jar
```

**Key Trade-offs:**

```
Throughput <------> Latency <------> Footprint

Pick two. You cannot optimize all three simultaneously.

- Throughput-optimized: large heap, infrequent but long GC pauses
- Latency-optimized: concurrent GC, uses more CPU (lower throughput), more memory overhead
- Footprint-optimized: small heap, frequent short GCs, may sacrifice both throughput and latency
```

**Edge Cases / Pitfalls**
- ZGC trades ~15% throughput for sub-millisecond pauses. For batch jobs where latency does not matter, Parallel GC is still faster.
- Setting `-XX:MaxGCPauseMillis` too low in G1 causes it to collect smaller regions, reducing throughput and potentially causing evacuation failures.
- Over-allocating GC threads starves application threads. `ConcGCThreads` should be ~25% of available CPUs.

**Best Practices**
Start with G1 for general workloads. Switch to ZGC only if p99 latency is unacceptable. Measure before tuning — use GC logs and JFR to identify the bottleneck before changing flags.

**Common Follow-Up Questions**
1. **What is `GCTimeRatio`?** Sets the ratio of application time to GC time. `GCTimeRatio=99` means the JVM targets spending <1% of time in GC (1/(1+99)).
2. **Does ZGC have any downsides?** Slightly lower throughput due to load barriers, higher memory overhead (colored pointers), and it requires more concurrent CPU budget.
3. **What is allocation rate and promotion rate?** Allocation rate: how fast objects are created (MB/s in Eden). Promotion rate: how fast objects move to Old Gen. High promotion rate is the main enemy of low-latency GC.

---

### What JVM profiling tools have you used (JFR, JMC, jstack, jmap, async-profiler)?

**Core Explanation**

| Tool | Purpose | Overhead | Type |
|------|---------|----------|------|
| **JFR** (Java Flight Recorder) | Continuous event recording (CPU, GC, I/O, allocations, locks) | <2% (production-safe) | Event-based profiling |
| **JMC** (Java Mission Control) | GUI for analyzing JFR recordings | N/A (offline analysis) | Analysis tool |
| **jstack** | Thread dump (stack traces of all threads) | Safepoint pause | Diagnostic |
| **jmap** | Heap dump, histogram, class loader stats | STW pause (for dump) | Diagnostic |
| **jcmd** | Swiss-army knife; can do everything jstack/jmap does + more | Varies | Diagnostic |
| **async-profiler** | Low-overhead CPU/allocation/lock profiling | <5% | Sampling profiler |
| **VisualVM** | GUI for monitoring, thread analysis, heap profiling | Agent overhead | Monitoring |
| **jstat** | GC statistics (heap/gc counters) | Minimal | Monitoring |

**Practical Example**

```bash
# JFR — start recording on a running JVM
jcmd <pid> JFR.start name=profile duration=60s filename=/tmp/profile.jfr \
  settings=profile

# JFR — launch with recording enabled from the start
java -XX:StartFlightRecording=duration=300s,filename=app.jfr,settings=profile \
  -jar myapp.jar

# Open the .jfr file in JMC for analysis

# jstack — thread dump (find deadlocks, blocked threads)
jstack <pid> > /tmp/thread_dump.txt
# Or:
jcmd <pid> Thread.print > /tmp/thread_dump.txt

# jmap — heap dump
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# jmap — class histogram (without full dump)
jcmd <pid> GC.class_histogram | head -30

# jstat — monitor GC in real-time
jstat -gcutil <pid> 1000   # every 1 second

# async-profiler — CPU flame graph
./asprof -d 30 -f /tmp/flamegraph.html <pid>

# async-profiler — allocation profiling
./asprof -d 30 -e alloc -f /tmp/alloc.html <pid>
```

**When to use each:**

| Symptom | Tool |
|---------|------|
| Application seems slow, unknown cause | JFR + JMC (broad analysis) |
| Suspected CPU bottleneck | async-profiler (CPU flame graph) |
| Suspected memory leak | jmap (heap dump) + Eclipse MAT |
| Application hangs / deadlock | jstack or `jcmd Thread.print` |
| GC pauses too long | jstat + GC logs + JFR GC events |
| High allocation rate | async-profiler (alloc mode) or JFR allocation events |

**Edge Cases / Pitfalls**
- `jstack` and `jmap` require a safepoint, which adds to pause time. Use `jcmd` where possible.
- JFR is built into the JVM since Java 11 (free in OpenJDK). In Java 8, it was a commercial feature requiring `-XX:+UnlockCommercialFeatures`.
- async-profiler uses `perf_events` on Linux, not safepoints, so it has no safepoint bias (unlike JFR's execution sample event).

**Best Practices**
Always run JFR in production with low-overhead settings. Use async-profiler for targeted CPU/allocation profiling during performance investigations.

**Common Follow-Up Questions**
1. **What is safepoint bias?** JVM-based profilers sample only at safepoints, so they miss methods that run between safepoints (e.g., tight int-counted loops). async-profiler avoids this by using OS-level signal-based sampling.
2. **Is JFR free?** Yes, since Java 11 (OpenJDK). JMC is also open-source.
3. **How is async-profiler different from JFR?** async-profiler uses Linux `perf_events` for CPU sampling (no safepoint bias) and TLAB-based allocation tracking. JFR is broader (covers GC, I/O, locks, exceptions, etc.) but has safepoint bias for CPU samples.

---

### When does JVM allocate objects on the stack instead of the heap?

**Core Explanation**

The JVM allocates objects on the stack (via **scalar replacement**) when the C2 JIT compiler's **escape analysis** proves that an object does **not escape** the method:

1. The object is not returned from the method.
2. The object is not stored in a field (static or instance).
3. The object is not passed to an external method that might store it.
4. The method is JIT-compiled at Tier 4 (C2).

When all conditions are met, the JVM performs **scalar replacement**: it decomposes the object into its individual fields and places them in CPU registers or the stack frame. The object header and allocation are eliminated entirely.

**Practical Example**

```java
// Stack-allocated (scalar replacement):
public double distance(double x1, double y1, double x2, double y2) {
    Point p1 = new Point(x1, y1); // NoEscape -> scalar replacement
    Point p2 = new Point(x2, y2); // NoEscape -> scalar replacement
    double dx = p1.x - p2.x;
    double dy = p1.y - p2.y;
    return Math.sqrt(dx * dx + dy * dy);
    // No heap allocation; p1.x, p1.y, p2.x, p2.y become local variables
}

// Heap-allocated (escapes):
public Point midpoint(double x1, double y1, double x2, double y2) {
    Point mid = new Point((x1 + x2) / 2, (y1 + y2) / 2);
    return mid; // Escapes the method -> must be heap-allocated
}

// Heap-allocated (stored in field):
private List<Point> points = new ArrayList<>();
public void addPoint(double x, double y) {
    Point p = new Point(x, y);
    points.add(p); // Escapes to a field -> heap-allocated
}
```

**Edge Cases / Pitfalls**
- Escape analysis requires method inlining to see through call chains. If a callee is too large to inline (`-XX:MaxInlineSize=35` bytes default, `-XX:FreqInlineSize=325` for hot methods), the object is conservatively considered escaping.
- Polymorphic calls (multiple possible implementations) prevent inlining and thus prevent escape analysis.
- Only the C2 compiler does escape analysis. Code running in the interpreter or C1 always heap-allocates.
- Very large objects may not be scalar-replaced even if they don't escape, due to register pressure.

**Best Practices**
Write small, focused methods that the JIT can inline. Prefer final classes and monomorphic call sites to maximize inlining and escape analysis opportunities.

**Common Follow-Up Questions**
1. **Is there a JVM flag to control escape analysis?** `-XX:+DoEscapeAnalysis` (default: on). `-XX:+EliminateAllocations` (default: on) enables scalar replacement specifically.
2. **Does this work with arrays?** Small fixed-size arrays may be scalar-replaced. Large or dynamically-sized arrays are not.
3. **How do you verify that escape analysis eliminated an allocation?** Use JMH with `-prof gc` to measure allocation rates, or use `-XX:+PrintEscapeAnalysis` (requires `-XX:+UnlockDiagnosticVMOptions`).

---

### How does G1 GC decide which regions to collect first?

**Core Explanation**

G1 stands for **Garbage First** because it prioritizes collecting regions with the most garbage (lowest liveness). This maximizes the amount of memory reclaimed per unit of pause time.

**G1 heap structure:**
```
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  E  │  S  │  O  │  O  │  E  │  H  │  O  │  E  │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  E = Eden, S = Survivor, O = Old, H = Humongous
  Each region is 1-32 MB (power of 2, same size)
```

**Collection phases:**

1. **Young Collection (Minor GC)**: Collects **all** Eden and Survivor regions (mandatory, not selective).
2. **Concurrent Marking**: Identifies liveness in Old regions. Computes a **"garbage ratio"** for each Old region.
3. **Mixed Collection**: Collects all Young regions PLUS selected Old regions with the highest garbage ratio. The number of Old regions included is tuned to meet `-XX:MaxGCPauseMillis`.

**Selection criteria for Old regions in mixed collections:**
- Regions are sorted by **reclaimable space** (garbage bytes) descending.
- G1 selects regions starting from the most garbage until it estimates the pause budget will be exhausted.
- The flag `-XX:G1MixedGCLiveThresholdPercent=85` (default) means regions with >85% live data are skipped (not worth collecting).
- `-XX:G1HeapWastePercent=5` — stop mixed collections once reclaimable waste drops below 5%.

**Practical Example**

```bash
# G1 Mixed GC log example:
# [GC pause (G1 Evacuation Pause) (mixed)
#   Eden: 512M -> 0M
#   Survivors: 64M -> 64M
#   Old regions collected: 120 out of 400 (chose regions with most garbage)
#   Old: 2048M -> 1600M
#   Pause: 150ms (within 200ms target)
# ]

# Key tuning flags:
-XX:MaxGCPauseMillis=200                    # Pause target (ms)
-XX:G1MixedGCLiveThresholdPercent=85        # Skip regions with >85% live data
-XX:G1HeapWastePercent=5                     # Stop mixed GC when <5% reclaimable
-XX:G1MixedGCCountTarget=8                   # Spread mixed GC over 8 cycles
-XX:G1OldCSetRegionThresholdPercent=10       # Max % of Old regions per mixed GC
```

**Edge Cases / Pitfalls**
- **Humongous objects** (>50% of region size) are allocated in contiguous Old regions and are expensive to collect. They bypass the normal liveness sorting.
- If the concurrent marking cannot keep up with allocation, G1 falls back to a **Full GC** (single-threaded in Java 8, multi-threaded in Java 10+).
- Setting `-XX:MaxGCPauseMillis` too low causes G1 to collect too few Old regions per cycle, leading to heap exhaustion.

**Best Practices**
Avoid humongous objects by increasing `-XX:G1HeapRegionSize` or reducing large allocation sizes. Let G1's adaptive tuning work; only adjust `MaxGCPauseMillis` and `InitiatingHeapOccupancyPercent` as starting points.

**Common Follow-Up Questions**
1. **What is a "mixed" collection vs a "young" collection?** Young = only Young regions. Mixed = Young + selected Old regions. There is no "Old-only" collection in G1 (except Full GC).
2. **What triggers the start of concurrent marking?** When Old Gen occupancy crosses `InitiatingHeapOccupancyPercent` (default 45%). G1 adaptively adjusts this threshold.
3. **How does G1 handle humongous objects?** They are allocated in special humongous regions (contiguous Old regions). Since Java 8u60, they can be collected during Young GC if unreachable.

---

### What is biased locking? When is it revoked? (Note: removed in Java 15+)

**Core Explanation**

**Biased locking** was an optimization for uncontended synchronization. When a thread first acquires a monitor, the JVM "biases" the lock to that thread by writing the thread ID into the object's mark word. Subsequent lock acquisitions by the **same thread** require only a cheap thread-ID comparison — no atomic CAS operation.

**Lock escalation path:**
```
Biased (single thread, no CAS)
  → Thin/Lightweight Lock (CAS on mark word, short-lived contention)
    → Heavyweight Lock (OS mutex, contended)
```

**Biased lock revocation** occurs when:
1. Another thread attempts to acquire the biased lock.
2. `Object.hashCode()` is called on a biased object (the mark word has no room for both bias and identity hashcode).
3. Bulk revocation: too many revocations of the same class trigger bulk revocation for all instances of that class.

Revocation requires a **safepoint** (STW), which was a significant downside.

**Why removed in Java 15+:**
- Modern applications have high contention (thread pools, shared caches), making biased locking counterproductive.
- Revocation requires safepoints, adding unpredictable latency.
- The complexity in the JVM codebase was not justified by the diminishing benefit.
- Modern CAS instructions are fast enough that thin locking is nearly as cheap as biased locking.

**Practical Example**

```java
// Pre-Java 15: biased locking scenario
Object lock = new Object();

// Thread A (repeatedly acquires the lock uncontended):
for (int i = 0; i < 1_000_000; i++) {
    synchronized (lock) {  // First time: bias toward Thread A (mark word = Thread A ID)
        // work             // Subsequent: just check thread ID — very fast, no CAS
    }
}

// Thread B (occasionally acquires the same lock):
synchronized (lock) {
    // Biased lock revocation triggered!
    // 1. JVM initiates a safepoint (STW for ALL threads)
    // 2. Revokes bias from Thread A
    // 3. Escalates to thin lock (CAS-based)
    // 4. Thread B acquires thin lock
}
```

```bash
# Java 14 and below (biased locking on by default):
# -XX:+UseBiasedLocking (default)
# -XX:BiasedLockingStartupDelay=4000  # 4-second delay after JVM start

# Java 15+: deprecated and disabled by default
# -XX:+UseBiasedLocking  # still available but deprecated; removed in Java 18
```

**Edge Cases / Pitfalls**
- Biased locking had a startup delay (default 4 seconds) because JVM initialization uses many locks contentiously. This confused developers who benchmarked during startup.
- Calling `System.identityHashCode()` on a biased object forces revocation because the mark word cannot hold both bias metadata and the hash code.
- Bulk revocation (class-level) was triggered when revocation count for a class exceeded a threshold (`-XX:BiasedLockingBulkRevokeThreshold=40`).

**Best Practices**
On Java 15+, biased locking is gone — no action needed. On older JVMs, disable it (`-XX:-UseBiasedLocking`) if you see frequent safepoint pauses caused by revocation.

**Common Follow-Up Questions**
1. **What replaced biased locking?** Nothing directly. Thin (lightweight) locking with CAS is now the starting point. The CAS overhead on modern hardware is minimal.
2. **How can you see biased lock revocations in logs?** `-Xlog:biasedlocking=info` (Java 9-14). Look for "Revoking bias" messages at safepoints.
3. **Does `ReentrantLock` use biased locking?** No. Biased locking is a JVM optimization for `synchronized` only. `ReentrantLock` uses CAS-based algorithms in `AbstractQueuedSynchronizer`.

---

### How do ClassLoader memory leaks happen in application servers?

**Core Explanation**

In application servers (Tomcat, WildFly), each deployed application gets its own `ClassLoader`. On undeploy/redeploy, the old ClassLoader should become unreachable and be garbage collected, along with all classes it loaded.

A **ClassLoader leak** occurs when **any single reference** from outside the application's ClassLoader keeps it reachable. Since a ClassLoader holds references to every class it loaded, and each class holds references to its static fields, a single leaked reference can pin **hundreds of MB** of Metaspace and Heap.

**Common leak patterns:**

```
GC Root -> SomeObject -> Class -> WebAppClassLoader -> ALL loaded classes + static fields
```

1. **ThreadLocal not cleaned up**: a thread-pool thread survives redeployment with a `ThreadLocal` value whose class was loaded by the old ClassLoader.
2. **JDBC driver registration**: `DriverManager` (in the system ClassLoader) holds a reference to the driver (loaded by the webapp ClassLoader).
3. **Shutdown hooks**: `Runtime.addShutdownHook()` registers a thread whose class references the webapp ClassLoader.
4. **Static fields in shared libraries**: a JDK or server class caches an object from the webapp.
5. **Logging frameworks**: `java.util.logging.Level` keeps static references to custom levels.

**Practical Example**

```java
// Leak pattern 1: ThreadLocal not removed
public class RequestContext {
    // Thread pool thread survives redeploy; value.getClass().getClassLoader() = WebAppClassLoader
    private static final ThreadLocal<UserSession> context = new ThreadLocal<>();

    public static void set(UserSession session) { context.set(session); }

    // If remove() is never called, the WebAppClassLoader cannot be GC'd
    // Fix: call context.remove() in a servlet filter's finally block
}

// Leak pattern 2: JDBC driver
// On webapp start:
Class.forName("com.mysql.cj.jdbc.Driver"); // Registers with DriverManager

// On webapp stop: DriverManager still references the driver
// Fix: Explicitly deregister in contextDestroyed():
@Override
public void contextDestroyed(ServletContextEvent sce) {
    Enumeration<Driver> drivers = DriverManager.getDrivers();
    while (drivers.hasMoreElements()) {
        Driver driver = drivers.nextElement();
        if (driver.getClass().getClassLoader() == getClass().getClassLoader()) {
            DriverManager.deregisterDriver(driver);
        }
    }
}

// Leak pattern 3: Shared library caching webapp classes
// java.beans.Introspector caches BeanInfo keyed by class
// Fix: Introspector.flushCaches() on undeploy
```

**Edge Cases / Pitfalls**
- A single `Class` reference from outside the webapp ClassLoader retains the entire ClassLoader graph. It is extremely easy to leak.
- Tomcat's memory leak detection (`org.apache.catalina.loader.WebappClassLoaderBase`) warns about these issues on undeploy — heed those warnings.
- Even `java.util.logging.Level` custom subclasses can pin a ClassLoader via `Level.known` (a static `ArrayList`).

**Best Practices**
In `ServletContextListener.contextDestroyed()`: deregister JDBC drivers, clear `ThreadLocal`s, cancel timers, and stop threads started by your application. Use Tomcat's leak prevention listener.

**Common Follow-Up Questions**
1. **How do you diagnose a ClassLoader leak?** Take a heap dump, search for multiple instances of `WebAppClassLoader`, and trace GC roots to find what is retaining the old one.
2. **Does the module system (JPMS) prevent ClassLoader leaks?** No. JPMS controls access/visibility, not lifecycle. The same leak patterns apply.
3. **Why doesn't restarting the JVM have this issue?** Because all ClassLoaders are destroyed on JVM exit. The leak only manifests during hot redeployment.

---

### What is string deduplication in G1 GC? Does it help speed or only memory?

**Core Explanation**

**String deduplication** (`-XX:+UseStringDeduplication`, G1 only) identifies `String` objects that have the same content but different underlying `char[]`/`byte[]` arrays, and makes them share a single backing array.

**How it works:**
1. During Young GC, G1 identifies newly promoted `String` objects.
2. A background deduplication thread hashes the `String` content.
3. If another `String` with the same content exists, the new `String`'s internal `value` field is pointed to the existing array.
4. The duplicate array becomes unreachable and is collected by the next GC.

**Key points:**
- The `String` objects themselves are NOT deduplicated (different identity). Only the backing `byte[]` is shared.
- This is different from `String.intern()`, which deduplicates the `String` reference itself via the String Pool.

**Practical Example**

```java
// Without deduplication:
String a = new String("hello"); // a.value = byte[]{104,101,108,108,111}  (array A)
String b = new String("hello"); // b.value = byte[]{104,101,108,108,111}  (array B, separate copy)
// Two byte[] arrays exist in memory

// With G1 String Deduplication enabled:
// After dedup runs, both 'a' and 'b' point to the same byte[] array
// a.value == b.value (same array object), but a != b (different String objects)
// Array B is collected by GC -> memory saved
```

```bash
# Enable string deduplication
java -XX:+UseG1GC -XX:+UseStringDeduplication -jar myapp.jar

# Monitor deduplication statistics
-Xlog:stringdedup*=info
# Output:
# [stringdedup] Concurrent String Deduplication
#   Inspected:    125000
#   Deduplicated: 80000 (64.0%)
#   Saved:        15.2 MB
```

**Does it help speed or only memory?**
- **Primary benefit: memory savings** (10-40% reduction in String memory for String-heavy applications).
- **Indirect speed benefit**: less memory pressure means fewer/shorter GC pauses.
- **Slight CPU cost**: the dedup thread consumes CPU for hashing and comparison.
- **No direct execution speed improvement**: method calls, String operations, etc. are not faster.

**Edge Cases / Pitfalls**
- Only works with G1 GC (not Parallel, ZGC, or Shenandoah).
- Only deduplicates Strings that survive Young GC (promoted to Old Gen). Short-lived Strings are never processed.
- The dedup thread adds CPU overhead. For CPU-bound applications with few duplicate Strings, it is a net negative.
- `-XX:StringDeduplicationAgeThreshold=3` controls how many GC cycles a String must survive before being considered for dedup.

**Best Practices**
Enable string deduplication for applications with many long-lived duplicate Strings (e.g., web apps parsing JSON/XML with repeated field names). Monitor with `-Xlog:stringdedup*` to verify actual savings.

**Common Follow-Up Questions**
1. **How is this different from `String.intern()`?** `intern()` deduplicates the `String` reference (returns a canonical instance from the String Pool). G1 dedup shares the backing array but keeps separate `String` objects.
2. **Does it work with compact strings (Java 9+)?** Yes. It deduplicates the internal `byte[]` regardless of encoding (Latin-1 or UTF-16).
3. **Can ZGC do string deduplication?** As of Java 21, no. String deduplication is G1-specific. JEP work is ongoing.

---

### How does JVM warm-up affect benchmark results? What is the JMH framework?

**Core Explanation**

**JVM warm-up** refers to the period after JVM startup during which:
1. Classes are loaded and initialized lazily.
2. The interpreter runs code (slow).
3. The JIT compiler collects profiling data and compiles hot methods (C1, then C2).
4. CPU branch predictors and caches are populated.
5. GC stabilizes (heap sizing, tenuring thresholds).

Benchmarking during warm-up produces **misleadingly slow results** because you are measuring the interpreter, not the optimized compiled code. Conversely, benchmarking only after warm-up may miss startup-sensitive scenarios.

**JMH (Java Microbenchmark Harness)** is the standard framework for writing correct JVM benchmarks. It handles:
- **Warm-up iterations**: runs code enough times to trigger JIT compilation before measuring.
- **Dead code elimination prevention**: ensures the JIT does not optimize away the benchmark code.
- **Constant folding prevention**: stops the JIT from precomputing results.
- **Fork isolation**: runs each benchmark in a separate JVM to avoid profile pollution.
- **Statistical rigor**: reports mean, error margin, percentiles.

**Practical Example**

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)           // Measure average time per op
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)                        // Each thread gets its own state
@Warmup(iterations = 5, time = 1)           // 5 warm-up iterations (NOT measured)
@Measurement(iterations = 10, time = 1)     // 10 measured iterations
@Fork(2)                                    // Run in 2 separate JVM forks
public class StringBenchmark {

    private String data;

    @Setup
    public void setup() {
        data = "Hello, World! This is a benchmark string.";
    }

    @Benchmark
    public int stringLength() {
        return data.length(); // Return value consumed by JMH -> prevents dead code elimination
    }

    @Benchmark
    public String stringConcat() {
        return data + " suffix"; // Return prevents DCE
    }

    @Benchmark
    public String stringBuilderConcat() {
        return new StringBuilder(data).append(" suffix").toString();
    }
}
```

```bash
# Run JMH benchmark:
mvn clean package
java -jar target/benchmarks.jar

# Output:
# Benchmark                              Mode  Cnt    Score   Error  Units
# StringBenchmark.stringLength           avgt   20    3.125 ± 0.042  ns/op
# StringBenchmark.stringConcat           avgt   20   45.678 ± 1.234  ns/op
# StringBenchmark.stringBuilderConcat    avgt   20   38.456 ± 0.987  ns/op
```

**Common benchmarking mistakes JMH prevents:**

```java
// WRONG: Dead Code Elimination — JIT removes the entire computation
@Benchmark
public void wrong() {
    Math.sin(42.0); // Result unused -> JIT eliminates it
}

// CORRECT: Return the result (JMH's Blackhole consumes it)
@Benchmark
public double correct() {
    return Math.sin(42.0);
}

// WRONG: Constant folding — JIT precomputes at compile time
@Benchmark
public double wrongConstant() {
    return Math.sin(42.0); // 42.0 is a constant -> JIT may precompute
}

// CORRECT: Use @State fields (JIT cannot constant-fold instance fields)
@State(Scope.Thread)
public class MyBenchmark {
    double x = 42.0;

    @Benchmark
    public double correctState() {
        return Math.sin(x); // x is a field -> not constant-folded
    }
}
```

**Edge Cases / Pitfalls**
- **Profile pollution**: running multiple benchmarks in the same JVM contaminates JIT profiles. JMH's `@Fork` isolates each benchmark.
- **Loop optimizations**: writing a `for` loop inside a `@Benchmark` method can be optimized away. Use JMH's `@OperationsPerInvocation` or let JMH handle iteration.
- **GC interference**: short benchmarks may complete between GC cycles, while longer ones trigger GC. Use `-gc true` in JMH to force GC between iterations.
- JMH warm-up iterations should match your production warm-up behavior. For serverless/cold-start scenarios, measure with `@Warmup(iterations = 0)`.

**Best Practices**
Always use JMH for microbenchmarks. Never use `System.nanoTime()` loops. Run with at least 2 forks and 5+ warm-up iterations to ensure stable results.

**Common Follow-Up Questions**
1. **What is Blackhole in JMH?** A mechanism that consumes computed values to prevent the JIT from eliminating dead code. You can either return the result from `@Benchmark` or call `Blackhole.consume(result)`.
2. **What is `@Fork` and why is it important?** `@Fork(N)` runs the benchmark in N separate JVM processes. This prevents profile pollution from previous benchmarks and averages out JVM-level variance.
3. **How do you benchmark cold start?** Use `@Warmup(iterations = 0)` and `@Fork(value = 10)` to measure un-warmed performance across multiple JVM starts. This is relevant for serverless and CLI applications.

---
# Section 5: Collections Framework

---

## Basic Questions

---

### What is the difference between `List`, `Set`, and `Map`? When would you use each?

**Core Explanation:**

| | `List` | `Set` | `Map` |
|---|---|---|---|
| Duplicates | Allowed | Not allowed | Keys: unique; Values: duplicates OK |
| Order | Maintains insertion order (most impls) | Depends on impl | Depends on impl |
| Null | Allowed (multiple) | One null (HashSet) or none | At most one null key (HashMap) |
| Access by | Index | Iteration | Key |
| Common impls | `ArrayList`, `LinkedList` | `HashSet`, `TreeSet`, `LinkedHashSet` | `HashMap`, `TreeMap`, `LinkedHashMap` |

**When to use:**
- **List**: Ordered collection, duplicates allowed, positional access needed (shopping cart items, task queue).
- **Set**: Unique elements only, fast membership check (`contains`), deduplication (visited URLs, unique user IDs).
- **Map**: Key-to-value lookup (user by ID, configuration settings, word frequency count).

**Practical Example:**
```java
// List — ordered, allows duplicates
List<String> items = new ArrayList<>(List.of("apple", "banana", "apple"));
items.get(0); // "apple" — index access

// Set — no duplicates, fast contains
Set<String> visited = new HashSet<>();
visited.add("page1");
boolean seen = visited.contains("page1"); // O(1)

// Map — key-value lookup
Map<String, User> userCache = new HashMap<>();
userCache.put("alice", new User("alice"));
User user = userCache.get("alice"); // O(1)
```

---

### What is the difference between `ArrayList` and `LinkedList`?

**Core Explanation:**

| Operation | `ArrayList` | `LinkedList` |
|---|---|---|
| Backing structure | Dynamic array | Doubly linked list |
| Random access `get(i)` | O(1) | O(n) |
| `add` at end | O(1) amortized | O(1) |
| `add`/`remove` at middle | O(n) — shifts elements | O(n) — traversal to position |
| Memory overhead | Low (just the array) | High (each node has 2 pointers + data) |
| Cache locality | Excellent | Poor |
| Implements | `List`, `RandomAccess` | `List`, `Deque` |

**When to use `LinkedList`:** Almost never for pure `List` usage — `ArrayList` outperforms in most cases due to CPU cache locality. Use `LinkedList` as a `Deque` (double-ended queue) if you need `addFirst()`/`removeLast()` operations frequently.

**Practical Example:**
```java
// ArrayList — best for most List use cases
List<String> list = new ArrayList<>();
list.add("a"); list.add("b"); list.add("c");
String s = list.get(1); // O(1) — direct array access

// LinkedList as Deque
Deque<String> deque = new LinkedList<>();
deque.addFirst("first");
deque.addLast("last");
deque.removeFirst(); // O(1) — no shifting
```

**Common Follow-Up:**
- *When is `ArrayList` slower than `LinkedList`?* Only during frequent insertions/deletions at the beginning. But in practice, `ArrayDeque` beats `LinkedList` even as a queue.

---

### What is the difference between `HashSet` and `TreeSet`?

**Core Explanation:**

| | `HashSet` | `TreeSet` |
|---|---|---|
| Ordering | No order guaranteed | Sorted (natural order or `Comparator`) |
| Performance | O(1) add/remove/contains | O(log n) |
| Backing structure | HashMap (keys) | Red-Black Tree |
| Null allowed | Yes (one null) | No (null causes `NullPointerException`) |
| Implements | `Set` | `SortedSet`, `NavigableSet` |

**When to use:**
- `HashSet`: Default choice — fast membership testing, no ordering needed.
- `TreeSet`: When you need sorted order, range queries (`headSet()`, `tailSet()`), or `first()`/`last()` access.

```java
HashSet<Integer> hs = new HashSet<>(List.of(3, 1, 2));
System.out.println(hs); // [1, 2, 3] or any order — not guaranteed

TreeSet<Integer> ts = new TreeSet<>(List.of(3, 1, 2));
System.out.println(ts); // [1, 2, 3] — always sorted
System.out.println(ts.headSet(2)); // [1] — elements < 2
```

---

### What is the difference between `HashMap` and `Hashtable`?

**Core Explanation:**

| | `HashMap` | `Hashtable` |
|---|---|---|
| Thread safety | Not thread-safe | Thread-safe (synchronized methods) |
| Null keys | 1 null key allowed | Null keys throw NPE |
| Null values | Allowed | Null values throw NPE |
| Performance | Faster (no sync overhead) | Slower |
| Iteration | Fail-fast iterator | Enumerator (not fail-fast) |
| Introduced | Java 2 (Collections framework) | Java 1 (legacy) |
| Preferred modern alternative | `HashMap` + `ConcurrentHashMap` for threads | N/A — obsolete |

**Bottom line:** `Hashtable` is effectively deprecated. Use `HashMap` for single-threaded and `ConcurrentHashMap` for multi-threaded contexts.

---

### What is the difference between `Comparable` and `Comparator`?

**Core Explanation:**

| | `Comparable` | `Comparator` |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T o1, T o2)` |
| Where defined | Inside the class being compared | Separate class or lambda |
| Modifies class | Yes | No |
| Multiple orderings | No — one natural order | Yes — create different `Comparator`s |

**Practical Example:**
```java
// Comparable — natural ordering built into the class
class Employee implements Comparable<Employee> {
    String name; int salary;
    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.salary, other.salary); // natural = by salary
    }
}

// Comparator — external, flexible
Comparator<Employee> byName = Comparator.comparing(e -> e.name);
Comparator<Employee> bySalaryDesc = Comparator.comparingInt(Employee::getSalary).reversed();
Comparator<Employee> combined = Comparator
    .comparingInt(Employee::getSalary).reversed()
    .thenComparing(Employee::getName);

List<Employee> employees = ...;
employees.sort(combined);
```

**Best Practices:**
- Implement `Comparable` for the natural, default ordering (e.g., `String` by alphabetical).
- Use `Comparator` for ad-hoc sorting without modifying the class.

---

### What is the difference between `Iterator` and `ListIterator`?

**Core Explanation:**

| | `Iterator` | `ListIterator` |
|---|---|---|
| Direction | Forward only | Forward and backward |
| Works on | Any `Collection` | `List` only |
| Methods | `hasNext()`, `next()`, `remove()` | + `hasPrevious()`, `previous()`, `add()`, `set()`, `nextIndex()`, `previousIndex()` |

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// ListIterator — can go backwards and modify
ListIterator<String> li = list.listIterator(list.size()); // start at end
while (li.hasPrevious()) {
    System.out.print(li.previous() + " "); // c b a
}
```

---

### What is the `Iterable` interface and why is it important?

**Core Explanation:**

`Iterable<T>` has a single method: `Iterator<T> iterator()`. Any class implementing `Iterable` can be used in a **for-each loop**. The for-each loop is syntactic sugar over `Iterator`.

```java
// for-each desugars to:
for (String s : list) { ... }
// ↓
Iterator<String> it = list.iterator();
while (it.hasNext()) { String s = it.next(); ... }

// Custom Iterable
class NumberRange implements Iterable<Integer> {
    private final int start, end;
    NumberRange(int start, int end) { this.start = start; this.end = end; }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            int current = start;
            public boolean hasNext() { return current <= end; }
            public Integer next() { return current++; }
        };
    }
}

for (int n : new NumberRange(1, 5)) {
    System.out.print(n + " "); // 1 2 3 4 5
}
```

---

## Intermediate Questions

---

### How does `HashMap` work internally? Explain hashing, buckets, and what changed in Java 8 (treeification at threshold 8).

**Core Explanation:**

`HashMap` uses an array of **buckets** (called `table[]`). When you `put(key, value)`:
1. Compute `key.hashCode()`, then spread it: `hash = (h = key.hashCode()) ^ (h >>> 16)` (reduces collisions in lower bits).
2. Find bucket index: `index = hash & (capacity - 1)` (capacity is always power of 2).
3. If bucket is empty, place the entry. If occupied (collision), chain entries.

**Before Java 8:** Collisions used a singly linked list → O(n) worst case with many collisions.

**Java 8 change — treeification:** When a bucket's chain exceeds **8 entries** AND the total map has ≥ **64 entries**, that bucket's linked list is converted to a **Red-Black Tree** → O(log n) lookup instead of O(n). If entries drop back below 6, it converts back to a linked list.

```
HashMap internals:
table[0] → null
table[1] → Node("alice", user1) → Node("bob", user2)  // collision chain
table[2] → null
...
table[i] → TreeNode → TreeNode → ...  // treeified bucket (8+ entries)
```

**Practical Example:**
```java
HashMap<String, Integer> map = new HashMap<>(16, 0.75f);
// Initial capacity = 16, load factor = 0.75
// Resize triggers when size > 16 * 0.75 = 12 entries

map.put("alice", 1);
// 1. hash("alice") → some hash code
// 2. index = hash & 15 (capacity-1)
// 3. place in that bucket
```

---

### What happens when a `HashMap` reaches its load factor threshold? Describe the resizing process.

**Core Explanation:**

**Load factor** (default 0.75) determines when to resize. When `size > capacity * loadFactor`, HashMap **doubles its capacity** and **rehashes** all entries.

**Rehashing process:**
1. New array of double the old capacity is allocated.
2. Every existing entry is re-indexed: `newIndex = hash & (newCapacity - 1)`.
3. Old array is discarded.

**Why 0.75?** It's a balance: lower load factor = fewer collisions but more memory wasted; higher load factor = more collisions, slower lookups.

**Cost of resizing:** O(n) — all entries must be rehashed. For performance-critical code, set initial capacity to avoid resizes: `new HashMap<>(expectedSize / 0.75 + 1)`.

```java
// Avoid repeated resizing for known-size maps
int expectedSize = 10_000;
Map<String, User> map = new HashMap<>((int)(expectedSize / 0.75) + 1);
```

---

### Why does `HashMap` allow one null key but `ConcurrentHashMap` does not?

**Core Explanation:**

`HashMap.put(null, value)` special-cases null: it always maps null to bucket index 0.

`ConcurrentHashMap` rejects null keys/values because null creates ambiguity in concurrent contexts: `map.get(key)` returning `null` could mean **key not found** OR **key exists with null value** — you can't distinguish them safely without additional synchronization. Since `ConcurrentHashMap` is designed for concurrent use, this ambiguity must be eliminated.

**Practical implication:**
```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put(null, "value"); // throws NullPointerException

// The problem null causes in concurrent context:
if (!map.containsKey(key)) {   // thread-safe? No!
    map.put(key, compute(key)); // another thread may have put by now
}
// ConcurrentHashMap provides putIfAbsent() and computeIfAbsent() for atomicity
```

---

### Why is `HashMap` not thread-safe even for read-only operations?

**Core Explanation:**

Even reads can be unsafe during a **concurrent resize**. When another thread triggers a rehash:
1. The internal array reference is being replaced.
2. The old array may have partially moved entries.
3. A reading thread may see an inconsistent intermediate state.

In Java 7, concurrent `put()` operations could create **infinite loops** in the linked-list chain during resize (a known bug fixed in Java 8 with immutable nodes).

**Modern Java 8+:** HashMap uses immutable nodes during rehash, so infinite loops are fixed, but visibility guarantees are still absent — a reading thread may see stale values or nulls without `volatile` or synchronization.

**Solution:** Use `ConcurrentHashMap` for concurrent access, or `Collections.synchronizedMap()` if you prefer simplicity with coarser locking.

---

### What is the difference between `HashMap`, `LinkedHashMap`, and `TreeMap`? When would you use each?

**Core Explanation:**

| | `HashMap` | `LinkedHashMap` | `TreeMap` |
|---|---|---|---|
| Ordering | None | Insertion order (or access order) | Sorted by key |
| Performance | O(1) avg | O(1) avg (slightly slower for maintaining order) | O(log n) |
| Null keys | 1 allowed | 1 allowed | Not allowed |
| Use case | General purpose | LRU cache, ordered iteration | Range queries, sorted output |

```java
// LinkedHashMap — insertion order preserved
Map<String, Integer> linked = new LinkedHashMap<>();
linked.put("c", 3); linked.put("a", 1); linked.put("b", 2);
System.out.println(linked.keySet()); // [c, a, b] — insertion order

// Access-order LinkedHashMap (basis of LRU cache)
Map<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
        return size() > 3; // evict when > 3 entries
    }
};

// TreeMap — sorted by key
TreeMap<String, Integer> tree = new TreeMap<>();
tree.put("banana", 2); tree.put("apple", 1); tree.put("cherry", 3);
System.out.println(tree.firstKey()); // "apple"
System.out.println(tree.headMap("cherry")); // {apple=1, banana=2}
```

---

### How does `LinkedHashMap` maintain insertion order? How does access-order mode work?

**Core Explanation:**

`LinkedHashMap` extends `HashMap` and adds a **doubly linked list** running through all entries in insertion order. Every `put()` adds to the tail of this list.

**Access-order mode** (`new LinkedHashMap<>(16, 0.75f, true)`): Every time an entry is **accessed** (via `get()` or `put()`), it moves to the **tail** of the list. The **head** becomes the least recently accessed — perfect for LRU caches.

```java
LinkedHashMap<String, Integer> accessOrder = new LinkedHashMap<>(16, 0.75f, true);
accessOrder.put("a", 1);
accessOrder.put("b", 2);
accessOrder.put("c", 3);
accessOrder.get("a"); // "a" moves to the tail

System.out.println(accessOrder.keySet()); // [b, c, a] — "a" most recently accessed
```

---

### How does `TreeMap` maintain sorted order internally (Red-Black tree)?

**Core Explanation:**

`TreeMap` uses a **Red-Black Tree** (self-balancing BST) where each node has a color (red/black) with invariants that guarantee O(log n) operations. The tree stays balanced by rotating nodes and recoloring after insertions/deletions.

**Properties of Red-Black Tree:**
1. Every node is red or black.
2. Root is black.
3. Red nodes have black children.
4. Every path from root to null has the same number of black nodes.

These properties guarantee the tree height is at most 2·log₂(n+1), ensuring O(log n) worst case.

---

### What is the difference between `ConcurrentHashMap` and `Collections.synchronizedMap()`?

**Core Explanation:**

| | `ConcurrentHashMap` | `Collections.synchronizedMap()` |
|---|---|---|
| Lock granularity | Bucket-level (CAS + segment locks) | Entire map (one mutex) |
| Read operations | Lock-free (in modern impl) | Synchronized (blocks readers) |
| Throughput | High — concurrent reads + partial writes | Low — serial access |
| Iterator | Weakly consistent — no `ConcurrentModificationException` | Requires external sync during iteration |
| Null keys/values | Not allowed | Allowed (depends on wrapped map) |

```java
// synchronizedMap — simple but low concurrency
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
// Must synchronize externally during iteration:
synchronized (syncMap) {
    for (Map.Entry<String, Integer> e : syncMap.entrySet()) { ... }
}

// ConcurrentHashMap — high throughput
ConcurrentHashMap<String, Integer> chm = new ConcurrentHashMap<>();
chm.put("a", 1);                           // fine
chm.putIfAbsent("b", 2);                   // atomic
chm.computeIfAbsent("c", k -> k.length()); // atomic compound operation
// Iteration is safe without external lock (weakly consistent)
for (Map.Entry<String, Integer> e : chm.entrySet()) { ... }
```

---

### How does `ConcurrentHashMap` achieve thread safety without locking the entire map (segment locking vs CAS)?

**Core Explanation:**

**Java 7:** Used **16 segments** (each a mini `HashMap` with its own lock). Writes to different segments were concurrent. Reads were generally lock-free within a segment.

**Java 8+:** Removed segments. Now uses:
- **CAS (Compare-And-Swap)** for inserting into empty buckets — no lock at all.
- **`synchronized` on the first node of a bucket** for writes to a non-empty bucket.
- Reads are **lock-free** — entries are `volatile`, ensuring visibility.

This achieves:
- Concurrent reads from all buckets simultaneously.
- Concurrent writes to different buckets simultaneously.
- Only serialized writes **within the same bucket**.

```
16 threads can write to 16 different buckets simultaneously.
Reading never blocks (except during resize for a specific bucket being migrated).
```

---

### What is the difference between fail-fast and fail-safe iterators? Give examples of each.

**Core Explanation:**

| | Fail-Fast | Fail-Safe |
|---|---|---|
| Behavior on modification | Throws `ConcurrentModificationException` | No exception |
| Mechanism | Checks `modCount` on each iteration | Iterates over a copy or weakly consistent view |
| Memory | No copy — uses original structure | May use copy (overhead) |
| Examples | `ArrayList`, `HashMap`, `HashSet` | `ConcurrentHashMap`, `CopyOnWriteArrayList` |

```java
// Fail-fast
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    list.remove(s); // ConcurrentModificationException!
}

// Correct removal during iteration:
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // safe — uses iterator's remove()
}

// Fail-safe
ConcurrentHashMap<String, Integer> chm = new ConcurrentHashMap<>();
chm.put("a", 1);
for (String key : chm.keySet()) {
    chm.put("b", 2); // OK — no exception, but "b" may or may not appear in this iteration
}

// CopyOnWriteArrayList — iterates over snapshot
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>(List.of("a", "b"));
for (String s : cowList) {
    cowList.add("c"); // safe — iterator uses old snapshot
}
```

---

### How do you avoid `ConcurrentModificationException`?

**Core Explanation:**

1. **Use `Iterator.remove()`** instead of `collection.remove()` during iteration.
2. **Collect items to remove** and remove after iteration.
3. **Use `removeIf()`** (Java 8+) — handles it internally.
4. **Use a copy** to iterate, modify the original.
5. **Use concurrent collections** (`CopyOnWriteArrayList`, `ConcurrentHashMap`).

```java
List<String> list = new ArrayList<>(List.of("a", "bb", "ccc", "d"));

// Option 1: removeIf
list.removeIf(s -> s.length() > 1);

// Option 2: collect + remove
List<String> toRemove = list.stream()
    .filter(s -> s.length() > 1)
    .collect(Collectors.toList());
list.removeAll(toRemove);

// Option 3: Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().length() > 1) it.remove();
}
```

---

### What happens if you override `equals()` but not `hashCode()`? Demonstrate the bug.

**Core Explanation:**

HashMap/HashSet uses `hashCode()` to find the bucket, then `equals()` to find the entry within the bucket. If two equal objects have different hash codes, they land in different buckets — `contains()` returns `false` even for logically equal objects.

```java
class Product {
    String id;
    Product(String id) { this.id = id; }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Product)) return false;
        return id.equals(((Product) o).id);
    }
    // No hashCode override — uses Object's default (memory address-based)
}

Set<Product> products = new HashSet<>();
products.add(new Product("P001"));

Product search = new Product("P001");
System.out.println(products.contains(search)); // FALSE! Different hash codes → different buckets

// Fix: add hashCode
@Override
public int hashCode() { return Objects.hash(id); }
System.out.println(products.contains(search)); // TRUE after fix
```

---

### How does a `Set` prevent duplicate elements internally?

**Core Explanation:**

`HashSet` is backed by a `HashMap` where elements are stored as **keys** and a dummy constant (`PRESENT`) as the value. When you `add(element)`:
1. Compute the element's hash code → find bucket.
2. Check if any existing key in that bucket is equal (via `equals()`).
3. If equal key found → don't add (return `false`).
4. If not found → add as a new key entry (return `true`).

`TreeSet` uses a `TreeMap` similarly — it stores elements as keys and uses the `compareTo()`/`Comparator` to detect duplicates.

---

### How does `PriorityQueue` work internally (min-heap)?

**Core Explanation:**

`PriorityQueue` is backed by an **array-based binary heap**. The smallest element (by natural ordering or `Comparator`) is always at the root (index 0).

**Heap properties:**
- Parent at index `i`; children at `2i+1` and `2i+2`.
- Parent ≤ children (min-heap).
- Insertion: add to end, "sift up" to restore heap property — O(log n).
- Removal of head: replace root with last element, "sift down" — O(log n).
- Peek (head): O(1).

```java
PriorityQueue<Integer> pq = new PriorityQueue<>(); // min-heap
pq.offer(5); pq.offer(1); pq.offer(3);
System.out.println(pq.poll()); // 1 (smallest)
System.out.println(pq.poll()); // 3

// Max-heap
PriorityQueue<Integer> maxPq = new PriorityQueue<>(Comparator.reverseOrder());
maxPq.offer(5); maxPq.offer(1); maxPq.offer(3);
System.out.println(maxPq.poll()); // 5

// Custom priority
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
);
```

---

### What is `CopyOnWriteArrayList`? When is it appropriate and when is it a performance trap?

**Core Explanation:**

`CopyOnWriteArrayList` creates a **fresh copy of the underlying array** on every write operation (`add`, `set`, `remove`). Reads operate on the current snapshot without locking.

**When appropriate:**
- Read-heavy, write-rare scenarios (event listener lists, observer patterns).
- Situations requiring safe iteration during concurrent modifications.

**Performance trap:**
- Heavy write load → O(n) copy on every write → excessive GC pressure.
- Large lists with frequent writes are catastrophically slow.

```java
// Good use case — listener list (added once, iterated many times)
CopyOnWriteArrayList<EventListener> listeners = new CopyOnWriteArrayList<>();
listeners.add(listener1);
// Multiple threads can safely iterate:
for (EventListener l : listeners) { l.onEvent(event); }
// If another thread adds during iteration → no CME, uses snapshot

// Bad use case — high write throughput
CopyOnWriteArrayList<String> log = new CopyOnWriteArrayList<>();
// DO NOT do this in a loop — each add copies the entire list
for (int i = 0; i < 100_000; i++) log.add("entry" + i); // very slow!
// Alternative: ConcurrentLinkedQueue, BlockingQueue, or synchronizedList
```

---

### How do you implement custom sorting using `Comparator` with lambda expressions?

**Core Explanation:**

```java
record Employee(String name, int salary, String department) {}

List<Employee> employees = List.of(
    new Employee("Charlie", 90_000, "Engineering"),
    new Employee("Alice", 75_000, "Engineering"),
    new Employee("Bob", 90_000, "Marketing")
);

// Single field
employees.stream()
    .sorted(Comparator.comparingInt(Employee::salary))
    .forEach(System.out::println);

// Multiple fields: salary desc, then name asc
employees.stream()
    .sorted(Comparator.comparingInt(Employee::salary).reversed()
                      .thenComparing(Employee::name))
    .forEach(System.out::println);

// Null-safe sorting
List<String> withNulls = Arrays.asList("banana", null, "apple");
withNulls.sort(Comparator.nullsLast(Comparator.naturalOrder()));
// [apple, banana, null]
```

---

## Advanced Questions

---

### Why is `HashSet` faster than `ArrayList` for `contains()` but slower for iteration?

**Core Explanation:**

**`contains()` performance:**
- `HashSet.contains()`: O(1) average — computes hash, jumps directly to bucket, checks equality.
- `ArrayList.contains()`: O(n) — linear scan of every element.

**Iteration performance:**
- `ArrayList` stores elements in a contiguous array → **excellent CPU cache locality** → tight `for` loop with sequential memory access.
- `HashSet` iterates over the backing array (which has many empty buckets) AND possibly linked chains or trees → **poor cache locality** → many cache misses.

**Numbers:** On modern hardware, iterating 1M elements in `ArrayList` is 2-5x faster than `HashSet` due to cache efficiency.

---

### What is `WeakHashMap`? How does it decide when to evict keys? Give a real caching use case.

**Core Explanation:**

`WeakHashMap` stores **weak references** to its keys. When a key has no strong references elsewhere in the application, it becomes eligible for GC. After the key is collected, the corresponding entry is automatically removed from the map.

**Eviction mechanism:**
1. Keys are stored as `WeakReference<K>` objects.
2. When GC collects a key, it enqueues the `WeakReference` in a `ReferenceQueue`.
3. On the next `WeakHashMap` operation, stale entries are purged from the queue.

**Real use case — resource metadata cache:**
```java
// Cache metadata about objects without preventing their GC
WeakHashMap<Object, String> metadataCache = new WeakHashMap<>();

Object resource = new SomeLargeObject();
metadataCache.put(resource, "metadata for large object");

// When 'resource' goes out of scope and GC runs,
// the WeakHashMap entry is automatically removed
resource = null; // the SomeLargeObject can now be GC'd
// GC runs...
System.out.println(metadataCache.size()); // 0 — entry evicted
```

**Pitfall:** Do NOT use `WeakHashMap` with `String` literals or `Integer` constants as keys — they are often strongly referenced by the JVM and will never be GC'd.

---

### When would you choose `TreeMap` over `HashMap` in a production system?

**Core Explanation:**

Choose `TreeMap` when you need:
1. **Sorted key iteration**: `entrySet()` always in ascending key order.
2. **Range queries**: `subMap(from, to)`, `headMap(to)`, `tailMap(from)`.
3. **Floor/ceiling queries**: `floorKey(k)`, `ceilingKey(k)` — find nearest key.
4. **Ordered min/max**: `firstKey()`, `lastKey()`.

**Production scenarios:**
```java
// Leaderboard — always sorted by score
TreeMap<Integer, String> leaderboard = new TreeMap<>(Comparator.reverseOrder());
leaderboard.put(1500, "Alice");
leaderboard.put(2000, "Bob");
leaderboard.put(1750, "Charlie");
// Iterate top 3 in order: Bob, Charlie, Alice

// Time-series data — range query by timestamp
TreeMap<LocalDateTime, Metric> metrics = new TreeMap<>();
metrics.put(LocalDateTime.now(), new Metric());
SortedMap<LocalDateTime, Metric> lastHour = metrics.tailMap(LocalDateTime.now().minusHours(1));

// Rate limiting — sliding window by timestamp bucket
```

**Accept O(log n) cost** for the sorting benefit. If you don't need ordering, stick with O(1) `HashMap`.

---

### How is `hashCode()` generated for `String`? What makes a good hash function?

**Core Explanation:**

`String.hashCode()` in Java:
```java
// Polynomial hash: s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
int hash = 0;
for (char c : value) {
    hash = 31 * hash + c;
}
```

**Why 31?** It's prime (reduces collisions), odd (avoids even-distribution issues), and `31 * x = (x << 5) - x` (efficient bitshift optimization).

**Properties of a good hash function:**
1. **Deterministic**: Same input always produces same output.
2. **Uniform distribution**: Hash values spread evenly across the range.
3. **Avalanche effect**: Small input changes produce very different hashes.
4. **Fast to compute**.
5. **Minimize collisions**.

`String.hashCode()` caches the result in a field — it's only computed once per `String` object.

---

### How does `ConcurrentHashMap.size()` work? Why is it not constant-time?

**Core Explanation:**

In Java 8+ `ConcurrentHashMap`, `size()` doesn't maintain a single counter (which would require locking). Instead, it uses a **distributed counter** approach:
- Writes update one of several `LongAdder`-like counter cells based on thread contention.
- `size()` sums all counter cells.

This is O(number of counter cells) — not constant-time, but still very fast and doesn't require locking the entire map.

**Why not a single `AtomicLong`?** Under high contention, a single `AtomicLong` causes CAS failures and retries. Distributed cells reduce contention.

**Implication:** `size()` is approximate in a concurrent context — between the time you call `size()` and use the result, other threads may have modified the map.

---

### How does `CopyOnWriteArrayList` perform under heavy write load? What is the alternative?

**Core Explanation:**

Under heavy writes, `CopyOnWriteArrayList` is catastrophically slow:
- Each `add()` allocates a new array of size n+1, copies all n elements, then replaces the reference.
- For a list of n elements with w writes: O(n·w) total work and O(n·w) GC pressure.

**Alternatives by use case:**

| Use Case | Alternative |
|---|---|
| Queue/deque operations | `ConcurrentLinkedQueue`, `ArrayDeque` with locks |
| Producer-consumer | `LinkedBlockingQueue`, `ArrayBlockingQueue` |
| Mixed read/write | `ConcurrentHashMap` (for map use cases) |
| Read-heavy with occasional writes | `CopyOnWriteArrayList` is fine |
| High-write list | `Collections.synchronizedList()` + `ArrayList` with explicit sync |

---

### What is the `NavigableMap` interface? What operations does it add over `SortedMap`?

**Core Explanation:**

`NavigableMap` extends `SortedMap` with navigation methods for finding entries relative to a given key:

```java
NavigableMap<Integer, String> nm = new TreeMap<>();
nm.put(1, "a"); nm.put(3, "c"); nm.put(5, "e"); nm.put(7, "g");

nm.floorKey(4);         // 3 — largest key ≤ 4
nm.ceilingKey(4);       // 5 — smallest key ≥ 4
nm.lowerKey(5);         // 3 — largest key < 5 (strictly)
nm.higherKey(5);        // 7 — smallest key > 5 (strictly)
nm.floorEntry(4);       // Entry(3, "c")

nm.descendingMap();     // reverse order view
nm.headMap(5, true);    // {1=a, 3=c, 5=e} — inclusive
nm.subMap(2, true, 6, false); // {3=c, 5=e}

nm.pollFirstEntry();    // removes and returns {1=a}
nm.pollLastEntry();     // removes and returns {7=g}
```

---

### How would you implement an LRU cache using only JDK collections?

**Core Explanation:**

Use `LinkedHashMap` in **access-order mode** with `removeEldestEntry()` override.

```java
public class LRUCache<K, V> {
    private final Map<K, V> cache;

    public LRUCache(int capacity) {
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > capacity;
            }
        };
    }

    public synchronized V get(K key) {
        return cache.getOrDefault(key, null);
        // getOrDefault triggers access-order update
    }

    public synchronized void put(K key, V value) {
        cache.put(key, value); // moves to tail, evicts eldest if over capacity
    }
}

// Usage
LRUCache<Integer, String> lru = new LRUCache<>(3);
lru.put(1, "one");
lru.put(2, "two");
lru.put(3, "three");
lru.get(1);          // "one" becomes most recently used
lru.put(4, "four");  // evicts "two" (least recently used)
```

**Note:** Add `synchronized` for thread safety, or use `ConcurrentHashMap` + `ConcurrentLinkedQueue` for a non-blocking LRU.

---

### Sorting millions of objects — `Comparable` or `Comparator`? What are the performance considerations?

**Core Explanation:**

**For pure sorting performance, both are equivalent** — `Arrays.sort()` and `Collections.sort()` use TimSort (O(n log n)) regardless.

**Performance considerations at scale:**

1. **Method call overhead**: `compareTo()` and `compare()` are called O(n log n) times. Keep comparisons simple.
2. **Object creation**: `Comparator.comparing(...)` creates lambda objects — create them once, not inside loops.
3. **Primitive arrays**: Sort `int[]` not `Integer[]` — avoids autoboxing and uses dual-pivot quicksort (faster than TimSort for primitives).
4. **Parallel sort**: `Arrays.parallelSort()` uses fork-join for large arrays (>8192 elements).
5. **Custom comparator vs natural order**: Natural order (Comparable) has no extra method reference overhead.

```java
// Best performance for millions of objects
Employee[] employees = ...; // array, not List

// Option 1: natural ordering (Comparable)
Arrays.parallelSort(employees); // parallel, no lambda overhead

// Option 2: Comparator — create ONCE
Comparator<Employee> comp = Comparator.comparingInt(Employee::getSalary)
                                      .thenComparing(Employee::getName);
Arrays.parallelSort(employees, comp);

// Worst: Comparator created inside loop or repeated sort calls
// Never: sort List<Integer> when you can sort int[]
```
# Section 6: Multithreading & Concurrency

---

## Basic Questions

---

### What problem does multithreading solve? When can it make things worse?

**Core Explanation:**

Multithreading allows a program to execute multiple tasks **concurrently**, improving:
- **Responsiveness**: Keep UI responsive while doing background work.
- **Throughput**: Utilize multiple CPU cores for CPU-bound work.
- **Resource utilization**: Overlap I/O wait time with computation.

**When it makes things worse:**
- **CPU-bound tasks with little data**: Thread creation and context-switching overhead exceeds the benefit.
- **Shared mutable state**: Requires synchronization → contention → serialized execution.
- **Small workloads**: Splitting 100 items across 8 threads adds overhead that outweighs the parallelism gain.
- **I/O-bound without async**: Blocking threads waste OS resources (each thread holds ~1MB stack).

**Rule of thumb:** Multithreading helps when tasks are independent, long enough to justify the overhead, and ideally I/O-bound or CPU-bound on separate data sets.

---

### What is the thread lifecycle in Java? Explain the difference between `start()` and `run()`.

**Core Explanation:**

**Thread states:**
```
NEW → RUNNABLE → BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```
- **NEW**: Thread created, not yet started.
- **RUNNABLE**: `start()` called; may be running or waiting for CPU.
- **BLOCKED**: Waiting to acquire a monitor lock.
- **WAITING**: Waiting indefinitely (`wait()`, `join()`).
- **TIMED_WAITING**: Waiting with timeout (`sleep(ms)`, `wait(ms)`).
- **TERMINATED**: Thread has completed execution.

**`start()` vs `run()`:**
- `start()`: Creates a new OS thread and calls `run()` on it concurrently.
- `run()`: Executes the task on the **calling thread** — no new thread is created.

```java
Thread t = new Thread(() -> System.out.println(Thread.currentThread().getName()));

t.start(); // prints "Thread-0" — new thread
t.run();   // prints "main" — same thread! (Don't do this)
```

---

### What is the difference between `Runnable` and `Callable`?

**Core Explanation:**

| | `Runnable` | `Callable<V>` |
|---|---|---|
| Return value | `void` | Returns `V` |
| Exception | Cannot throw checked exceptions | Can throw checked exceptions |
| Since | Java 1.0 | Java 5 |
| Used with | `Thread`, `Executor` | `ExecutorService.submit()` → `Future<V>` |

```java
// Runnable
Runnable task = () -> System.out.println("Running");
new Thread(task).start();

// Callable — returns a result
Callable<Integer> computation = () -> {
    Thread.sleep(1000);
    return 42;
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(computation);
Integer result = future.get(); // blocks until result is available
```

---

### What is the difference between `wait()` and `sleep()`?

**Core Explanation:**

| | `wait()` | `sleep()` |
|---|---|---|
| Class | `Object` | `Thread` (static) |
| Releases lock | Yes | No |
| Woken by | `notify()` / `notifyAll()` / timeout | Timeout only |
| Requires `synchronized` | Yes — must own the monitor | No |
| Use case | Inter-thread communication | Pausing execution |

```java
// wait() — releases lock and waits for signal
synchronized (queue) {
    while (queue.isEmpty()) {
        queue.wait(); // releases lock, waits for notify
    }
    process(queue.poll());
}

// sleep() — holds the lock, just pauses
synchronized (this) {
    Thread.sleep(1000); // HOLDS the lock! Other threads can't enter
}
```

---

### What is the `synchronized` keyword? What does it guarantee?

**Core Explanation:**

`synchronized` acquires a **monitor lock** (intrinsic lock) on an object. It provides:
1. **Mutual exclusion**: Only one thread executes the synchronized block at a time.
2. **Visibility**: Changes made inside a synchronized block are visible to other threads that subsequently synchronize on the same object.

```java
class Counter {
    private int count = 0;
    private final Object lock = new Object();

    // synchronized method — locks on 'this'
    public synchronized void increment() { count++; }

    // synchronized block — more granular
    public void decrement() {
        synchronized (lock) {
            count--;
        }
    }

    // Static synchronized — locks on Counter.class
    public static synchronized void staticMethod() { ... }
}
```

**Guarantees:** No two threads can be inside synchronized blocks that share the same lock simultaneously. Also ensures happens-before: release of lock happens-before acquire of the same lock.

---

### What is `volatile`? What problem does it solve?

**Core Explanation:**

`volatile` tells the JVM that a variable's value may be changed by multiple threads and should:
1. **Always be read from and written to main memory** (bypasses CPU caches/registers).
2. **Prevent instruction reordering** around the volatile read/write.

**Problem it solves:** Without `volatile`, a thread may cache a variable's value in its CPU register or L1 cache and never see updates from other threads.

```java
class Server {
    private volatile boolean running = true; // without volatile, loop may never see false

    void stop() { running = false; }

    void serve() {
        while (running) { // reads from main memory on every iteration
            handleRequest();
        }
    }
}
```

**What `volatile` does NOT guarantee:** Atomicity. `count++` is read-modify-write — three operations. Even on a volatile field, this is not atomic.

---

### What is a deadlock? What are the four necessary conditions?

**Core Explanation:**

A **deadlock** occurs when two or more threads are permanently blocked, each waiting for a resource held by another.

**Four necessary conditions (Coffman conditions):**
1. **Mutual exclusion**: Resources can only be held by one thread.
2. **Hold and wait**: A thread holds a resource while waiting for another.
3. **No preemption**: Resources cannot be forcibly taken.
4. **Circular wait**: Thread A waits for B; B waits for A (cycle).

```java
// Classic deadlock
Object lockA = new Object(), lockB = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lockA) {
        Thread.sleep(100);
        synchronized (lockB) { System.out.println("T1 done"); }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lockB) {   // t2 holds lockB
        Thread.sleep(100);
        synchronized (lockA) { System.out.println("T2 done"); } // waits for lockA
    }
});
// t1 holds lockA, wants lockB; t2 holds lockB, wants lockA → deadlock

// Prevention: always acquire locks in the same order
```

---

### What is a race condition?

**Core Explanation:**

A race condition occurs when the program's output depends on the **relative timing** of threads — the "race" to execute. The behavior is non-deterministic and often produces incorrect results.

```java
class BankAccount {
    private int balance = 1000;

    // NOT thread-safe — race condition
    public void withdraw(int amount) {
        if (balance >= amount) { // Thread A reads balance = 1000
            // Thread B also reads balance = 1000, passes check
            balance -= amount;  // Thread A: balance = 500
            // Thread B: balance = 0 (total withdrawn: 1500 — more than 1000!)
        }
    }

    // Thread-safe
    public synchronized void safeWithdraw(int amount) {
        if (balance >= amount) balance -= amount;
    }
}
```

---

### What is the difference between concurrency and parallelism?

**Core Explanation:**

- **Concurrency**: Multiple tasks are **in progress** at the same time — they may interleave on a single CPU (context switching). Concurrency is about **dealing with** multiple things at once.
- **Parallelism**: Multiple tasks execute **simultaneously** on multiple CPUs/cores. Parallelism is about **doing** multiple things at once.

**Analogy:**
- Concurrency: One barista making multiple drinks by switching between them.
- Parallelism: Multiple baristas, each making a drink simultaneously.

Java achieves:
- Concurrency: Threads on a single core (time-slicing).
- Parallelism: Threads on multiple cores (`ForkJoinPool`, `parallelStream()`).

---

## Intermediate Questions

---

### What is `ExecutorService`? What are the different types of thread pools (`fixed`, `cached`, `scheduled`, `single`)?

**Core Explanation:**

`ExecutorService` is a higher-level abstraction over raw threads. Instead of creating and managing threads manually, you submit tasks to a pool that manages thread lifecycle.

```java
// Fixed thread pool — fixed number of threads
ExecutorService fixed = Executors.newFixedThreadPool(4);
// Use for: CPU-bound tasks where # threads = # cores

// Cached thread pool — creates threads on demand, reuses idle ones
ExecutorService cached = Executors.newCachedThreadPool();
// Use for: many short-lived, lightweight tasks (I/O-heavy)
// Risk: unbounded thread creation under heavy load → OOM

// Single thread executor — one thread, tasks execute sequentially
ExecutorService single = Executors.newSingleThreadExecutor();
// Use for: ordered task execution, background serialized processing

// Scheduled — delay or periodic tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);
scheduled.scheduleAtFixedRate(task, 0, 30, TimeUnit.SECONDS); // every 30s

// Modern recommendation: use ThreadPoolExecutor directly
ExecutorService executor = new ThreadPoolExecutor(
    4,    // corePoolSize
    8,    // maxPoolSize
    60L, TimeUnit.SECONDS, // keepAliveTime
    new ArrayBlockingQueue<>(100), // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);
```

---

### What is `ForkJoinPool`? How does work-stealing improve performance?

**Core Explanation:**

`ForkJoinPool` is a specialized executor for **divide-and-conquer** tasks. It uses `ForkJoinTask` subclasses (`RecursiveTask<V>` with result, `RecursiveAction` without).

**Work-stealing algorithm:** Each thread has its own **deque** of tasks. When a thread finishes its tasks, it "steals" tasks from the **tail** of another thread's deque (while the owner works from the head). This minimizes contention and keeps all threads busy.

```java
class SumTask extends RecursiveTask<Long> {
    private final int[] arr;
    private final int start, end;
    static final int THRESHOLD = 1000;

    SumTask(int[] arr, int start, int end) {
        this.arr = arr; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += arr[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();              // async
        return right.compute() + left.join(); // right runs on current thread
    }
}

ForkJoinPool pool = new ForkJoinPool();
long result = pool.invoke(new SumTask(array, 0, array.length));
```

---

### What is the difference between `ExecutorService` and `ForkJoinPool`?

**Core Explanation:**

| | `ExecutorService` | `ForkJoinPool` |
|---|---|---|
| Task model | Independent tasks | Fork-join (recursive subdivision) |
| Queue | Single shared queue | Per-thread deques with work-stealing |
| Task type | `Runnable`/`Callable` | `ForkJoinTask` (`RecursiveTask`, `RecursiveAction`) |
| Best for | I/O-bound, independent tasks | CPU-bound, recursive divide-and-conquer |
| `parallelStream()` | Uses ForkJoinPool internally | Is ForkJoinPool |

---

### What is the difference between a synchronized method and a synchronized block?

**Core Explanation:**

```java
class Example {
    private int count = 0;
    private final Object lock = new Object();

    // Synchronized method — locks on 'this' for entire method duration
    public synchronized void incrementMethod() {
        count++;
        doOtherStuff(); // entire method is locked — possibly too coarse
    }

    // Synchronized block — locks only the critical section
    public void incrementBlock() {
        doNonCriticalWork(); // outside lock — other threads can proceed
        synchronized (this) {
            count++;          // only this section is locked
        }
        doMoreNonCritical(); // outside lock
    }

    // Synchronized block with explicit lock — more flexible
    public void incrementWithLock() {
        synchronized (lock) { // can use different lock per resource
            count++;
        }
    }
}
```

**Best practice:** Prefer synchronized blocks over methods — they minimize the locked region, reduce contention, and allow using separate locks for independent resources.

---

### What is object-level lock vs class-level lock?

**Core Explanation:**

- **Object-level lock** (`synchronized(this)` or `synchronized` instance method): Locks on the specific instance. Two threads can execute synchronized methods on **different instances** simultaneously.
- **Class-level lock** (`synchronized(MyClass.class)` or `static synchronized` method): Locks on the `Class` object. Only one thread can execute across **all instances**.

```java
class MyClass {
    // Object-level — each instance has its own lock
    public synchronized void instanceMethod() { ... }

    // Class-level — one lock for the class
    public static synchronized void classMethod() { ... }
}

MyClass a = new MyClass(), b = new MyClass();
// a.instanceMethod() and b.instanceMethod() can run simultaneously
// MyClass.classMethod() locks all instances — only one runs at a time
```

---

### Why does `volatile` not guarantee atomicity? Give an example with `count++`.

**Core Explanation:**

`count++` is actually three operations:
1. Read `count` from memory.
2. Increment the value.
3. Write the new value back.

Even if `count` is `volatile`, two threads can:
1. Both read `count = 5`.
2. Both compute `6`.
3. Both write `6` → result is `6` instead of `7` (one increment lost).

`volatile` only guarantees that reads and writes are visible — not that compound operations are atomic.

```java
volatile int count = 0;

// Thread 1: read=5, compute=6, write=6
// Thread 2: read=5 (before T1 writes), compute=6, write=6
// Result: count = 6, not 7

// Fix: use AtomicInteger
AtomicInteger atomicCount = new AtomicInteger(0);
atomicCount.incrementAndGet(); // atomic — CAS operation
```

---

### When should you prefer `volatile` over `synchronized`?

**Core Explanation:**

Use `volatile` when:
1. Only one thread **writes**, multiple threads **read** (the writer's changes must be immediately visible).
2. The operation is a **simple assignment** (not compound like `++`).
3. The variable is a **flag/state indicator** (e.g., `running`, `initialized`).

Use `synchronized` when:
1. Multiple threads write.
2. You need atomicity of compound operations.
3. You need mutual exclusion over a code block.

```java
// Good volatile use: single writer, multiple readers
private volatile boolean serviceInitialized = false;

void initialize() {
    // ... heavy init ...
    serviceInitialized = true; // single write — volatile is enough
}

boolean isReady() { return serviceInitialized; } // readers see true immediately

// Bad volatile use: multiple writers
private volatile int counter = 0;
counter++; // RACE CONDITION — use AtomicInteger instead
```

---

### What is the difference between `synchronized` and `Lock` (`ReentrantLock`)?

**Core Explanation:**

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Try to acquire | No — blocks indefinitely | `tryLock()` — non-blocking attempt |
| Timeout | No | `tryLock(timeout, unit)` |
| Interruptible | No | `lockInterruptibly()` |
| Fairness | Unfair (barging) | Can be fair: `new ReentrantLock(true)` |
| Condition variables | `wait()`/`notify()` (one per object) | Multiple `Condition`s per lock |
| Lock release | Automatic on block exit | Must call `unlock()` in `finally` |

```java
ReentrantLock lock = new ReentrantLock();

// Basic usage
lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // MUST release in finally
}

// tryLock — avoids blocking
if (lock.tryLock(200, TimeUnit.MILLISECONDS)) {
    try {
        // got the lock
    } finally {
        lock.unlock();
    }
} else {
    // couldn't acquire — do something else
}

// Multiple conditions
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();
// Producer signals notEmpty; consumer signals notFull
```

---

### What is `ReentrantLock`? What advantages does it have over `synchronized`?

**Core Explanation:**

`ReentrantLock` is a `Lock` implementation that supports **re-entrant** locking — the same thread can acquire the lock multiple times without deadlocking (tracks lock count).

**Advantages:**
1. **`tryLock()`**: Non-blocking acquisition — allows fallback behavior.
2. **Interruptible locking**: `lockInterruptibly()` — thread can be interrupted while waiting.
3. **Fair mode**: `new ReentrantLock(true)` — threads acquire lock in FIFO order (prevents starvation).
4. **Multiple `Condition` objects**: Fine-grained signaling (not possible with `synchronized`).
5. **Lock inspection**: `isHeldByCurrentThread()`, `getQueueLength()` for monitoring.

**When to use `synchronized` instead:** For simple critical sections where you don't need the advanced features — `synchronized` is less error-prone (no forgotten `unlock()`).

---

### What are atomic variables (`AtomicInteger`, `AtomicReference`)? What is CAS?

**Core Explanation:**

Atomic variables provide **lock-free thread-safe operations** using **Compare-And-Swap (CAS)** hardware instructions. CAS is an atomic operation: "if current value == expected, set to new value; else fail."

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet(); // atomic — equivalent to synchronized count++
counter.compareAndSet(5, 10); // CAS: if counter == 5, set to 10

// AtomicReference — atomic reference swapping
AtomicReference<Config> configRef = new AtomicReference<>(new Config());
Config old = configRef.get();
Config newConfig = new Config(old.getSettings());
configRef.compareAndSet(old, newConfig); // atomic swap

// How CAS works internally:
// do {
//     int expected = value;
//     int newVal = expected + 1;
// } while (!compareAndSwap(expected, newVal)); // retry on failure
```

**Advantage over synchronized:** No locking → no blocking → higher throughput under low contention.
**Disadvantage:** Under high contention, many CAS failures cause "spin" overhead.

---

### What is `CountDownLatch`? Give a practical example (waiting for services to initialize).

**Core Explanation:**

`CountDownLatch` is a synchronizer where threads wait until a counter reaches zero. Once at zero, all waiting threads proceed. The latch cannot be reset.

```java
// Application startup — wait for all services to be ready
public class ServiceStartup {
    public static void main(String[] args) throws InterruptedException {
        int serviceCount = 3;
        CountDownLatch latch = new CountDownLatch(serviceCount);

        // Start each service in its own thread
        for (String service : List.of("Database", "Cache", "MessageQueue")) {
            new Thread(() -> {
                try {
                    initializeService(service);
                    System.out.println(service + " ready");
                    latch.countDown(); // decrement counter
                } catch (Exception e) {
                    latch.countDown(); // still decrement on failure
                }
            }).start();
        }

        latch.await(); // main thread blocks until count reaches 0
        System.out.println("All services ready — accepting traffic");
    }
}

// Testing: wait for async tasks
CountDownLatch testLatch = new CountDownLatch(1);
executorService.submit(() -> {
    process();
    testLatch.countDown();
});
assertTrue(testLatch.await(5, TimeUnit.SECONDS));
```

---

### What is `CyclicBarrier`? How does it differ from `CountDownLatch`?

**Core Explanation:**

| | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Reusable | No — one-shot | Yes — resets after each barrier |
| Who counts down | Any thread | Participating threads (each calls `await()`) |
| Action on completion | Nothing | Optional barrier action (runs once) |
| Use case | Wait for events | Synchronize threads at a rendezvous point |

```java
// CyclicBarrier — all threads must reach the barrier before any proceeds
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All workers reached checkpoint — proceeding");
});

for (int i = 0; i < 3; i++) {
    final int workerId = i;
    new Thread(() -> {
        doPartialWork(workerId);
        barrier.await(); // wait for all 3 threads
        // all 3 proceed after barrier action runs
        doNextPhase(workerId);
    }).start();
}
// After phase 1 completes, barrier resets for phase 2 automatically
```

---

### What is `Semaphore`? When would you use it?

**Core Explanation:**

`Semaphore` controls access to a resource by limiting concurrent users. It maintains a permit count:
- `acquire()`: Takes a permit (blocks if count is 0).
- `release()`: Returns a permit.

**Use cases:**
- Limit concurrent DB connections.
- Rate limiting API calls.
- Implementing bounded resource pools.

```java
// Allow only 5 concurrent database connections
Semaphore dbConnectionPool = new Semaphore(5);

void queryDatabase(String sql) throws InterruptedException {
    dbConnectionPool.acquire(); // wait for a slot
    try {
        // execute query using a connection
    } finally {
        dbConnectionPool.release(); // return the slot
    }
}

// Rate limiter — limit to 10 requests per second
Semaphore rateLimiter = new Semaphore(10);
ScheduledExecutorService replenisher = Executors.newSingleThreadScheduledExecutor();
replenisher.scheduleAtFixedRate(() -> {
    rateLimiter.release(10 - rateLimiter.availablePermits()); // refill to 10
}, 0, 1, TimeUnit.SECONDS);
```

---

### What is `BlockingQueue`? How is it used in the Producer-Consumer pattern?

**Core Explanation:**

`BlockingQueue` is a thread-safe queue that **blocks**:
- `put()`: Blocks if full.
- `take()`: Blocks if empty.

Implementations: `ArrayBlockingQueue` (bounded), `LinkedBlockingQueue` (bounded or unbounded), `PriorityBlockingQueue`, `SynchronousQueue`.

```java
public class ProducerConsumer {
    static final BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

    static class Producer implements Runnable {
        @Override public void run() {
            for (int i = 0; i < 20; i++) {
                try {
                    queue.put("task-" + i); // blocks if queue is full
                    System.out.println("Produced: task-" + i);
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }
        }
    }

    static class Consumer implements Runnable {
        @Override public void run() {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    String task = queue.take(); // blocks if queue is empty
                    System.out.println("Consumed: " + task);
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }
        }
    }
}
```

---

### What is `ThreadLocal`? Give a real use case (request-scoped data, transaction context).

**Core Explanation:**

`ThreadLocal<T>` provides each thread with its own isolated copy of a variable. No synchronization needed because no sharing occurs.

```java
// Real use case 1: Request context in web applications
public class RequestContext {
    private static final ThreadLocal<String> userId = new ThreadLocal<>();

    public static void setUserId(String id) { userId.set(id); }
    public static String getUserId() { return userId.get(); }
    public static void clear() { userId.remove(); } // CRITICAL in thread pools!
}

// In filter:
public class UserContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        try {
            RequestContext.setUserId(extractUserId(req));
            chain.doFilter(req, res);
        } finally {
            RequestContext.clear(); // must clear to prevent memory leaks in thread pools
        }
    }
}

// Service — no parameter passing needed:
public class OrderService {
    public void placeOrder(Order order) {
        String userId = RequestContext.getUserId(); // always current request's user
        audit(userId, order);
    }
}
```

---

### What is thread starvation? How does it differ from deadlock?

**Core Explanation:**

- **Deadlock**: All involved threads are permanently blocked, unable to proceed.
- **Thread starvation**: A thread can run but **never gets CPU time** because other threads consistently preempt it.

**Causes of starvation:**
1. Unfair lock acquisition (higher-priority threads always win).
2. Long-running synchronized methods blocking lower-priority threads.
3. Thread pool exhaustion — high-priority tasks flood the pool.

**Prevention:** Use fair locks (`new ReentrantLock(true)`), bounded thread pools, or priority-based task scheduling.

---

### What is thread safety? List five ways to achieve it in Java.

**Core Explanation:**

Thread safety means a class behaves correctly when accessed concurrently from multiple threads without requiring external synchronization.

**Five ways:**
1. **Immutability**: Immutable objects are inherently thread-safe (no state changes). Use `final` fields, defensive copies.
2. **Synchronization**: `synchronized` methods/blocks ensure mutual exclusion.
3. **Concurrent collections**: `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`.
4. **Atomic classes**: `AtomicInteger`, `AtomicReference` for lock-free operations.
5. **Thread confinement**: Keep data local to a single thread (`ThreadLocal`, stack-confined variables).

---

### How does immutability eliminate the need for synchronization?

**Core Explanation:**

Synchronization is needed for **shared mutable state**. If state cannot be changed, there's nothing to protect. Multiple threads can read an immutable object simultaneously without any coordination because:
- No writes can occur → no visibility issues.
- No state changes → no inconsistency possible.
- No need for happens-before guarantees → no locking.

```java
// Immutable — zero synchronization needed
final class Money {
    private final long amount;
    private final Currency currency;

    Money(long amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
    // No setters — freely shared across threads
    Money add(Money other) { return new Money(this.amount + other.amount, currency); }
}

// String, BigDecimal, LocalDate — all immutable → thread-safe
```

---

### Does `Thread.sleep()` release a lock? Does `wait()` release a lock?

**Core Explanation:**

- **`Thread.sleep()`**: Does **NOT** release any locks. The thread sleeps while holding all its monitors.
- **`Object.wait()`**: **Releases** the object's monitor lock and enters WAITING state. When notified, it re-acquires the lock before continuing.

```java
synchronized (lock) {
    Thread.sleep(1000); // holds 'lock' — other threads CANNOT enter any synchronized(lock) block!
}

synchronized (lock) {
    lock.wait(); // releases 'lock' — other threads CAN enter synchronized(lock) blocks
    // after notify(), re-acquires lock before continuing
}
```

---

### Why must `wait()` be called inside a `synchronized` block?

**Core Explanation:**

`wait()` must be inside `synchronized` because:
1. The thread must **own the monitor** to call `wait()` — otherwise `IllegalMonitorStateException`.
2. The condition check and `wait()` must be **atomic** to prevent the "lost signal" problem (missed notify between check and wait).

```java
// WRONG — lost signal possible
if (queue.isEmpty()) {
    // notify() from another thread can happen HERE before wait()
    queue.wait(); // may wait forever — missed the signal
}

// CORRECT — check and wait are atomic under the same lock
synchronized (queue) {
    while (queue.isEmpty()) { // 'while', not 'if' — recheck after spurious wakeup
        queue.wait();
    }
    process(queue.poll());
}
```

**Why `while` instead of `if`?** Spurious wakeups — `wait()` can return without being notified. Always recheck the condition.

---

### Why are `wait()`, `notify()`, and `notifyAll()` defined in `Object`, not `Thread`?

**Core Explanation:**

These methods are about **monitor ownership**, not thread identity. Any object can be used as a lock in Java. Since `synchronized(anyObject)` is valid, `wait()` and `notify()` must be on `Object` to work with any lock.

If they were on `Thread`, you could only synchronize on thread objects, severely limiting locking granularity. The design allows any object to serve as a condition variable.

---

### What is the difference between `yield()` and `join()`?

**Core Explanation:**

- **`yield()`**: Hint to the scheduler that the current thread is willing to give up CPU time. The scheduler may ignore it. Thread goes from RUNNING to RUNNABLE (not WAITING).
- **`join()`**: Current thread waits until the target thread **completes**. Thread goes into WAITING state.

```java
Thread t = new Thread(() -> doWork());
t.start();
t.join(); // main thread waits until t finishes
System.out.println("t is done");

// yield — rarely needed in practice
Thread.yield(); // "I'm fine pausing, give others a turn"
```

---

### How do you safely stop a running thread (interrupt + flag pattern)?

**Core Explanation:**

**Wrong way:** `Thread.stop()` — deprecated, can corrupt data by stopping a thread mid-operation.

**Correct approach:** Cooperative interruption using `Thread.interrupt()` + checking `Thread.interrupted()` / `isInterrupted()`.

```java
class WorkerThread implements Runnable {
    private volatile boolean stop = false; // flag for non-blocking loops

    @Override
    public void run() {
        while (!stop && !Thread.currentThread().isInterrupted()) {
            try {
                doWork();
                Thread.sleep(100); // interruptible blocking operation
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); // restore interrupt flag
                break; // exit loop
            }
        }
        cleanup();
    }

    void stop() { stop = true; } // signal to stop
}

WorkerThread worker = new WorkerThread();
Thread t = new Thread(worker);
t.start();
// To stop:
worker.stop();
t.interrupt(); // wake up if blocked in sleep/wait/IO
```

---

## Advanced Questions

---

### What is the happens-before relationship in the Java Memory Model? Give three examples.

**Core Explanation:**

**Happens-before (HB)** is a guarantee from the Java Memory Model (JMM) that if action A happens-before action B, then A's effects (writes) are visible to B. Without HB, threads may see stale/cached values.

**Three examples:**
1. **Monitor lock**: Release of a `synchronized` lock HB before any subsequent acquisition of the same lock.
2. **`volatile` write**: A write to a `volatile` variable HB before every subsequent read of that variable.
3. **Thread start**: `thread.start()` HB before any action in the started thread.

```java
// Example 1: Monitor
int data = 0;
boolean ready = false;

synchronized (lock) {
    data = 42;
    ready = true;
} // lock release — HB before...

synchronized (lock) { // lock acquire by another thread
    if (ready) assert data == 42; // guaranteed visible
}

// Example 2: volatile
volatile boolean initialized = false;
String config = null;

// Writer thread:
config = "jdbc://localhost"; // write before volatile
initialized = true;          // volatile write — HB for all subsequent reads

// Reader thread:
if (initialized) {           // volatile read
    assert config != null;   // guaranteed — HB from volatile write
}
```

---

### Why is Double-Checked Locking broken without `volatile`? Explain the memory visibility issue.

**Core Explanation:**

In the classic double-checked locking pattern, `instance = new Singleton()` is **not atomic**. The JVM can reorder it to:
1. Allocate memory.
2. **Write reference to `instance`** (now non-null).
3. Call constructor.

If a second thread reads `instance` between steps 2 and 3, it gets a non-null but **incompletely constructed** object.

`volatile` prevents this reordering — a `volatile` write establishes a happens-before, so the constructor must complete before the reference is visible.

```java
// BROKEN without volatile
public class Singleton {
    private static Singleton instance; // NOT volatile
    public static Singleton getInstance() {
        if (instance == null) { // Thread B sees non-null here, but constructor not done!
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // steps may be reordered
                }
            }
        }
        return instance; // Thread B returns partially constructed instance
    }
}

// FIXED with volatile
public class Singleton {
    private static volatile Singleton instance; // prevents reordering
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // constructor completes before reference is visible
                }
            }
        }
        return instance;
    }
}
```

---

### What is instruction reordering? How does the JVM/CPU reorder instructions and why is it dangerous?

**Core Explanation:**

Both the **JIT compiler** and the **CPU** reorder instructions for performance — as long as the effect within a **single thread** is the same. However, reordering can cause other threads to observe operations in unexpected orders.

```java
int a = 0, b = 0; // not volatile
boolean flag = false;

// Thread 1 (may be reordered by JIT/CPU):
a = 1;
flag = true; // CPU might execute this before "a = 1"

// Thread 2:
if (flag) {
    assert a == 1; // may FAIL — Thread 2 sees flag=true but a=0 due to reordering
}
```

**Prevention:**
- `volatile` fields create memory barriers preventing reordering around them.
- `synchronized` blocks have acquire/release semantics (memory barriers).
- `java.util.concurrent.atomic` classes use memory barriers internally.

---

### How does Java handle biased, lightweight, and heavyweight locks internally?

**Core Explanation:**

Java's HotSpot JVM uses **adaptive locking** to optimize the common case:

1. **Biased locking**: If only one thread ever accesses the object, the lock is "biased" toward that thread. Subsequent acquisitions require no CAS — just a check that the thread ID matches. Zero synchronization cost for the common case.

2. **Lightweight lock (thin lock)**: When a second thread tries to acquire, biased locking is revoked and lightweight locking begins. Uses CAS to set a pointer in the object header to the acquiring thread's stack frame.

3. **Heavyweight lock (inflated lock)**: If CAS fails (lock is contended), the lock inflates to a heavyweight monitor with OS mutex. Thread blocks in the OS — context switch overhead.

**Note:** Biased locking was deprecated in Java 15 and removed in Java 21 — modern workloads with thread pools (threads change frequently) made it more costly than beneficial.

---

### What is the cost of context switching? How does it affect throughput?

**Core Explanation:**

A **context switch** saves the current thread's state (registers, program counter, stack pointer) and restores another thread's state. Cost includes:
- CPU time to save/restore registers (~100ns–10μs).
- Cache invalidation — new thread's data evicts cached data.
- TLB (Translation Lookaside Buffer) flushes.

**Effect on throughput:** With too many threads, the system spends more time context-switching than doing actual work ("thrashing"). This is why thread pool size should approximate CPU cores for CPU-bound work, not match the number of tasks.

**Rule of thumb:**
- CPU-bound: threads ≈ number of CPU cores.
- I/O-bound: threads = cores × (1 + wait_time / compute_time).

---

### What is `ThreadLocal`? How can it cause memory leaks in thread pools and servlet containers?

**Core Explanation:**

`ThreadLocal` maps the current thread to a value. In thread pools (like Tomcat's thread pool), threads are **reused** across requests. If a `ThreadLocal` value is set but never removed, the value lives on in the thread indefinitely — even after the request completes.

**Memory leak scenario:**
```java
// In a web filter
RequestContext.setUser(user); // stores User in ThreadLocal
chain.doFilter(req, res);
// If this throws and you don't clean up:
// The User object stays in ThreadLocal of the thread pool thread
// Next request gets the wrong user!
// After thousands of requests: thousands of User objects held in memory

// CORRECT — always use try/finally
try {
    RequestContext.setUser(user);
    chain.doFilter(req, res);
} finally {
    RequestContext.clear(); // MUST remove to prevent leak
}
```

**ClassLoader leak in app servers:** If a ThreadLocal holds a reference to a class loaded by a web app's ClassLoader, undeploying the app doesn't GC the ClassLoader because the ThreadLocal (in the thread pool thread) still holds a reference.

---

### What happens when a thread pool queue is full? How do rejection policies (`AbortPolicy`, `CallerRunsPolicy`) work?

**Core Explanation:**

When the queue is full AND all threads are busy, `ThreadPoolExecutor` invokes its **RejectionHandler**:

| Policy | Behavior |
|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | The **submitting thread** runs the task — provides back-pressure |
| `DiscardPolicy` | Silently drops the task |
| `DiscardOldestPolicy` | Drops the oldest waiting task, retries submission |

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2, 4, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(10),
    new ThreadPoolExecutor.CallerRunsPolicy() // back-pressure mechanism
);

// CallerRunsPolicy is useful: when the queue is full,
// the HTTP request-handling thread runs the task itself
// → Tomcat can't accept new requests → natural rate limiting
```

---

### What happens if a thread dies while holding a lock?

**Core Explanation:**

With `synchronized`: The JVM automatically releases the monitor when the thread terminates (due to exception or normal completion). Other waiting threads can then acquire the lock. However, the protected data may be in an **inconsistent state** if the thread died mid-operation.

With `ReentrantLock`: Same — when the thread terminates, the lock is released if the thread held it (as part of thread cleanup). However, you should use `try/finally` to release explicitly — if the thread throws without releasing, other threads unblock but may get corrupt state.

**Real danger:** Corrupt shared state, not permanent blocking. Always protect critical sections against partial failures with defensive coding.

---

### What is the difference between `LockSupport.park()/unpark()` and `wait()/notify()`?

**Core Explanation:**

| | `LockSupport.park()/unpark()` | `wait()/notify()` |
|---|---|---|
| Object required | No — thread-level | Yes — must own the object monitor |
| `synchronized` required | No | Yes |
| Spurious wakeup handling | Same — recheck condition | Same |
| Permit system | Has permit (unpark before park = no block) | No permit — notify before wait = lost |
| Usage | Low-level concurrency framework building | Higher-level producer-consumer |

`LockSupport` is used internally by `AbstractQueuedSynchronizer` (which backs `ReentrantLock`, `CountDownLatch`, etc.).

---

### What is over-synchronization? How does it reduce throughput?

**Core Explanation:**

Over-synchronization is locking more code, more objects, or more frequently than necessary.

**Problems:**
1. **Contention**: Threads line up waiting for the same lock → serial execution.
2. **Scalability collapse**: More threads = more contention = lower throughput (Amdahl's Law).
3. **Liveness issues**: More locks = higher deadlock risk.

```java
// Over-synchronized — locks the entire method even for reads
public synchronized String getConfig(String key) {
    return configs.get(key); // reads don't need synchronization if ConcurrentHashMap
}

// Better — use ConcurrentHashMap (lock-free reads)
private ConcurrentHashMap<String, String> configs = new ConcurrentHashMap<>();
public String getConfig(String key) {
    return configs.get(key); // no synchronized needed
}
```

---

### How do you detect deadlocks in production (jstack, `ThreadMXBean`)?

**Core Explanation:**

**1. `jstack` — thread dump:**
```bash
jstack <PID> | grep -A 30 "deadlock"
# Shows "Found one Java-level deadlock" with involved threads and locks
```

**2. `ThreadMXBean` — programmatic:**
```java
ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreadIds = mxBean.findDeadlockedThreads();
if (deadlockedThreadIds != null) {
    ThreadInfo[] infos = mxBean.getThreadInfo(deadlockedThreadIds, true, true);
    for (ThreadInfo info : infos) {
        log.error("Deadlocked thread: {}\nStack: {}",
            info.getThreadName(),
            Arrays.toString(info.getStackTrace()));
    }
}
```

**3. Scheduled detection:**
```java
ScheduledExecutorService monitor = Executors.newSingleThreadScheduledExecutor();
monitor.scheduleAtFixedRate(() -> {
    long[] deadlocked = mxBean.findDeadlockedThreads();
    if (deadlocked != null) alertOps("Deadlock detected in " + deadlocked.length + " threads");
}, 0, 30, TimeUnit.SECONDS);
```

---

### How do you design a high-performance, thread-safe counter without `synchronized`?

**Core Explanation:**

```java
// Option 1: AtomicLong — good for moderate contention
AtomicLong counter = new AtomicLong(0);
counter.incrementAndGet(); // CAS-based, fast, lock-free

// Option 2: LongAdder — best for high contention (Java 8+)
// Uses striped counting (multiple cells) to reduce CAS contention
LongAdder adder = new LongAdder();
adder.increment(); // Thread 1 updates cell 0
adder.increment(); // Thread 2 updates cell 1 (no contention)
long total = adder.sum(); // sums all cells

// Benchmark: LongAdder >> AtomicLong >> synchronized under high contention
// LongAdder tradeoff: sum() is approximate — cells may be inconsistent at read time

// Option 3: ThreadLocal + merge — for absolute minimal contention
ThreadLocal<Long> localCount = ThreadLocal.withInitial(() -> 0L);
localCount.set(localCount.get() + 1);
// Merge at the end from all threads
```

---

### How do you ensure safe publication of shared objects?

**Core Explanation:**

**Safe publication** ensures that when an object is published (made visible to other threads), its fully constructed state is visible. Unsafe publication can expose partially initialized objects.

**Safe publication mechanisms:**
1. **`static final` fields**: Initialized once by class loading — always safe.
2. **`volatile` fields**: Write is visible to all threads.
3. **Atomic references**: `AtomicReference` guarantees visibility.
4. **`synchronized` blocks**: Release ensures visibility.
5. **Concurrent collections**: Objects placed in `ConcurrentHashMap` are safely published.
6. **`final` fields**: JMM guarantees final fields are fully initialized before the reference escapes.

```java
// UNSAFE — reference may escape before construction
class UnsafePublication {
    int value;
    static UnsafePublication instance;
    UnsafePublication() {
        instance = this; // BAD — 'this' escapes before constructor completes!
        value = 42;
    }
}

// SAFE — use static factory + volatile
class SafeSingleton {
    private final int value;
    private static volatile SafeSingleton instance;
    private SafeSingleton() { this.value = 42; }
    static SafeSingleton getInstance() {
        if (instance == null) {
            synchronized (SafeSingleton.class) {
                if (instance == null) instance = new SafeSingleton();
            }
        }
        return instance;
    }
}
```

---

### What is false sharing and how does it impact multithreaded performance on multicore CPUs?

**Core Explanation:**

Modern CPUs transfer memory in **cache lines** (~64 bytes). **False sharing** occurs when two threads modify different variables that happen to reside on the **same cache line**. Each write invalidates the entire cache line for all other CPUs, forcing them to reload it — even though they access different variables.

```java
// False sharing — 'a' and 'b' are adjacent in memory (same cache line)
class Counters {
    volatile long a = 0; // bytes 0-7
    volatile long b = 0; // bytes 8-15 — on the same 64-byte cache line
}

// Thread 1 writes 'a' → invalidates cache line on all CPUs
// Thread 2 writes 'b' → also invalidates → ping-pong between CPUs
// Result: terrible performance even though threads don't share state logically

// Fix: padding to separate cache lines
class PaddedCounters {
    volatile long a = 0;
    long p1, p2, p3, p4, p5, p6, p7; // padding to fill 64 bytes
    volatile long b = 0;
}

// Modern Java: @Contended (JVM flag: -XX:-RestrictContended)
@jdk.internal.vm.annotation.Contended
class Counter { volatile long value; }
```

---

### How do virtual threads (Java 21) change the concurrency model? Do they eliminate the need for `synchronized`?

**Core Explanation:**

Virtual threads are **lightweight, JVM-managed threads** (not OS threads). The JVM mounts virtual threads on OS "carrier" threads and unmounts them when they block. This allows millions of virtual threads without OS memory overhead.

**Impact:** Blocking I/O becomes cheap — a virtual thread that blocks on a socket unmounts from the carrier thread, which then picks up another virtual thread.

**Do they eliminate `synchronized`?** **No.** If a virtual thread blocks inside a `synchronized` block, it **pins** the carrier thread — the carrier cannot pick up another virtual thread while pinned. This defeats the concurrency benefit.

```java
// Problem: synchronized pins carrier thread
synchronized (lock) {
    // Virtual thread blocks here → carrier thread is PINNED → no other VT can run on it
    socket.read(buffer); // blocking I/O while holding lock
}

// Solution: use ReentrantLock (unmounts cleanly)
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    socket.read(buffer); // virtual thread unmounts → carrier is free
} finally {
    lock.unlock();
}
```

**Best practices for virtual threads:**
- Use `ReentrantLock` instead of `synchronized` for I/O-bound critical sections.
- One-thread-per-request model returns — simple blocking code can replace reactive code.
- Thread pools for CPU-bound work still needed.

---

### How do you size a thread pool? Explain the difference between CPU-bound and IO-bound tuning formulas.

**Core Explanation:**

**CPU-bound tasks** (pure computation, no waiting):
```
Optimal threads = Number of CPU cores (possibly + 1)
```
Adding more threads causes context switching overhead with no throughput gain.

**I/O-bound tasks** (HTTP calls, DB queries, file I/O):
```
Optimal threads = Number of CPU cores × (1 + Wait Time / Service Time)
```
Example: If each task spends 90% of its time waiting for DB → `Wait/Service = 9`
→ `threads = 8 × (1 + 9) = 80`

**Little's Law approach:**
```
Threads = Throughput × Latency
```
If you need 100 RPS and each request takes 200ms → `100 × 0.2 = 20 threads`

**Practical example:**
```java
int cpuCores = Runtime.getRuntime().availableProcessors();

// CPU-bound pool
ExecutorService cpuPool = Executors.newFixedThreadPool(cpuCores);

// IO-bound pool — empirically tune, start with 2× or 4× cores
ExecutorService ioPool = Executors.newFixedThreadPool(cpuCores * 4);

// Or use the formula with known wait ratios
double waitRatio = 0.8; // 80% of time waiting
int ioThreads = (int)(cpuCores * (1 + waitRatio / (1 - waitRatio)));
```

**Monitor and adjust:** Start with a reasonable estimate, measure thread pool queue depth and CPU utilization, then tune. Queue depth > 0 consistently → add threads. CPU < 50% with many threads → reduce threads (you're context-switching too much).
# Section 7: Spring Boot

---

## Basic Questions

---

### What is Spring Boot? Why is it preferred over the traditional Spring framework?

**Core Explanation:**

Spring Boot is an opinionated extension of the Spring Framework that eliminates boilerplate configuration and provides production-ready features out of the box. It uses:
- **Auto-configuration**: Configures beans automatically based on classpath and properties.
- **Embedded servers**: Tomcat, Jetty, or Undertow embedded — no WAR deployment needed.
- **Starter dependencies**: Single dependency pulls in everything needed for a feature.
- **Actuator**: Built-in health checks, metrics, and monitoring.
- **Spring Initializr**: Project generation without manual setup.

**Traditional Spring pain points removed:**
- No `web.xml`, no `applicationContext.xml`.
- No manual `DispatcherServlet` registration.
- No explicit DataSource, TransactionManager, JPA setup — all auto-configured.

---

### What is `@SpringBootApplication`? What three annotations does it combine?

**Core Explanation:**

`@SpringBootApplication` is a composed annotation that combines:

1. **`@SpringBootConfiguration`** (extends `@Configuration`): Marks as a configuration class, allowing `@Bean` definitions.
2. **`@EnableAutoConfiguration`**: Triggers Spring Boot's auto-configuration mechanism.
3. **`@ComponentScan`**: Scans the package (and sub-packages) for `@Component`, `@Service`, `@Repository`, `@Controller`, etc.

```java
@SpringBootApplication // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// Equivalent to:
@Configuration
@EnableAutoConfiguration
@ComponentScan(basePackages = "com.example")
public class OrderServiceApplication { ... }
```

---

### What is Dependency Injection? What are the types (constructor, setter, field)?

**Core Explanation:**

Dependency Injection (DI) is a design pattern where an object's dependencies are provided externally rather than created by the object itself. Spring's IoC container manages object creation and wiring.

```java
@Service
public class OrderService {

    // 1. Constructor injection (RECOMMENDED)
    private final OrderRepository repository;
    private final EmailService emailService;

    @Autowired // optional with single constructor in Spring 4.3+
    public OrderService(OrderRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }

    // 2. Setter injection (optional dependencies, circular dep workaround)
    private NotificationService notificationService;
    @Autowired(required = false)
    public void setNotificationService(NotificationService service) {
        this.notificationService = service;
    }

    // 3. Field injection (NOT recommended for production)
    @Autowired
    private AuditService auditService; // hard to test, breaks immutability
}
```

**Best practice:** Always use constructor injection for required dependencies — it enables immutability (`final` fields), makes dependencies explicit, and simplifies testing without Spring context.

---

### What is the Spring IoC Container?

**Core Explanation:**

The IoC (Inversion of Control) container is the core of Spring — it creates, configures, wires, and manages the lifecycle of beans. The container reads configuration metadata (annotations, XML, Java config) and assembles the application object graph.

Two primary implementations:
- **`BeanFactory`**: Basic container — lazy initialization, minimum features.
- **`ApplicationContext`**: Extended container — adds event publishing, AOP, internationalization, eager initialization. Always use `ApplicationContext` in practice.

```java
// Accessing the container
ApplicationContext context = SpringApplication.run(App.class, args);
OrderService orderService = context.getBean(OrderService.class);
String[] beanNames = context.getBeanDefinitionNames(); // list all beans
```

---

### What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?

**Core Explanation:**

All four are specializations of `@Component` — they all enable component scanning. The differences are **semantic** and provide **additional behavior**:

| Annotation | Layer | Additional behavior |
|---|---|---|
| `@Component` | Generic | None — base annotation |
| `@Service` | Service/business | None — signals intent |
| `@Repository` | Persistence | Enables Spring's exception translation (converts `SQLException` → `DataAccessException`) |
| `@Controller` | Web layer | Enables request mapping; handles web MVC |
| `@RestController` | Web (REST) | `@Controller` + `@ResponseBody` |

```java
@Repository // enables exception translation
public class UserRepository {
    public User findById(Long id) { /* JDBC/JPA code */ }
    // If JPA throws PersistenceException, Spring wraps it in DataAccessException
}
```

---

### What is `@Autowired`? What is the difference between constructor injection and field injection?

**Core Explanation:**

`@Autowired` tells Spring to inject a dependency automatically by type. If there are multiple beans of the same type, combine with `@Qualifier` to disambiguate.

**Constructor vs Field injection:**

| | Constructor Injection | Field Injection |
|---|---|---|
| Fields | Can be `final` | Cannot be `final` |
| Testability | Easy — pass mocks via constructor | Requires reflection or Spring context |
| Circular deps | Detected at startup (fail fast) | Hidden until runtime |
| Code clarity | Dependencies are explicit | Hidden — class looks simpler but is misleading |
| Spring needed | Not needed for testing | Required |

```java
// Constructor — testable without Spring
@Service
public class PaymentService {
    private final PaymentGateway gateway;

    public PaymentService(PaymentGateway gateway) { this.gateway = gateway; }
    // Test: new PaymentService(new MockGateway())
}

// Field — test requires Spring or reflection
@Service
public class PaymentService {
    @Autowired private PaymentGateway gateway; // BAD for production code
}
```

---

### What is `@Qualifier` and when do you need it?

**Core Explanation:**

When multiple beans of the same type exist, Spring can't determine which to inject — it throws `NoUniqueBeanDefinitionException`. `@Qualifier` specifies which bean to inject by name.

```java
@Configuration
public class PaymentConfig {
    @Bean("stripeGateway")
    PaymentGateway stripeGateway() { return new StripeGateway(); }

    @Bean("paypalGateway")
    PaymentGateway paypalGateway() { return new PaypalGateway(); }
}

@Service
public class OrderService {
    private final PaymentGateway gateway;

    public OrderService(@Qualifier("stripeGateway") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}

// Alternative: @Primary — marks one bean as the default choice
@Bean @Primary
PaymentGateway defaultGateway() { return new StripeGateway(); }
```

---

### What are bean scopes? Name the five scopes in Spring.

**Core Explanation:**

Bean scope determines how many instances Spring creates and how they're shared:

| Scope | Instances | Availability |
|---|---|---|
| **Singleton** (default) | One per container | All requests |
| **Prototype** | New instance per request | Any context |
| **Request** | One per HTTP request | Web context only |
| **Session** | One per HTTP session | Web context only |
| **Application** | One per `ServletContext` | Web context only |

```java
@Component @Scope("singleton") // default — one instance
class ConfigService { }

@Component @Scope("prototype") // new instance each time
class ReportGenerator { }

@Component @Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
class RequestData { } // one per HTTP request

@Component @Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
class UserSession { } // one per HTTP session
```

**Pitfall:** Injecting a `prototype` bean into a `singleton` — the prototype gets created once and reused (defeating the purpose). Fix: use `ObjectProvider<T>` or `@Lookup` method injection.

---

### What is `application.properties` / `application.yml`? What is it used for?

**Core Explanation:**

These files hold externalized configuration for the Spring Boot application. Spring Boot auto-reads them and maps values to `@Value` fields or `@ConfigurationProperties` classes.

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: app_user
    password: ${DB_PASSWORD} # env var override
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

app:
  features:
    email-notifications: true
  payment:
    stripe-key: ${STRIPE_KEY}
    timeout-seconds: 30
```

---

### What is `@RestController` vs `@Controller`?

**Core Explanation:**

- `@Controller`: Spring MVC controller. Methods return view names (templates) by default. Need `@ResponseBody` to return JSON/data.
- `@RestController`: `@Controller` + `@ResponseBody`. Every method returns data directly serialized to the response body (JSON by default).

```java
@Controller
public class PageController {
    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("name", "World");
        return "home"; // resolves to src/main/resources/templates/home.html
    }
}

@RestController
@RequestMapping("/api/v1")
public class OrderController {
    @GetMapping("/orders/{id}")
    public OrderDTO getOrder(@PathVariable Long id) {
        return orderService.findById(id); // serialized to JSON automatically
    }
}
```

---

### What is `ResponseEntity`?

**Core Explanation:**

`ResponseEntity<T>` gives full control over the HTTP response: status code, headers, and body. Use it when you need to customize beyond the default 200 OK.

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<OrderDTO> createOrder(@RequestBody CreateOrderRequest request) {
        OrderDTO order = orderService.create(request);
        return ResponseEntity
            .status(HttpStatus.CREATED)                    // 201
            .header("Location", "/api/orders/" + order.getId())
            .body(order);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        orderService.delete(id);
        return ResponseEntity.noContent().build(); // 204
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderDTO> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build()); // 404
    }
}
```

---

### What is the difference between `@RequestParam`, `@RequestBody`, and `@PathVariable`?

**Core Explanation:**

| Annotation | Source | Usage |
|---|---|---|
| `@PathVariable` | URL path segment | `/orders/{id}` |
| `@RequestParam` | URL query parameter | `/orders?status=PENDING` |
| `@RequestBody` | Request body (JSON) | POST/PUT payloads |

```java
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long id) { ... }
// GET /orders/123 → id=123

@GetMapping("/orders")
public List<OrderDTO> listOrders(
    @RequestParam(defaultValue = "PENDING") String status,
    @RequestParam(defaultValue = "0") int page) { ... }
// GET /orders?status=SHIPPED&page=2

@PostMapping("/orders")
public OrderDTO createOrder(@RequestBody @Valid CreateOrderRequest request) { ... }
// POST /orders  Body: {"productId": 5, "quantity": 2}
```

---

## Intermediate Questions

---

### How does Spring Boot auto-configuration work internally? What role does `@Conditional` play?

**Core Explanation:**

Spring Boot auto-configuration works by registering `@Configuration` classes that apply conditionally based on classpath, existing beans, and properties.

**Mechanism:**
1. Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 2.7+) for auto-configuration class names.
2. Each auto-configuration class is annotated with `@Conditional` variants that control whether it applies.
3. If conditions are met, its `@Bean` methods are executed.

```java
// Example: Spring Boot's DataSourceAutoConfiguration
@AutoConfiguration
@ConditionalOnClass(DataSource.class)          // only if DataSource is on classpath
@ConditionalOnMissingBean(DataSource.class)    // only if no DataSource bean defined yet
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnProperty(name = "spring.datasource.url")
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}

// Common @Conditional variants:
// @ConditionalOnClass(name = "com.mysql.cj.jdbc.Driver") — class on classpath
// @ConditionalOnMissingBean(DataSource.class) — no bean of that type exists
// @ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
// @ConditionalOnExpression("${feature.enabled:true}")
// @ConditionalOnWebApplication — only in web context
```

---

### What is the role of `spring.factories` / `AutoConfiguration.imports` in auto-configuration?

**Core Explanation:**

Spring Boot must discover auto-configuration classes without scanning all jars (too slow). It uses a service-loader pattern:

- **Spring Boot 2.6 and earlier**: `META-INF/spring.factories` lists `EnableAutoConfiguration` key with class names.
- **Spring Boot 2.7+**: `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` — one class per line.

```properties
# META-INF/spring.factories (old format)
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration,\
  com.example.AnotherAutoConfiguration

# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports (new)
com.example.MyAutoConfiguration
com.example.AnotherAutoConfiguration
```

This is how Spring Boot Starters work — each starter jar includes these files, and Spring Boot reads them at startup to register their auto-configurations.

---

### What happens internally when a Spring Boot application starts (bootstrap sequence)?

**Core Explanation:**

1. **`SpringApplication.run()`** creates a `SpringApplication` instance.
2. Determines application type (SERVLET, REACTIVE, NONE).
3. Loads `ApplicationContextInitializer`s and `ApplicationListener`s from `spring.factories`.
4. Creates the `ApplicationContext` (e.g., `AnnotationConfigServletWebServerApplicationContext`).
5. **Prepares context**: Sets environment, applies initializers.
6. **Refreshes context** (`AbstractApplicationContext.refresh()`):
   - Scans for beans (`@ComponentScan`).
   - Processes auto-configuration.
   - Creates and wires all singleton beans.
   - Runs `BeanFactoryPostProcessor`s and `BeanPostProcessor`s.
   - Calls `@PostConstruct` methods.
7. **Starts embedded web server** (Tomcat, Jetty).
8. Calls `ApplicationRunner` and `CommandLineRunner` beans.
9. Publishes `ApplicationStartedEvent`.

---

### What is the difference between `@Bean` and `@Component`?

**Core Explanation:**

| | `@Bean` | `@Component` |
|---|---|---|
| Location | Inside `@Configuration` classes | Directly on a class |
| Control | You write the factory method | Spring instantiates automatically |
| Third-party classes | Yes — you can create beans for classes you don't own | No — you must annotate the class |
| Conditional logic | Easy — write regular Java in the method | Requires `@Conditional` |

```java
// @Component — you own the class
@Component
public class UserService { }

// @Bean — for classes you don't own (e.g., third-party libraries)
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            .registerModule(new JavaTimeModule());
    }

    @Bean
    @ConditionalOnProperty("feature.audit")
    public AuditService auditService(AuditRepository repo) {
        return new AuditService(repo, "prod-audit");
    }
}
```

---

### What is the difference between `@Configuration` and `@Component` (full vs lite mode)?

**Core Explanation:**

- **`@Configuration` (full mode)**: Class is proxied by CGLIB. `@Bean` methods are intercepted — calling one `@Bean` method from another returns the same singleton bean, not a new instance.
- **`@Component` (lite mode)**: Class is NOT proxied. `@Bean` methods behave like regular Java methods — each call creates a new object.

```java
@Configuration // proxied by CGLIB
public class FullConfig {
    @Bean
    public ServiceA serviceA() { return new ServiceA(sharedDep()); }

    @Bean
    public ServiceB serviceB() { return new ServiceB(sharedDep()); }

    @Bean
    public SharedDep sharedDep() { return new SharedDep(); } // called twice above
    // With @Configuration: SAME instance returned both times (singleton)
}

@Component // NOT proxied
public class LiteConfig {
    @Bean
    public ServiceA serviceA() { return new ServiceA(sharedDep()); }

    @Bean
    public SharedDep sharedDep() { return new SharedDep(); }
    // DIFFERENT instances — regular Java method call, not intercepted!
}
```

---

### What is the difference between `ApplicationContext` and `BeanFactory`?

**Core Explanation:**

| Feature | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| Bean creation | Lazy by default | Eager (singletons at startup) |
| AOP | Manual setup | Integrated |
| Event publishing | No | `ApplicationEvent` system |
| Internationalization | No | MessageSource |
| Environment/profiles | No | Full support |
| Use in production | Almost never | Always |

`ApplicationContext` extends `BeanFactory` with many enterprise features. `BeanFactory` is used internally and in testing with minimal bootstrapping.

---

### What is the lifecycle of a Spring Bean (`@PostConstruct` -> initialization -> `@PreDestroy`)?

**Core Explanation:**

```
1. Instantiation (constructor)
2. Dependency injection (setters / @Autowired fields)
3. Aware interfaces (BeanNameAware, ApplicationContextAware...)
4. BeanPostProcessor.postProcessBeforeInitialization()
5. @PostConstruct method
6. InitializingBean.afterPropertiesSet()
7. @Bean(initMethod = "init")
8. BeanPostProcessor.postProcessAfterInitialization()
--- Bean is ready for use ---
... application runs ...
9. @PreDestroy method
10. DisposableBean.destroy()
11. @Bean(destroyMethod = "cleanup")
```

```java
@Component
public class ConnectionPool {
    private DataSource dataSource;

    @Autowired
    public ConnectionPool(DataSource ds) { this.dataSource = ds; } // step 1-2

    @PostConstruct
    public void init() {
        System.out.println("Pool initialized with: " + dataSource.getUrl()); // step 5
        // Good for: validation, creating connections, loading cache
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Closing all connections"); // step 9
        // Good for: releasing resources, flushing buffers
    }
}
```

---

### What is the difference between Singleton and Prototype beans? What happens when you inject a Prototype into a Singleton?

**Core Explanation:**

- **Singleton**: One instance per container. Created at startup, shared everywhere.
- **Prototype**: New instance created every time it's requested from the container.

**Injecting Prototype into Singleton problem:**
When a Singleton bean has `@Autowired` Prototype, Spring injects ONE instance of the Prototype at startup — and reuses it forever. The Prototype behaves like a Singleton!

```java
@Component @Scope("prototype")
class ReportGenerator { private final UUID id = UUID.randomUUID(); }

@Component
class ReportService {
    @Autowired
    private ReportGenerator generator; // WRONG — always the same instance!

    void generate() {
        System.out.println(generator.getId()); // same ID every time
    }
}

// Solutions:
// 1. ObjectProvider
@Component
class ReportService {
    @Autowired
    private ObjectProvider<ReportGenerator> generatorProvider;

    void generate() {
        ReportGenerator generator = generatorProvider.getObject(); // new each time
    }
}

// 2. @Lookup method injection
@Component
class ReportService {
    @Lookup
    protected ReportGenerator createGenerator() { return null; } // Spring overrides

    void generate() {
        ReportGenerator generator = createGenerator(); // new instance each call
    }
}
```

---

### How does Spring handle circular dependencies? What are the strategies to resolve them?

**Core Explanation:**

Spring handles circular dependencies for **setter and field injection** of singleton beans by using a three-phase cache:
1. Fully created beans.
2. Early references (created but not fully initialized).
3. Bean factories.

For **constructor injection** circular dependencies, Spring **cannot** resolve them — it throws `BeanCurrentlyInCreationException`.

**Resolution strategies:**
```java
// 1. Refactor to break the cycle (best)
// A → B, B → A becomes A → C, B → C (extract shared logic)

// 2. @Lazy — inject a proxy, real bean created on first use
@Service
public class ServiceA {
    @Autowired @Lazy
    private ServiceB serviceB; // proxy injected, real ServiceB created on first call
}

// 3. Setter injection (allows Spring's early reference mechanism)
@Service
public class ServiceA {
    private ServiceB serviceB;
    @Autowired
    public void setServiceB(ServiceB b) { this.serviceB = b; }
}

// 4. ApplicationContext.getBean() (pull instead of inject)
@Service
public class ServiceA {
    @Autowired ApplicationContext context;
    private ServiceB getServiceB() { return context.getBean(ServiceB.class); }
}
```

**Note:** Spring Boot 2.6+ disallows circular dependencies by default. Set `spring.main.allow-circular-references=true` to enable (not recommended).

---

### What is `@Value` vs `@ConfigurationProperties`? When should you use each?

**Core Explanation:**

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Binding | Single value | Entire prefix group |
| Type safety | Limited | Full (with nested objects, lists) |
| Validation | No built-in | `@Validated` supported |
| IDE support | Limited | Excellent (metadata generation) |
| Testing | Harder | Easier (`@TestPropertySource`) |

```java
// @Value — fine for one or two properties
@Component
public class StripeConfig {
    @Value("${stripe.api.key}")
    private String apiKey;

    @Value("${stripe.timeout:30}")  // default = 30
    private int timeoutSeconds;
}

// @ConfigurationProperties — preferred for groups of related config
@Component
@ConfigurationProperties(prefix = "app.payment")
@Validated
public class PaymentConfig {
    @NotBlank private String stripeKey;
    @Min(5) @Max(120) private int timeoutSeconds = 30;
    private Retry retry = new Retry();

    public static class Retry {
        private int maxAttempts = 3;
        private Duration backoff = Duration.ofSeconds(1);
    }
    // getters/setters
}
```

```yaml
# application.yml
app:
  payment:
    stripe-key: sk_live_...
    timeout-seconds: 45
    retry:
      max-attempts: 5
      backoff: 2s
```

---

### What is `DispatcherServlet`? Describe the full request lifecycle in Spring MVC.

**Core Explanation:**

`DispatcherServlet` is the front controller in Spring MVC — all HTTP requests pass through it.

**Request lifecycle:**
1. Request arrives at **DispatcherServlet**.
2. `HandlerMapping` finds the right controller method (based on URL, method).
3. `HandlerAdapter` calls the controller method (wraps it for different controller types).
4. **Controller method** executes business logic.
5. Returns `ModelAndView` or `@ResponseBody`.
6. If `@ResponseBody`: `HttpMessageConverter` serializes to JSON/XML.
7. If view name: `ViewResolver` resolves to a template (Thymeleaf, JSP).
8. **`HandlerInterceptor`** `postHandle()` and `afterCompletion()` run.
9. Response sent to client.

**Interceptors and Filters in the lifecycle:**
- **Filters** (`javax.servlet.Filter`): Before/after DispatcherServlet — outside Spring context.
- **Interceptors** (`HandlerInterceptor`): After DispatcherServlet dispatches — inside Spring context.

---

### How do you implement global exception handling with `@ControllerAdvice` and `@ExceptionHandler`?

**Core Explanation:**

```java
@RestControllerAdvice // @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleOrderNotFound(OrderNotFoundException ex) {
        return new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("VALIDATION_FAILED", errors.toString()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception for {}: {}", req.getRequestURI(), ex.getMessage(), ex);
        return ResponseEntity.internalServerError()
            .body(new ErrorResponse("INTERNAL_ERROR", "Unexpected error"));
    }
}

record ErrorResponse(String code, String message) {}
```

---

### What is the difference between Filters and Interceptors? When do you use each?

**Core Explanation:**

| | Filter | Interceptor |
|---|---|---|
| Standard | Servlet spec | Spring MVC |
| Access to Spring beans | Limited (must use `DelegatingFilterProxy`) | Full |
| Called | Before/after DispatcherServlet | After DispatcherServlet dispatches |
| Use for | Auth tokens, CORS, request logging, compression | Logging with Spring context, AOP-style pre/post processing |

```java
// Filter — for cross-cutting at the Servlet level
@Component
public class RequestLoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        long start = System.currentTimeMillis();
        chain.doFilter(req, res); // proceed
        log.info("Request took {}ms", System.currentTimeMillis() - start);
    }
}

// Interceptor — for Spring MVC cross-cutting
@Component
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) {
        String token = request.getHeader("Authorization");
        if (!tokenValidator.isValid(token)) {
            response.setStatus(401);
            return false; // stop processing
        }
        return true;
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor).addPathPatterns("/api/**");
    }
}
```

---

### How does Spring Boot handle profile-specific configuration?

**Core Explanation:**

```yaml
# application.yml — common config
spring:
  application:
    name: order-service

---
# Profile: dev
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb

---
# Profile: prod
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-db:5432/orders
```

```java
// Activate with: SPRING_PROFILES_ACTIVE=prod java -jar app.jar
// Or in properties: spring.profiles.active=prod

// Profile-specific beans
@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public MetricsCollector metricsCollector() { return new DatadogMetrics(); }
}

@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public MetricsCollector metricsCollector() { return new InMemoryMetrics(); }
}
```

---

### What is the priority order of configuration sources?

**Core Explanation:**

Spring Boot property priority (highest to lowest):
1. Command-line arguments: `--server.port=9090`
2. `SPRING_APPLICATION_JSON` environment variable
3. OS environment variables: `SERVER_PORT=9090`
4. Java System properties: `-Dserver.port=9090`
5. `application-{profile}.properties/yml`
6. `application.properties/yml`
7. `@PropertySource` annotations
8. Default properties

**Implication:** Environment variables override `application.yml` — critical for Kubernetes/Docker deployments.

---

### What is Spring Boot Actuator? Name five useful endpoints.

**Core Explanation:**

Spring Boot Actuator adds production-ready monitoring and management endpoints:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,loggers
  endpoint:
    health:
      show-details: when-authorized
```

**Five most useful endpoints:**
1. **`/actuator/health`**: Application health (UP/DOWN) with component details (DB, Redis, Kafka).
2. **`/actuator/metrics`**: JVM, HTTP, custom metrics. `/actuator/metrics/http.server.requests`.
3. **`/actuator/env`**: Shows all configuration properties (mask sensitive with `management.endpoint.env.keys-to-sanitize`).
4. **`/actuator/loggers`**: View and dynamically change log levels — no restart needed!
5. **`/actuator/threaddump`**: Current thread dump — diagnose deadlocks and performance issues.

---

### What is AOP in Spring? Explain `@Aspect`, `@Around`, `@Before`, `@After` with a logging example.

**Core Explanation:**

AOP (Aspect-Oriented Programming) separates cross-cutting concerns (logging, security, transactions) from business logic by intercepting method calls.

**Key terms:**
- **Aspect**: Class with cross-cutting logic (`@Aspect`).
- **Join Point**: Point where advice can be applied (method call).
- **Pointcut**: Expression selecting join points (`execution(* com.example.service.*.*(..))`).
- **Advice**: The action (`@Before`, `@After`, `@Around`).
- **Weaving**: Applying aspects — Spring does this via proxies at runtime.

```java
@Aspect
@Component
public class LoggingAspect {

    // Pointcut: all methods in any class in the service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    private void serviceMethods() {}

    @Before("serviceMethods()")
    public void logBefore(JoinPoint jp) {
        log.info("Calling: {}.{}", jp.getTarget().getClass().getSimpleName(),
                 jp.getSignature().getName());
    }

    @AfterReturning(pointcut = "serviceMethods()", returning = "result")
    public void logAfterReturn(Object result) {
        log.info("Returned: {}", result);
    }

    @AfterThrowing(pointcut = "serviceMethods()", throwing = "ex")
    public void logException(Exception ex) {
        log.error("Exception in service: {}", ex.getMessage());
    }

    @Around("serviceMethods()")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed(); // call actual method
            log.info("{} took {}ms", pjp.getSignature().getName(),
                     System.currentTimeMillis() - start);
            return result;
        } catch (Throwable t) {
            log.error("{} failed after {}ms", pjp.getSignature().getName(),
                      System.currentTimeMillis() - start);
            throw t;
        }
    }
}
```

---

## Advanced Questions

---

### Why does `@Transactional` not work for private methods or self-invocation? Explain the proxy mechanism.

**Core Explanation:**

Spring `@Transactional` works via **AOP proxies**. When you call `orderService.placeOrder()`, you actually call a CGLIB proxy's `placeOrder()`, which starts a transaction, then delegates to the real `OrderService.placeOrder()`.

**Private methods:** CGLIB cannot override private methods — they're not intercepted.

**Self-invocation:** When method A calls `this.methodB()` inside the same class, `this` refers to the real object — bypassing the proxy entirely. The transaction on `methodB` is not started.

```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(Order order) {
        saveOrder(order); // calls this.saveOrder() — BYPASSES PROXY
        // saveOrder's @Transactional is IGNORED
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void saveOrder(Order order) { // this annotation has NO EFFECT when called from processOrder()
        repository.save(order);
    }
}

// Fixes:
// 1. Inject self (Spring's self-injection via @Autowired)
@Service
public class OrderService {
    @Autowired private OrderService self; // proxy injected

    public void processOrder(Order order) {
        self.saveOrder(order); // through proxy — WORKS
    }
}

// 2. Extract to another @Service (cleanest)
// 3. Use AspectJ weaving instead of Spring proxy (compile-time)
```

---

### Why does `@Async` sometimes not run asynchronously? What common mistakes cause this?

**Core Explanation:**

`@Async` requires a proxy — same limitations as `@Transactional`:

1. **Self-invocation**: Calling `@Async` method from within the same class bypasses the proxy.
2. **Missing `@EnableAsync`**: Must annotate a `@Configuration` class with `@EnableAsync`.
3. **Private methods**: CGLIB can't intercept private methods.
4. **No `AsyncConfigurer`**: Without configuration, uses `SimpleAsyncTaskExecutor` (creates a new thread for every call — not a pool!).

```java
@Configuration
@EnableAsync // REQUIRED
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> log.error("Async error in {}: {}", method, ex.getMessage());
    }
}

@Service
public class EmailService {
    @Async // only works when called through Spring proxy (from another bean)
    public CompletableFuture<Void> sendEmail(String to, String subject) {
        // runs in async-N thread
        emailClient.send(to, subject, body);
        return CompletableFuture.completedFuture(null);
    }
}
```

---

### Why does `@Cacheable` sometimes not cache? What are the proxy-related gotchas?

**Core Explanation:**

Same proxy issue as `@Transactional` and `@Async`:

1. **Self-invocation**: `this.getCachedResult()` bypasses proxy → no caching.
2. **Private methods**: Not intercepted.
3. **Missing `@EnableCaching`**.
4. **`null` return values**: By default, `null` results are not cached. Use `@Cacheable(unless = "#result == null")`.
5. **Same cache key for different calls**: Incorrect SpEL key expression.

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("users", "products");
    }
}

@Service
public class UserService {

    @Cacheable(value = "users", key = "#id",
               condition = "#id > 0", unless = "#result == null")
    public User findById(Long id) {
        return repository.findById(id).orElse(null);
    }

    @CacheEvict(value = "users", key = "#user.id")
    public void update(User user) {
        repository.save(user);
    }

    @CachePut(value = "users", key = "#result.id")
    public User create(User user) {
        return repository.save(user); // updates cache with actual saved result
    }
}
```

---

### How do you implement graceful shutdown in Spring Boot?

**Core Explanation:**

```yaml
# application.yml
server:
  shutdown: graceful       # wait for in-flight requests
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # max wait time
```

```java
// With Spring Boot 2.3+, graceful shutdown handles:
// 1. Stops accepting new requests (readiness probe fails)
// 2. Waits for active requests to complete (up to timeout)
// 3. Shuts down beans in reverse creation order
// 4. Calls @PreDestroy methods
// 5. Closes thread pools

// Custom shutdown hook for additional cleanup
@Component
public class GracefulShutdown {
    private final ExecutorService asyncExecutor;

    @PreDestroy
    public void onShutdown() {
        log.info("Initiating graceful shutdown");
        asyncExecutor.shutdown();
        try {
            if (!asyncExecutor.awaitTermination(25, TimeUnit.SECONDS)) {
                asyncExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            asyncExecutor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}

// Kubernetes: readiness probe fails before SIGTERM → load balancer drains
// SIGTERM → Spring shutdown hook → graceful shutdown → SIGKILL after timeout
```

---

### What is the difference between WebClient, RestTemplate, and FeignClient? Which is recommended and why?

**Core Explanation:**

| | `RestTemplate` | `WebClient` | `FeignClient` |
|---|---|---|---|
| Status | Deprecated (maintenance mode) | Active (Spring 5+) | Active |
| Model | Blocking, synchronous | Non-blocking, reactive | Declarative proxy |
| Reactive | No | Yes (`Mono`, `Flux`) | No (unless with WebFlux) |
| Configuration | Manual | Builder pattern | Annotations on interface |
| Load balancing | With Ribbon (legacy) | Yes | Yes (with Spring Cloud LB) |
| Circuit breaker | Manual | Manual | Built-in with Resilience4j |

**Recommendation:**
- **WebClient**: Preferred for new Spring Boot apps — non-blocking, supports both sync and async.
- **FeignClient**: Best for microservices with Spring Cloud — declarative, integrates with service discovery and circuit breakers.
- **RestTemplate**: Only in legacy codebases.

```java
// WebClient — reactive, non-blocking
@Component
public class InventoryClient {
    private final WebClient client;

    public InventoryClient(WebClient.Builder builder) {
        this.client = builder.baseUrl("http://inventory-service").build();
    }

    public Mono<StockResponse> getStock(String productId) {
        return client.get()
            .uri("/stock/{id}", productId)
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, response ->
                Mono.error(new ProductNotFoundException(productId)))
            .bodyToMono(StockResponse.class)
            .timeout(Duration.ofSeconds(5));
    }
}

// FeignClient — declarative
@FeignClient(name = "inventory-service", fallbackFactory = InventoryFallback.class)
public interface InventoryClient {
    @GetMapping("/stock/{productId}")
    StockResponse getStock(@PathVariable String productId);
}
```

---

### How does Spring Boot integrate with cloud services (AWS S3, RDS, SQS)?

**Core Explanation:**

Spring Cloud AWS provides auto-configured Spring Boot integration with AWS services:

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-s3</artifactId>
</dependency>
```

```java
// S3
@Service
public class DocumentService {
    private final S3Template s3Template;

    public void upload(String key, InputStream content) {
        s3Template.upload("my-bucket", key, content);
    }

    public S3Resource download(String key) {
        return s3Template.download("my-bucket", key);
    }
}

// SQS — listener
@SqsListener("order-events")
public void processOrderEvent(OrderEvent event) {
    orderProcessor.process(event);
}

// RDS — just configure the DataSource
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://${RDS_HOSTNAME}:5432/${RDS_DB_NAME}
    username: ${RDS_USERNAME}
    password: ${RDS_PASSWORD}
  # Use IAM authentication with RDS IAM plugin for production
```

---

### How do you create a custom Spring Boot Starter?

**Core Explanation:**

A custom starter bundles auto-configuration + dependencies that others can include with a single dependency.

**Structure:**
```
my-feature-spring-boot-starter/     # thin POM with dependencies
my-feature-spring-boot-autoconfigure/ # actual auto-configuration code
```

```java
// 1. Auto-configuration class
@AutoConfiguration
@ConditionalOnClass(FeatureService.class)
@EnableConfigurationProperties(FeatureProperties.class)
public class FeatureAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public FeatureService featureService(FeatureProperties props) {
        return new DefaultFeatureService(props.getApiKey());
    }
}

// 2. Properties class
@ConfigurationProperties(prefix = "my.feature")
public class FeatureProperties {
    private String apiKey;
    private boolean enabled = true;
    // getters/setters
}

// 3. Register in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.feature.FeatureAutoConfiguration

// 4. Provide configuration metadata for IDE support
// src/main/resources/META-INF/spring-configuration-metadata.json
```

---

### How does `@Conditional` family work?

**Core Explanation:**

```java
@Configuration
public class MessagingConfig {

    // Only if kafka-clients is on classpath
    @Bean
    @ConditionalOnClass(name = "org.apache.kafka.clients.producer.KafkaProducer")
    public KafkaMessageSender kafkaSender() { return new KafkaMessageSender(); }

    // Only if spring.messaging.type=rabbitmq
    @Bean
    @ConditionalOnProperty(name = "spring.messaging.type", havingValue = "rabbitmq")
    public RabbitMessageSender rabbitSender() { return new RabbitMessageSender(); }

    // Only if no MessageSender bean already defined
    @Bean
    @ConditionalOnMissingBean(MessageSender.class)
    public InMemoryMessageSender inMemorySender() { return new InMemoryMessageSender(); }

    // Only in web application context
    @Bean
    @ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
    public RequestContextFilter requestFilter() { return new RequestContextFilter(); }

    // Complex condition via custom Condition class
    @Bean
    @Conditional(ProductionEnvironmentCondition.class)
    public MonitoringService monitoring() { return new DatadogMonitoring(); }
}

public class ProductionEnvironmentCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String[] profiles = context.getEnvironment().getActiveProfiles();
        return Arrays.asList(profiles).contains("prod");
    }
}
```
# Section 8: Spring Security

---

## Basic Questions

---

### What is Spring Security? What problems does it solve?

**Core Explanation:**

Spring Security is a powerful, customizable authentication and access-control framework for Java applications. It solves:

1. **Authentication**: Verifying who the user is (username/password, JWT, OAuth2, LDAP).
2. **Authorization**: Verifying what the user can do (roles, permissions, URL-level, method-level).
3. **Security vulnerabilities**: CSRF protection, session fixation, clickjacking prevention, security headers.
4. **Password encoding**: Secure storage with bcrypt, scrypt, argon2.

Spring Security integrates seamlessly with Spring Boot — just add the dependency and it protects all endpoints by default.

---

### What is the difference between authentication and authorization?

**Core Explanation:**

- **Authentication** (AuthN): "Who are you?" — verifying identity.
  - Input: credentials (username/password, token, certificate).
  - Output: authenticated principal (user object with identity).
  - Example: Login with username and password.

- **Authorization** (AuthZ): "What are you allowed to do?" — verifying permissions.
  - Input: authenticated principal + requested resource/action.
  - Output: allow or deny.
  - Example: Admin can DELETE users; regular users cannot.

**Practical Example:**
```java
// Authentication — who is this request from?
// Spring Security verifies the JWT token and establishes identity

// Authorization — can this user do this?
@PreAuthorize("hasRole('ADMIN')") // only admins
@DeleteMapping("/users/{id}")
public void deleteUser(@PathVariable Long id) { ... }

@PreAuthorize("hasRole('USER') and #id == authentication.principal.id")
@GetMapping("/users/{id}/profile")
public UserProfile getProfile(@PathVariable Long id) { ... }
// User can only see their own profile
```

---

### What is `UserDetailsService`? What is its role in authentication?

**Core Explanation:**

`UserDetailsService` is the core interface Spring Security uses to load user-specific data during authentication. It has one method: `loadUserByUsername(String username)` that returns a `UserDetails` object.

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPasswordHash()) // already encoded
            .roles(user.getRoles().toArray(new String[0]))
            .accountNonExpired(true)
            .accountNonLocked(!user.isLocked())
            .credentialsNonExpired(true)
            .enabled(user.isActive())
            .build();
    }
}
```

Spring Security calls `loadUserByUsername()` during form-login or Basic Auth authentication, then compares the returned password with the submitted credential using `PasswordEncoder`.

---

### What is `PasswordEncoder`? Why should passwords never be stored in plain text?

**Core Explanation:**

`PasswordEncoder` is Spring Security's abstraction for password hashing. The most common implementation is `BCryptPasswordEncoder`.

**Why not plain text:**
1. Database breach exposes all user passwords immediately.
2. Users often reuse passwords across services.
3. Employees with DB access can see all passwords.
4. Legal/compliance requirements (GDPR, PCI-DSS).

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // work factor 12 (default 10)
}

// Usage in registration
@Service
public class UserService {
    @Autowired PasswordEncoder encoder;

    public void register(String email, String rawPassword) {
        String hashed = encoder.encode(rawPassword);  // bcrypt hash
        userRepository.save(new User(email, hashed));
    }

    public boolean verify(String rawPassword, String storedHash) {
        return encoder.matches(rawPassword, storedHash); // constant-time comparison
    }
}
```

---

### What is the difference between hashing and encryption?

**Core Explanation:**

| | Hashing | Encryption |
|---|---|---|
| Direction | One-way (irreversible) | Two-way (reversible with key) |
| Purpose | Integrity verification, password storage | Confidentiality (data transmission) |
| Key required | No | Yes |
| Examples | SHA-256, BCrypt, Argon2 | AES, RSA |
| Password storage | Use this | Never use this |

**Why not encrypt passwords?** If the encryption key is compromised, all passwords are immediately decryptable. Hashing has no reversal — even with the algorithm known, you can't recover the original password (only brute force).

---

### What is CSRF? Why is CSRF protection important for web applications?

**Core Explanation:**

**CSRF (Cross-Site Request Forgery)**: An attacker tricks a user's browser into sending authenticated requests to your application without the user's knowledge.

**How it works:**
1. User logs into `bank.com` (session cookie stored).
2. User visits malicious `evil.com`.
3. `evil.com` contains: `<img src="https://bank.com/transfer?to=attacker&amount=1000">`.
4. Browser sends the request WITH the session cookie → transfer happens!

**Spring Security's CSRF protection:** Adds a CSRF token to each form/response. The token must be included in state-changing requests (POST, PUT, DELETE). Attacker can't read the token from another origin (same-origin policy).

```java
// CSRF disabled for stateless REST APIs (where JWT is used instead of sessions)
http.csrf(csrf -> csrf.disable());

// For web applications with sessions — keep CSRF enabled (default)
// Spring Boot includes CSRF token in responses automatically
```

---

### What is CORS? How is it different from CSRF?

**Core Explanation:**

**CORS (Cross-Origin Resource Sharing)**: Browser policy restricting requests from one origin to another. It's a **client-side** browser restriction (not a server security measure by itself).

**CSRF vs CORS:**
- CORS: Prevents `evil.com`'s JavaScript from **reading** responses from your API.
- CSRF: Prevents `evil.com` from tricking the browser into **sending** state-changing requests.

CORS does NOT prevent CSRF — form submissions and image tags bypass CORS restrictions!

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com")); // specific origins
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true); // allow cookies
        config.setMaxAge(3600L); // preflight cache duration

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
    // ...
}
```

---

### What is JWT? What are its three parts (header, payload, signature)?

**Core Explanation:**

**JWT (JSON Web Token)** is a compact, self-contained token for transmitting claims between parties. It's stateless — the server doesn't need to store session data.

**Structure:** `header.payload.signature` (Base64URL encoded, separated by dots)

1. **Header**: Algorithm and token type.
   ```json
   {"alg": "HS256", "typ": "JWT"}
   ```

2. **Payload**: Claims (user data). NOT encrypted — visible to anyone with the token.
   ```json
   {"sub": "user123", "email": "alice@example.com", "roles": ["USER"], "exp": 1701000000}
   ```

3. **Signature**: HMAC of `header.payload` using the secret key. Ensures the token wasn't tampered with.
   ```
   HMACSHA256(base64(header) + "." + base64(payload), secretKey)
   ```

**Example token:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiJ1c2VyMTIzIiwiZXhwIjoxNzAxMDAwMDAwfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

---

## Intermediate Questions

---

### What are the core components of Spring Security (SecurityFilterChain, AuthenticationManager, AuthenticationProvider, GrantedAuthority)?

**Core Explanation:**

```
HTTP Request
    ↓
SecurityFilterChain (ordered chain of filters)
    ↓
UsernamePasswordAuthenticationFilter (or JwtAuthenticationFilter)
    ↓
AuthenticationManager (delegates to providers)
    ↓
AuthenticationProvider (actual authentication logic)
    ↓ uses
UserDetailsService → loads UserDetails
    ↓
SecurityContextHolder.getContext().setAuthentication(auth)
    ↓
Authorization check: GrantedAuthority vs required permissions
```

- **`SecurityFilterChain`**: Ordered chain of security filters applied to HTTP requests.
- **`AuthenticationManager`**: Coordinates authentication, delegates to configured `AuthenticationProvider`s.
- **`AuthenticationProvider`**: Performs actual authentication (e.g., `DaoAuthenticationProvider` checks username/password).
- **`GrantedAuthority`**: Permission/role granted to a principal (e.g., `ROLE_ADMIN`, `READ_ORDERS`).
- **`SecurityContext`**: Thread-local holder of the current authenticated principal.

---

### How does JWT authentication flow work in Spring Boot?

**Core Explanation:**

```java
// 1. JWT Filter — runs before every request
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired JwtTokenProvider tokenProvider;
    @Autowired UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String token = extractToken(request); // from "Authorization: Bearer <token>"

        if (token != null && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsernameFromToken(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(userDetails, null,
                                                        userDetails.getAuthorities());
            auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(auth); // set auth
        }

        filterChain.doFilter(request, response);
        SecurityContextHolder.clearContext(); // clear after request
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}

// 2. Register filter in SecurityFilterChain
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/auth/**").permitAll()
            .anyRequest().authenticated());
    return http.build();
}
```

---

### What is the difference between Access Token and Refresh Token? How do you implement the refresh flow?

**Core Explanation:**

| | Access Token | Refresh Token |
|---|---|---|
| Lifetime | Short (15 min – 1 hour) | Long (7 – 30 days) |
| Usage | Every API request | Only to get new access tokens |
| Storage | Memory (JS) or httpOnly cookie | httpOnly cookie (more secure) |
| Revocable | Difficult (stateless) | Yes (stored in DB) |

**Refresh flow:**
```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @PostMapping("/login")
    public AuthResponse login(@RequestBody LoginRequest request) {
        Authentication auth = authManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password()));

        String accessToken = jwtProvider.generateAccessToken(auth, Duration.ofMinutes(15));
        String refreshToken = jwtProvider.generateRefreshToken(auth, Duration.ofDays(7));

        // Store refresh token hash in DB for revocation support
        refreshTokenRepo.save(new RefreshToken(auth.getName(),
                                               DigestUtils.sha256Hex(refreshToken)));

        // Refresh token in httpOnly cookie
        ResponseCookie cookie = ResponseCookie.from("refresh_token", refreshToken)
            .httpOnly(true).secure(true).path("/api/auth/refresh").maxAge(7 * 86400).build();
        response.addHeader("Set-Cookie", cookie.toString());

        return new AuthResponse(accessToken);
    }

    @PostMapping("/refresh")
    public AuthResponse refresh(@CookieValue("refresh_token") String refreshToken) {
        if (!jwtProvider.validateToken(refreshToken)) throw new UnauthorizedException();

        String username = jwtProvider.getUsernameFromToken(refreshToken);

        // Check if refresh token exists in DB (not revoked)
        refreshTokenRepo.findByUsernameAndTokenHash(username, DigestUtils.sha256Hex(refreshToken))
            .orElseThrow(UnauthorizedException::new);

        String newAccessToken = jwtProvider.generateAccessToken(username, Duration.ofMinutes(15));
        return new AuthResponse(newAccessToken);
    }

    @PostMapping("/logout")
    public void logout(@CookieValue("refresh_token") String refreshToken) {
        String username = jwtProvider.getUsernameFromToken(refreshToken);
        refreshTokenRepo.deleteByUsernameAndTokenHash(username, DigestUtils.sha256Hex(refreshToken));
    }
}
```

---

### What is OAuth 2.0? Name the four grant types and when to use each.

**Core Explanation:**

OAuth 2.0 is an **authorization** framework that allows third-party applications to access resources on behalf of a user without exposing the user's credentials.

**Four grant types:**

1. **Authorization Code** (most secure, for server-side web apps):
   - User redirected to auth server → gets code → server exchanges for token.
   - Token never exposed to browser.

2. **Authorization Code with PKCE** (for SPAs/mobile apps):
   - Same as Auth Code but uses a code verifier/challenge instead of a client secret.
   - Secure without a client secret.

3. **Client Credentials** (machine-to-machine):
   - Service authenticates with its own credentials — no user involved.
   - Used for microservice-to-microservice auth.

4. **Device Code** (input-constrained devices):
   - Smart TVs, IoT — displays code on device, user authenticates on phone.

**Deprecated:** Implicit flow (tokens in URL fragments — insecure).

---

### How do you implement role-based access control (RBAC)?

**Core Explanation:**

```java
// Method-level security
@Service
public class UserAdminService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) { ... }

    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public List<User> listUsers() { ... }

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public User getProfile(Long userId) { ... } // admin or own profile

    @PostAuthorize("returnObject.department == authentication.principal.department")
    public Report getReport(Long reportId) { ... } // check after method executes
}

// URL-level security
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/admin/**").hasRole("ADMIN")
        .requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")
        .requestMatchers("/api/reports/**").hasAnyRole("ADMIN", "ANALYST")
        .requestMatchers("/api/public/**").permitAll()
        .anyRequest().authenticated()
    );
    return http.build();
}

// Enable method security
@Configuration
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class MethodSecurityConfig { }
```

---

### What is hashing and salting? Why is BCrypt preferred over SHA-256 for passwords?

**Core Explanation:**

**Hashing**: One-way transformation of a password to a fixed-length string.

**Salting**: Adding a unique random value (salt) before hashing. Prevents:
- **Rainbow table attacks**: Precomputed hashes are useless with unique salts.
- **Identical password = identical hash**: Two users with the same password have different hashes.

**Why BCrypt over SHA-256:**
1. **BCrypt includes a built-in salt**: Each hash is unique.
2. **Intentionally slow**: Work factor (`10–14`) makes brute force impractical (milliseconds per hash in BCrypt vs nanoseconds in SHA-256).
3. **Adaptive**: Work factor can be increased as hardware gets faster.
4. SHA-256 is designed for **speed** (document integrity), making it trivially fast to brute force.

```java
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
// "password" → "$2a$12$...60 chars..." — different each call (unique salt embedded)
// 2^12 = 4096 rounds of processing → ~100-300ms per hash — brute force resistant
```

---

## Advanced Questions

---

### What is `SecurityContext` and `SecurityContextHolder`? How is the context propagated across async threads?

**Core Explanation:**

`SecurityContextHolder` stores the `SecurityContext` (which contains the authenticated `Authentication` object) in a `ThreadLocal` by default. This is why the current user is accessible anywhere without passing it explicitly.

**Async propagation problem:** `@Async` and `CompletableFuture` run on different threads — they don't inherit the parent thread's `SecurityContext`.

```java
// Problem
@Async
public void processOrder(Long orderId) {
    // SecurityContextHolder.getContext().getAuthentication() == null!
    String username = getCurrentUser(); // fails
}

// Solution 1: DelegatingSecurityContextExecutor
@Bean
public Executor taskExecutor() {
    return new DelegatingSecurityContextExecutorService(
        Executors.newFixedThreadPool(4) // wraps standard executor
    );
}

// Solution 2: Configure SecurityContextHolder strategy
@PostConstruct
void configureSecurityContextPropagation() {
    SecurityContextHolder.setStrategyName(
        SecurityContextHolder.MODE_INHERITABLETHREADLOCAL // inherits to child threads
    );
}

// Solution 3: Manual propagation
SecurityContext context = SecurityContextHolder.getContext();
CompletableFuture.runAsync(() -> {
    SecurityContextHolder.setContext(context); // copy to async thread
    try {
        processOrder(orderId);
    } finally {
        SecurityContextHolder.clearContext();
    }
});
```

---

### What is session fixation? How does Spring Security protect against it?

**Core Explanation:**

**Session fixation attack:**
1. Attacker visits your site, gets a session ID (e.g., `JSESSIONID=ABC123`).
2. Attacker crafts a login URL with this session ID and tricks the victim into clicking it.
3. Victim logs in — their authenticated session has ID `ABC123`.
4. Attacker (who knows `ABC123`) is now authenticated as the victim!

**Spring Security's protection:** After successful authentication, Spring Security creates a **new session** with a new session ID, invalidating the old one.

```java
http.sessionManagement(session -> session
    .sessionFixation(fixation ->
        fixation.changeSessionId()) // default — new ID, keep attributes
        // OR: fixation.newSession() — new ID, new session (lose attributes)
        // OR: fixation.none() — dangerous, disable protection
);
```

---

### How does the Authorization Code Grant flow with PKCE work? Why is the implicit flow deprecated?

**Core Explanation:**

**Authorization Code with PKCE (Proof Key for Code Exchange):**

```
1. App generates code_verifier (random string)
2. App computes code_challenge = SHA256(code_verifier) → Base64URL
3. User directed to: /authorize?code_challenge=...&code_challenge_method=S256
4. User authenticates, auth server returns authorization code
5. App exchanges code + code_verifier: POST /token {code, code_verifier}
6. Auth server verifies SHA256(code_verifier) == code_challenge
7. Returns access_token + refresh_token
```

**Why implicit flow is deprecated:**
- In implicit flow, the access token is returned in the URL fragment (`#access_token=...`).
- URL fragments are visible in browser history, referrer headers, and server logs.
- No refresh token — forced re-login or insecure token storage.
- PKCE solves the same problem (no client secret for SPAs) more securely — code is useless without the verifier.

---

### What is a `DelegatingPasswordEncoder`? How does it support password migration?

**Core Explanation:**

`DelegatingPasswordEncoder` prefixes hashes with the encoding algorithm ID and delegates to the appropriate encoder:

```
{bcrypt}$2a$10$...  → BCryptPasswordEncoder
{noop}password123   → NoOpPasswordEncoder (plain text — NEVER in production)
{argon2}$argon2id$  → Argon2PasswordEncoder
```

**Password migration scenario:**
```java
// Old system used MD5 (insecure) — need to migrate to BCrypt
// DelegatingPasswordEncoder can verify MD5 AND encode new passwords with BCrypt

PasswordEncoder encoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();
// Default encoding: BCrypt
// Supports verification of: bcrypt, scrypt, argon2, noop, sha256, etc.

// At login time:
// 1. Load stored hash: "{md5}5f4dcc3b5aa765d61d8327deb882cf99"
// 2. DelegatingPasswordEncoder picks MD5 encoder, verifies password
// 3. If verified AND stored hash is old format: re-encode with BCrypt, save
// 4. Next login: stored hash is "{bcrypt}..." — upgraded!

// Gradual migration without password resets:
@Service
public class LoginService {
    public void login(String username, String rawPassword) {
        User user = userRepo.findByUsername(username);
        if (encoder.matches(rawPassword, user.getPasswordHash())) {
            // upgrade if using old algorithm
            if (!encoder.upgradeEncoding(user.getPasswordHash())) return;
            user.setPasswordHash(encoder.encode(rawPassword));
            userRepo.save(user);
        }
    }
}
```

---

### How do you implement custom authentication providers for non-standard auth mechanisms?

**Core Explanation:**

```java
// Custom AuthenticationProvider — e.g., for API key authentication
@Component
public class ApiKeyAuthenticationProvider implements AuthenticationProvider {

    @Autowired ApiKeyRepository apiKeyRepo;

    @Override
    public Authentication authenticate(Authentication auth) throws AuthenticationException {
        String apiKey = (String) auth.getCredentials();

        return apiKeyRepo.findByKey(apiKey)
            .map(key -> {
                if (key.isExpired()) throw new CredentialsExpiredException("API key expired");
                List<GrantedAuthority> authorities = key.getScopes().stream()
                    .map(scope -> new SimpleGrantedAuthority("SCOPE_" + scope))
                    .collect(Collectors.toList());
                return new UsernamePasswordAuthenticationToken(key.getOwner(), null, authorities);
            })
            .orElseThrow(() -> new BadCredentialsException("Invalid API key"));
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return ApiKeyAuthenticationToken.class.isAssignableFrom(authentication);
    }
}

// Register in security config
@Bean
public SecurityFilterChain filterChain(HttpSecurity http,
                                        ApiKeyAuthenticationProvider apiKeyProvider) throws Exception {
    http.authenticationProvider(apiKeyProvider);
    // Add filter to extract API key from headers...
    return http.build();
}
```
# Section 9: Hibernate / JPA

---

## Basic Questions

---

### What is JPA? How is it different from Hibernate?

**Core Explanation:**

- **JPA (Jakarta Persistence API)**: A **specification** (JSR 338) that defines how Java objects map to relational databases. It's a set of interfaces and annotations — it has no implementation.
- **Hibernate**: The most popular **implementation** of the JPA specification. It adds features beyond the JPA spec (like `@Where`, `@SQLDelete`, extra caching strategies).

**Analogy:** JPA is to Hibernate as JDBC is to MySQL Connector. JPA defines the contract; Hibernate fulfills it.

```java
// JPA — spec interfaces
import jakarta.persistence.EntityManager;  // JPA interface
import jakarta.persistence.TypedQuery;     // JPA interface

// Hibernate — implementation classes (use via JPA interfaces in most cases)
import org.hibernate.Session;             // Hibernate's own API
import org.hibernate.SessionFactory;      // Hibernate's own API
```

---

### What is ORM? What problem does it solve?

**Core Explanation:**

**ORM (Object-Relational Mapping)** bridges the gap between object-oriented Java models and relational database tables — the "impedance mismatch."

**Problems it solves:**
1. **Boilerplate**: Eliminates writing `PreparedStatement`, `ResultSet`, manual mapping for every query.
2. **Type mapping**: Automatically converts Java types to SQL types.
3. **Object graphs**: Handles relationships (one-to-many, many-to-many) via annotations.
4. **Lazy loading**: Only fetches data when needed.
5. **Caching**: First-level and second-level caches reduce DB roundtrips.
6. **Database portability**: Switch from MySQL to PostgreSQL by changing the dialect.

---

### What is an Entity? What annotations make a POJO a JPA entity?

**Core Explanation:**

An **Entity** is a Java class mapped to a database table. Required annotations:

```java
@Entity                  // marks as JPA entity
@Table(name = "orders",  // maps to specific table (optional if class name matches)
       schema = "public")
public class Order {

    @Id                              // primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY) // auto-increment
    private Long id;

    @Column(name = "order_date",    // column mapping (optional if field name matches)
            nullable = false,
            columnDefinition = "TIMESTAMP DEFAULT NOW()")
    private LocalDateTime orderDate;

    @Column(length = 100)
    private String status;

    // Default constructor REQUIRED by JPA (can be protected)
    protected Order() {}

    public Order(LocalDateTime orderDate) {
        this.orderDate = orderDate;
        this.status = "PENDING";
    }
}
```

**Minimum required:** `@Entity` + `@Id`. Everything else has defaults.

---

### What is `@GeneratedValue`? What are the ID generation strategies?

**Core Explanation:**

| Strategy | Behavior | Best for |
|---|---|---|
| `AUTO` | JPA picks a strategy based on database | Portability |
| `IDENTITY` | DB auto-increment (`AUTO_INCREMENT`, `SERIAL`) | MySQL, PostgreSQL — most common |
| `SEQUENCE` | Uses DB sequence object | PostgreSQL, Oracle — best performance |
| `TABLE` | Uses a separate table as sequence — portable | Avoid in production (locks) |

```java
// IDENTITY — simple, DB assigns on INSERT
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// SEQUENCE — better for batch inserts (Hibernate can pre-allocate IDs)
@Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
@SequenceGenerator(name = "order_seq", sequenceName = "orders_id_seq", allocationSize = 50)
private Long id;
// allocationSize=50: Hibernate fetches 50 IDs at once from DB — fewer roundtrips

// UUID — application-generated, good for distributed systems
@Id @GeneratedValue(strategy = GenerationType.UUID)
private UUID id; // Java 17+ with Hibernate 6+
```

---

### What is an `EntityManager`?

**Core Explanation:**

`EntityManager` is the central JPA API for performing persistence operations. It manages the **Persistence Context** (the cache of managed entities).

```java
@Repository
public class OrderRepository {
    @PersistenceContext
    private EntityManager em;

    public Order findById(Long id) {
        return em.find(Order.class, id); // returns null if not found
    }

    public void save(Order order) {
        em.persist(order); // NEW → MANAGED
    }

    public Order update(Order order) {
        return em.merge(order); // DETACHED → MANAGED (returns managed copy)
    }

    public void delete(Long id) {
        Order order = em.find(Order.class, id);
        em.remove(order); // MANAGED → REMOVED
    }

    public List<Order> findByStatus(String status) {
        return em.createQuery("SELECT o FROM Order o WHERE o.status = :status", Order.class)
                 .setParameter("status", status)
                 .getResultList();
    }
}
```

In Spring Data JPA, you rarely use `EntityManager` directly — `JpaRepository` wraps it.

---

### What is a Persistence Context?

**Core Explanation:**

The **Persistence Context** is a first-level cache that tracks managed entities within a session/transaction. Think of it as a "unit of work":

- Every entity loaded or saved within a transaction is tracked.
- Changes to managed entities are automatically detected (dirty checking).
- Duplicate queries for the same entity return the cached version.
- Flushed to DB on transaction commit.

```java
@Transactional
public void demonstratePersistenceContext() {
    Order o1 = em.find(Order.class, 1L); // DB query
    Order o2 = em.find(Order.class, 1L); // NO DB query — returns same instance from cache
    assert o1 == o2; // TRUE — same object reference

    o1.setStatus("SHIPPED"); // change detected
    // No em.save() needed — dirty checking will flush on commit
}
```

---

### What is lazy loading vs eager loading?

**Core Explanation:**

- **Lazy loading**: Related entities are loaded from DB **only when accessed** (first access triggers a query). Default for `@OneToMany` and `@ManyToMany`.
- **Eager loading**: Related entities are loaded **immediately** with the parent entity (JOIN). Default for `@ManyToOne` and `@OneToOne`.

```java
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)  // default for collections
    private List<OrderItem> items; // not loaded until items is accessed

    @ManyToOne(fetch = FetchType.EAGER) // default for @ManyToOne
    private Customer customer; // always loaded with Order
}

// Lazy loading pitfall — must be inside a transaction
@Transactional
public void process(Long orderId) {
    Order order = repo.findById(orderId).get(); // loads Order only
    order.getItems().size(); // triggers SELECT for items — still inside transaction, OK
}

// LazyInitializationException — accessing lazy outside transaction
public void outside(Long orderId) {
    Order order = repo.findById(orderId).get(); // transaction ends
    order.getItems().size(); // LazyInitializationException!
}
```

---

### What is the difference between `CrudRepository` and `JpaRepository`?

**Core Explanation:**

| | `CrudRepository<T, ID>` | `JpaRepository<T, ID>` |
|---|---|---|
| Basic CRUD | `save()`, `findById()`, `delete()`, etc. | All of CrudRepository |
| Batch operations | No | `saveAll()`, `deleteAllInBatch()` |
| Pagination | No | `findAll(Pageable)` |
| Flush | No | `flush()`, `saveAndFlush()` |
| Entity-specific features | No | Yes |

**Best practice:** Use `JpaRepository` in Spring Boot applications — it provides everything you need.

---

### What is `@Transactional`?

**Core Explanation:**

`@Transactional` declaratively manages database transactions. Spring creates a transaction before the method starts and commits/rolls back when it ends.

```java
@Service
public class OrderService {

    @Transactional // starts transaction, commits on success, rolls back on RuntimeException
    public Order placeOrder(CreateOrderRequest request) {
        Order order = new Order(request.customerId(), request.items());
        orderRepository.save(order);
        inventoryService.deductStock(request.items()); // in same transaction
        emailService.sendConfirmation(order); // if this throws, entire tx rolls back
        return order;
    }

    @Transactional(readOnly = true) // performance optimization — no dirty checking, no flush
    public List<Order> findRecentOrders(Long customerId) {
        return orderRepository.findByCustomerIdOrderByCreatedAtDesc(customerId);
    }

    @Transactional(rollbackFor = CheckedException.class) // roll back on checked exception too
    public void processRefund(Long orderId) throws CheckedException { ... }
}
```

---

## Intermediate Questions

---

### What is the entity lifecycle (Transient, Managed, Detached, Removed)?

**Core Explanation:**

```
new Entity()          →  TRANSIENT (no ID, not tracked)
       ↓ em.persist()
    MANAGED           (has ID, tracked by persistence context)
       ↓ transaction ends / em.detach() / em.clear()
    DETACHED          (has ID, NOT tracked — changes not persisted automatically)
       ↓ em.merge()
    MANAGED           (re-attached)
       ↓ em.remove()
    REMOVED           (will be deleted on flush)
```

```java
@Transactional
public void lifecycle() {
    // TRANSIENT
    Order order = new Order();

    // MANAGED — Hibernate tracks this
    em.persist(order); // assigned ID, entered persistence context

    // Modifying managed entity — auto-detected (dirty checking)
    order.setStatus("PROCESSING"); // no explicit save needed

    // DETACHED (after transaction ends or em.detach())
    em.detach(order);
    order.setStatus("SHIPPED"); // change NOT tracked

    // Back to MANAGED
    Order merged = em.merge(order); // re-attaches, returns managed copy
    // Note: 'order' is still detached; 'merged' is managed

    // REMOVED
    em.remove(merged);
    // DELETE query executed on flush/commit
}
```

---

### What is dirty checking? How does Hibernate detect changes without explicit `update()` calls?

**Core Explanation:**

When an entity enters the persistence context (via `find()`, `persist()`, or `merge()`), Hibernate takes a **snapshot** of its state. At flush time (before commit or explicit flush), Hibernate compares the current entity state to the snapshot. If they differ, it generates an `UPDATE` statement.

```java
@Transactional
public void updateOrder(Long id, String newStatus) {
    Order order = orderRepository.findById(id).get(); // snapshot taken here
    order.setStatus(newStatus); // change current state
    // No save() call needed!
    // At commit: Hibernate detects status changed → generates UPDATE
}
```

**Performance note:** With many entities, dirty checking scans all managed entities at flush time. For batch processing, use `@Modifying` queries, clear the persistence context, and use `FlushMode.COMMIT` to avoid unnecessary scanning.

---

### What is the difference between `save()`, `persist()`, and `merge()`?

**Core Explanation:**

| Method | Source | Behavior |
|---|---|---|
| `persist()` (JPA) | EntityManager | Transitions TRANSIENT → MANAGED; does not return anything |
| `merge()` (JPA) | EntityManager | Copies DETACHED state to MANAGED; returns the managed copy |
| `save()` (Spring Data) | JpaRepository | `persist()` for new entities, `merge()` for existing |

```java
// persist() — new entity only
Order newOrder = new Order(); // no ID
em.persist(newOrder); // persists, ID assigned by DB

// merge() — detached entity
Order detached = fetchFromSomewhere(); // has ID, not managed
Order managed = em.merge(detached); // copy state to managed entity
// Use 'managed' going forward, not 'detached'

// save() — Spring Data (uses isNew() check based on ID/version)
orderRepository.save(newOrder);    // → persist
orderRepository.save(detachedOrder); // → merge
```

---

### What is the N+1 select problem? Demonstrate it and show two solutions.

**Core Explanation:**

N+1 occurs when loading a collection triggers 1 query for the parent + N separate queries for each child.

```java
// Entity
@Entity class Customer {
    @OneToMany(fetch = LAZY)
    List<Order> orders;
}

// N+1 problem
List<Customer> customers = customerRepo.findAll(); // 1 query
for (Customer c : customers) {
    c.getOrders().size(); // 1 query PER customer → N queries!
}
// Total: 1 + N queries (e.g., 1 + 100 = 101 queries for 100 customers!)
```

**Solution 1: JOIN FETCH (JPQL)**
```java
@Query("SELECT DISTINCT c FROM Customer c JOIN FETCH c.orders")
List<Customer> findAllWithOrders();
// 1 query with JOIN — returns all data at once
```

**Solution 2: @EntityGraph**
```java
@EntityGraph(attributePaths = {"orders"})
List<Customer> findAll();
// Spring Data generates JOIN FETCH automatically

// Or define on entity:
@Entity
@NamedEntityGraph(name = "Customer.withOrders",
    attributeNodes = @NamedAttributeNode("orders"))
class Customer { ... }
```

**Solution 3: Batch fetching (Hibernate-specific)**
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=20
# Loads up to 20 children per parent in 1 query using WHERE id IN (...)
```

---

### What is the difference between `JOIN` and `JOIN FETCH` in JPQL?

**Core Explanation:**

- **`JOIN`**: Filters the query using the join condition but does **NOT** initialize the collection. The association is still lazy.
- **`JOIN FETCH`**: Loads the joined entities **into the persistence context** — initializes the collection eagerly for this query.

```java
// JOIN — used for filtering only
@Query("SELECT o FROM Order o JOIN o.customer c WHERE c.country = 'US'")
List<Order> findUSOrders();
// Orders loaded; customer is still lazy (accessing order.getCustomer() triggers another query!)

// JOIN FETCH — loads customer eagerly for this query
@Query("SELECT o FROM Order o JOIN FETCH o.customer c WHERE c.country = 'US'")
List<Order> findUSOrdersWithCustomer();
// Order AND customer loaded in one query — no additional queries
```

---

### What is `LazyInitializationException`? What are the proper fixes?

**Core Explanation:**

`LazyInitializationException` occurs when you try to access a lazy-loaded association **outside an active transaction** (the Hibernate session/persistence context is closed).

```java
// Problem
Order order = orderRepository.findById(id).get(); // transaction ends here in typical Spring setup
order.getItems().size(); // LazyInitializationException — session closed!
```

**Proper fixes:**
```java
// Fix 1: JOIN FETCH in repository (best — load what you need upfront)
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);

// Fix 2: @EntityGraph
@EntityGraph(attributePaths = {"items", "items.product"})
Optional<Order> findById(Long id);

// Fix 3: @Transactional on service (keep session open)
@Transactional(readOnly = true)
public OrderDTO getOrderWithItems(Long id) {
    Order order = orderRepository.findById(id).get();
    order.getItems(); // inside transaction — OK
    return mapper.toDTO(order);
}

// Fix 4: Projections — only fetch what you need
interface OrderSummary {
    Long getId();
    String getStatus();
    List<ItemSummary> getItems(); // nested projection
}
```

**Anti-fix:** `spring.jpa.open-in-view=true` (Open Session In View) — keeps session open for the entire HTTP request. Causes lazy queries during view rendering, terrible for performance. Disable it.

---

### What are transaction propagation types? Explain `REQUIRED` vs `REQUIRES_NEW` with a real scenario.

**Core Explanation:**

Propagation controls what happens when a `@Transactional` method calls another `@Transactional` method:

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing transaction; if none, create new one |
| `REQUIRES_NEW` | Always creates a new transaction; suspends outer if exists |
| `SUPPORTS` | Use existing if available; no transaction if none |
| `NOT_SUPPORTED` | Suspend existing transaction; run without transaction |
| `MANDATORY` | Must have existing transaction; throw if none |
| `NEVER` | Must NOT have transaction; throw if one exists |
| `NESTED` | Savepoint within existing transaction |

**Real scenario:**
```java
@Service
public class OrderService {

    @Transactional // REQUIRED — outer transaction
    public void placeOrder(Order order) {
        orderRepo.save(order);
        auditService.log("Order placed: " + order.getId()); // JOINS outer tx
        // if log() fails → entire order transaction rolls back

        auditService.logSafely("Order placed: " + order.getId()); // REQUIRES_NEW
        // logSafely has its own tx — if it fails, only audit fails; order is COMMITTED
    }
}

@Service
public class AuditService {

    @Transactional // REQUIRED — joins outer transaction
    public void log(String message) {
        auditRepo.save(new AuditLog(message));
        throw new RuntimeException("audit failed"); // ROLLS BACK ENTIRE ORDER!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW) // new tx
    public void logSafely(String message) {
        try {
            auditRepo.save(new AuditLog(message));
        } catch (Exception e) {
            // This transaction rolls back, outer transaction is unaffected
            log.warn("Audit logging failed", e);
        }
    }
}
```

---

### What is optimistic locking? How does `@Version` work?

**Core Explanation:**

**Optimistic locking** assumes conflicts are rare. Instead of locking rows, it detects conflicts at commit time by comparing a version column.

`@Version` adds a `version` column (INTEGER or TIMESTAMP). When updating:
1. Hibernate reads the current version.
2. `UPDATE ... SET status='SHIPPED', version=version+1 WHERE id=? AND version=?`
3. If `version` changed (another transaction updated), the WHERE clause matches 0 rows → `OptimisticLockException`.

```java
@Entity
public class Product {
    @Id private Long id;
    private int quantity;

    @Version
    private Long version; // Hibernate manages this automatically
}

// Two concurrent requests:
// Transaction 1: load Product (version=1), deduct stock
// Transaction 2: load Product (version=1), deduct stock
// Transaction 1 commits: UPDATE ... version=2 WHERE version=1 → success
// Transaction 2 tries: UPDATE ... version=2 WHERE version=1 → 0 rows → OptimisticLockException!

// Handle in REST API:
@PutMapping("/products/{id}/stock")
public ResponseEntity<?> deductStock(@PathVariable Long id, @RequestBody int amount) {
    try {
        productService.deductStock(id, amount);
        return ResponseEntity.ok().build();
    } catch (OptimisticLockingFailureException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body("Concurrent modification — please retry");
    }
}
```

---

## Advanced Questions

---

### How do you optimize large batch inserts?

**Core Explanation:**

By default, Hibernate inserts one row at a time. For bulk inserts, configure batch processing:

```properties
# application.properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
# Required for IDENTITY generation to work with batching:
spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true
```

```java
@Service
@Transactional
public class BulkImportService {
    @Autowired EntityManager em;

    public void importProducts(List<ProductCSV> rows) {
        for (int i = 0; i < rows.size(); i++) {
            Product p = mapper.toProduct(rows.get(i));
            em.persist(p);

            if (i % 50 == 0) { // match batch_size
                em.flush();  // send batch to DB
                em.clear();  // clear persistence context (prevent memory bloat)
            }
        }
    }
}

// Alternative: Spring Data's saveAll() with configured batch_size
// Or: Spring Batch for very large datasets (100k+ rows)
// Or: JDBC bulk insert for maximum performance
```

**Note:** `IDENTITY` generation strategy disables JDBC batching (Hibernate needs the ID after each INSERT to populate the entity). Use `SEQUENCE` with `allocationSize` for true batching.

---

### How do you map inheritance in JPA? What are the trade-offs?

**Core Explanation:**

| Strategy | DB Structure | Pros | Cons |
|---|---|---|---|
| `SINGLE_TABLE` | One table, discriminator column | Best performance, simple queries | Many nullable columns, no NOT NULL on subclass fields |
| `TABLE_PER_CLASS` | One table per concrete class | Clean, no joins | Polymorphic queries use UNION (slow), no shared PK sequence |
| `JOINED` | Parent table + subclass tables | Normalized, no nulls | JOIN on every query (slower) |

```java
// SINGLE_TABLE (default and most used)
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type")
public abstract class Payment {
    @Id Long id;
    double amount;
}

@Entity @DiscriminatorValue("CARD")
public class CardPayment extends Payment { String cardNumber; }

@Entity @DiscriminatorValue("BANK")
public class BankPayment extends Payment { String bankAccount; }
// Both in 'payment' table with payment_type='CARD' or 'BANK'

// JOINED (normalized)
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment { @Id Long id; double amount; }

@Entity
public class CardPayment extends Payment { String cardNumber; }
// 'payment' table: id, amount
// 'card_payment' table: id (FK → payment), card_number
// Query: SELECT ... FROM payment JOIN card_payment ON ...
```

---

### How do you implement auditing?

**Core Explanation:**

```java
// Enable JPA auditing
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .map(Authentication::getName);
    }
}

// Auditable base entity
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(nullable = false, updatable = false, length = 100)
    private String createdBy;

    @LastModifiedBy
    @Column(nullable = false, length = 100)
    private String updatedBy;
}

@Entity
public class Order extends AuditableEntity {
    @Id private Long id;
    // createdAt, updatedAt, createdBy, updatedBy inherited
}
```

---

### How do you implement soft deletes?

**Core Explanation:**

```java
@Entity
@SQLDelete(sql = "UPDATE orders SET deleted = true WHERE id = ?") // override DELETE
@Where(clause = "deleted = false") // filter out soft-deleted records
public class Order {
    @Id Long id;
    private boolean deleted = false;
    // Hibernate uses @SQLDelete instead of DELETE query
    // @Where filters all queries on this entity
}

// Usage
orderRepository.delete(order); // runs UPDATE, not DELETE
orderRepository.findAll(); // automatically adds WHERE deleted = false

// To include deleted records (admin use case):
@Query(value = "SELECT * FROM orders WHERE id = :id", nativeQuery = true)
Optional<Order> findByIdIncludingDeleted(@Param("id") Long id);
```

---

### What are common JPA anti-patterns?

**Core Explanation:**

1. **N+1 problem**: Accessing lazy collections in a loop without JOIN FETCH.
2. **Open Session In View**: Keeps Hibernate session open for entire HTTP request — triggers lazy queries during view rendering.
3. **Entity as DTO**: Returning `@Entity` from REST controllers — risk of lazy loading exceptions and over-exposing data.
4. **Wrong fetch type**: `@ManyToOne(fetch=EAGER)` on collections — loads everything always.
5. **Missing `@Version`**: No optimistic locking on concurrent entities → lost updates.
6. **Large `@Transactional` methods**: Holding a transaction open during HTTP calls or long processing.
7. **`CascadeType.ALL` carelessly**: `REMOVE` cascade on collections can delete child entities unintentionally.
8. **No connection pool tuning**: Default HikariCP settings may not suit production load.

---

### What are projections in Spring Data JPA?

**Core Explanation:**

Projections fetch only specific columns, improving performance over loading entire entities:

```java
// Interface-based projection — Spring generates proxy
public interface OrderSummary {
    Long getId();
    String getStatus();
    LocalDateTime getCreatedAt();
    // Nested:
    CustomerInfo getCustomer();
    interface CustomerInfo {
        String getName();
        String getEmail();
    }
}

// Usage
List<OrderSummary> findByStatus(String status); // SELECT id, status, created_at, customer info

// Class-based (DTO) projection
public record OrderDTO(Long id, String status, String customerName) {}

@Query("SELECT new com.example.OrderDTO(o.id, o.status, c.name) FROM Order o JOIN o.customer c")
List<OrderDTO> findOrderDTOs();

// Dynamic projection — caller decides what shape
<T> List<T> findByStatus(String status, Class<T> type);

// Usage:
List<OrderSummary> summaries = repo.findByStatus("PENDING", OrderSummary.class);
List<Order> fullOrders = repo.findByStatus("PENDING", Order.class);
```
# Section 10: Microservices Architecture

---

## Basic Questions

---

### What are microservices? How do they differ from monolithic architecture?

**Core Explanation:**

**Microservices** decompose an application into small, independently deployable services, each owning its data and business logic.

| | Monolith | Microservices |
|---|---|---|
| Deployment | One unit | Independent per service |
| Scalability | Scale entire application | Scale individual services |
| Technology | Single stack | Polyglot (each service can differ) |
| Data | Shared database | Database per service |
| Team | Single team | Independent teams per service |
| Failure | One bug can crash all | Failure isolated to one service |
| Complexity | Low initially | High (network, distributed data) |

---

### What are the key benefits of microservices?

**Core Explanation:**

1. **Independent deployment**: Deploy one service without redeploying others — faster release cycles.
2. **Independent scaling**: Scale only the services under load (e.g., scale payment service during sales peak).
3. **Technology diversity**: Each service can use the best-fit technology (Java, Go, Python).
4. **Fault isolation**: A crash in the notification service doesn't take down the order service.
5. **Team autonomy**: Small teams own services end-to-end → reduced coordination overhead.
6. **Easier maintenance**: Smaller codebases are easier to understand and modify.

---

### What are the key challenges of microservices?

**Core Explanation:**

1. **Network complexity**: Service calls can fail, be slow, or time out.
2. **Distributed data**: No shared DB → data consistency is harder (eventual consistency).
3. **Service discovery**: Services need to find each other dynamically.
4. **Distributed tracing**: A single user request may span 10+ services — hard to debug.
5. **Operational overhead**: Dozens of services need individual monitoring, logging, CI/CD.
6. **Testing complexity**: Integration tests need multiple services running.
7. **Data consistency**: Transactions across services require Saga or 2PC.
8. **Latency**: Network roundtrips add latency compared to in-process calls.

---

### When should you NOT use microservices?

**Core Explanation:**

Avoid microservices when:
1. **Small team** (< 5-10 engineers): The operational overhead outweighs the benefits.
2. **Early-stage product**: Domain boundaries are unclear — premature decomposition creates wrong services.
3. **Simple domain**: Low complexity where a monolith is sufficient.
4. **Data-intensive operations**: Joins across service boundaries are painful (requires data duplication or aggregation APIs).
5. **High consistency requirements**: If you need strong ACID transactions across all data, microservices make this very hard.

**Rule of thumb:** Start with a well-structured monolith. Extract services when a specific component needs independent scaling or when team ownership requires it.

---

### What is an API Gateway? Why is it needed?

**Core Explanation:**

An **API Gateway** is the single entry point for all client requests to microservices. It handles:

1. **Request routing**: Routes `/api/orders` to order-service, `/api/users` to user-service.
2. **Authentication/Authorization**: Validates JWT before forwarding to services.
3. **Rate limiting**: Prevents abuse.
4. **SSL termination**: Handles HTTPS externally; services communicate over HTTP internally.
5. **Load balancing**: Distributes requests across service instances.
6. **Request/response transformation**: Aggregates responses from multiple services (BFF pattern).
7. **Circuit breaking**: Stops cascading failures to clients.

**Examples:** AWS API Gateway, Kong, Netflix Zuul, Spring Cloud Gateway.

---

### What is the Circuit Breaker pattern?

**Core Explanation:**

The **Circuit Breaker** prevents cascading failures by monitoring calls to a service and stopping calls when the service is failing:

- **Closed** (normal): Requests pass through. Tracks failure rate.
- **Open** (service failing): Requests immediately fail with fallback — no calls to the failing service.
- **Half-Open** (recovery check): Allows a few test requests. If they succeed, closes the circuit.

```java
// Resilience4j
@CircuitBreaker(name = "inventoryService", fallbackMethod = "defaultInventory")
public StockResponse getStock(String productId) {
    return inventoryClient.getStock(productId);
}

public StockResponse defaultInventory(String productId, Exception ex) {
    log.warn("Circuit breaker fallback for {}: {}", productId, ex.getMessage());
    return new StockResponse(productId, 0, "UNKNOWN"); // degraded response
}
```

---

## Intermediate Questions

---

### How do you decompose a monolith into microservices (DDD, bounded contexts)?

**Core Explanation:**

Use **Domain-Driven Design (DDD)** to identify bounded contexts:

1. **Event Storming**: Workshop where domain experts and developers map out domain events (OrderPlaced, PaymentProcessed, ItemShipped).
2. **Identify aggregates**: Groups of entities that change together (Order + OrderItems).
3. **Find bounded contexts**: Areas where a model is internally consistent. Different meanings of "Customer" in Sales vs Support → different bounded contexts.
4. **Strangler Fig pattern**: Gradually extract bounded contexts from the monolith, routing traffic to new services incrementally.

**Warning signs of wrong decomposition:**
- Services constantly calling each other for simple operations (distributed monolith).
- Shared database between services.
- One business operation requires synchronized calls to 5 services.

---

### What is the Saga pattern? Explain choreography vs orchestration with trade-offs.

**Core Explanation:**

**Saga** is a pattern for managing distributed transactions without 2PC. Each step has a compensating transaction (rollback action).

**Choreography**: Services emit events and react to each other's events. No central coordinator.
```
OrderService: emit "OrderCreated"
    → InventoryService: reserve stock, emit "StockReserved"
        → PaymentService: charge, emit "PaymentProcessed"
            → OrderService: confirm order

On failure:
PaymentService: emit "PaymentFailed"
    → InventoryService: release stock
        → OrderService: cancel order
```

**Orchestration**: A central saga orchestrator tells each service what to do.
```java
@Component
public class OrderSagaOrchestrator {
    public void execute(Order order) {
        try {
            inventoryClient.reserveStock(order.getItems());
            paymentClient.charge(order.getPayment());
            orderService.confirmOrder(order.getId());
        } catch (PaymentException e) {
            inventoryClient.releaseStock(order.getItems()); // compensate
            orderService.cancelOrder(order.getId());
        }
    }
}
```

| | Choreography | Orchestration |
|---|---|---|
| Coupling | Loose | Tighter (orchestrator depends on all services) |
| Visibility | Hard to track overall flow | Clear flow in orchestrator |
| Single point of failure | No | Orchestrator |
| Debugging | Hard | Easier |
| Best for | Simple flows | Complex multi-step workflows |

---

### What is the Outbox pattern? How does it solve the dual-write problem?

**Core Explanation:**

**Dual-write problem**: You need to save to DB AND publish an event to Kafka. If one succeeds and the other fails, you have inconsistency.

**Outbox pattern** solves this by writing to an `outbox` table **in the same DB transaction** as the business entity. A separate publisher reads the outbox and sends events:

```java
@Transactional
public Order placeOrder(CreateOrderRequest request) {
    Order order = new Order(request);
    orderRepository.save(order); // main entity

    // In SAME transaction:
    outboxRepository.save(new OutboxEvent(
        "OrderPlaced",
        "{\"orderId\": " + order.getId() + ", ...}", // event payload
        "orders" // target Kafka topic
    ));
    return order;
    // If Kafka is down: order saved, outbox event saved → publisher retries later
    // No inconsistency possible
}

// Separate publisher (Debezium CDC or polling):
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishOutboxEvents() {
    List<OutboxEvent> unpublished = outboxRepository.findByPublishedFalse();
    for (OutboxEvent event : unpublished) {
        kafkaTemplate.send(event.getTopic(), event.getPayload());
        event.setPublished(true);
    }
}
// Debezium (Change Data Capture) watches DB transaction log — more efficient than polling
```

---

### What is CQRS? When should you use it and when is it overkill?

**Core Explanation:**

**CQRS (Command Query Responsibility Segregation)**: Separate the model for writing (Commands) from the model for reading (Queries). They can have different data stores optimized for each.

```
Write side:                    Read side:
[Command] → [Domain Model] →   [Read Model / Projection]
            [Write DB]    →    [Read DB (denormalized)]
```

**When CQRS makes sense:**
- High read/write ratio requiring different optimization (scale reads independently).
- Complex domain logic on writes + simple flat data on reads.
- Event sourcing (CQRS is the natural complement).

**When it's overkill:**
- CRUD-heavy apps with simple domain logic.
- Small teams — operational and cognitive complexity isn't worth it.
- No significant read/write asymmetry.

---

### How does Resilience4j implement Circuit Breaker, Retry, and Bulkhead?

**Core Explanation:**

```java
// application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        failure-rate-threshold: 50          # open when 50% of last 10 calls fail
        wait-duration-in-open-state: 10s    # wait before half-open
        permitted-number-of-calls-in-half-open-state: 3
        sliding-window-size: 10

  retry:
    instances:
      inventoryService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException

  bulkhead:
    instances:
      externalApi:
        max-concurrent-calls: 10            # at most 10 concurrent calls
        max-wait-duration: 100ms            # wait up to 100ms for a slot

// Application code
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
@Retry(name = "inventoryService")
@Bulkhead(name = "externalApi")
public PaymentResponse charge(PaymentRequest request) {
    return paymentClient.charge(request);
}

public PaymentResponse paymentFallback(PaymentRequest request, Exception ex) {
    log.warn("Payment service unavailable, queuing for retry");
    pendingPaymentQueue.add(request);
    return PaymentResponse.pending(request.getOrderId());
}
```

---

### What is the Saga pattern at scale? How do you handle compensation and partial failure rollback?

**Core Explanation:**

At scale, saga compensation requires:

1. **Idempotent compensations**: Compensating actions must be safe to repeat (network retries).
2. **Compensation log**: Track which steps succeeded so you know what to compensate.
3. **Timeout handling**: Define maximum saga duration; compensate if exceeded.
4. **Out-of-order event handling**: Events may arrive out of order; saga must handle this.

```java
// Using a saga state machine
@Entity
public class OrderSaga {
    @Id Long id;
    String orderId;
    SagaStatus status; // STARTED, INVENTORY_RESERVED, PAYMENT_CHARGED, COMPLETED, COMPENSATING, FAILED
    @ElementCollection
    List<CompletedStep> completedSteps;

    void complete(SagaStep step) { completedSteps.add(new CompletedStep(step)); }
    boolean needsCompensation(SagaStep step) { return completedSteps.contains(step); }
}

// Compensation logic
public void compensate(OrderSaga saga) {
    // Compensate in reverse order
    if (saga.needsCompensation(PAYMENT_CHARGED)) {
        paymentClient.refund(saga.getPaymentId()); // idempotent
    }
    if (saga.needsCompensation(INVENTORY_RESERVED)) {
        inventoryClient.releaseStock(saga.getItems()); // idempotent
    }
    saga.setStatus(SagaStatus.FAILED);
}
```

---

### What is the Outbox + Kafka + CQRS together?

**Core Explanation:**

```
Command side:
[PlaceOrder command] → [OrderService] → [orders table + outbox table] (single tx)
                                        ↓ Debezium CDC
                                     [Kafka: orders topic]

Query side:
[Kafka consumer] → reads events → updates [order_read_model table]
[Read API] → queries order_read_model (denormalized, fast reads)
```

This combination provides:
- **Reliability**: Outbox guarantees event delivery.
- **Performance**: Read model optimized for queries (no joins needed).
- **Scalability**: Read and write sides scaled independently.

---

### What is idempotency? How do you design idempotent APIs in practice?

**Core Explanation:**

An **idempotent** operation produces the same result no matter how many times it's called with the same input. Critical for retries in distributed systems.

**Idempotency key pattern:**
```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> processPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    // Check if this request was already processed
    return idempotencyStore.get(idempotencyKey)
        .map(cachedResponse -> ResponseEntity.ok(cachedResponse)) // return cached
        .orElseGet(() -> {
            PaymentResponse response = paymentService.charge(request);
            idempotencyStore.put(idempotencyKey, response, Duration.ofDays(1));
            return ResponseEntity.ok(response);
        });
}

// Idempotency store (Redis)
@Component
public class RedisIdempotencyStore {
    @Autowired RedisTemplate<String, String> redis;

    public Optional<PaymentResponse> get(String key) {
        String cached = redis.opsForValue().get("idempotency:" + key);
        return Optional.ofNullable(cached).map(json -> deserialize(json, PaymentResponse.class));
    }

    public void put(String key, PaymentResponse response, Duration ttl) {
        redis.opsForValue().set("idempotency:" + key, serialize(response), ttl);
    }
}
```

---

### How do you implement distributed tracing?

**Core Explanation:**

```java
// Spring Boot with Micrometer + OpenTelemetry (Spring Boot 3+)
// application.yml
management:
  tracing:
    sampling:
      probability: 1.0 # trace all requests (use 0.1 in production)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans

// Automatic: Spring adds trace-id and span-id to all logs and HTTP headers
// Log output: [order-service,traceId=abc123,spanId=def456] - Order placed

// Propagate context across service calls:
@Component
public class TracingWebClient {
    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .filter((request, next) -> {
                // Spring automatically propagates B3/W3C trace headers
                return next.exchange(request);
            })
            .build();
    }
}

// View traces: Zipkin at localhost:9411 or Jaeger at localhost:16686
// Search by trace ID to see the full request journey across services
```

---

### What is blue-green deployment? How does it differ from canary and rolling?

**Core Explanation:**

| Strategy | Description | Traffic | Rollback |
|---|---|---|---|
| **Blue-Green** | Two identical environments; switch traffic 100% at once | 0% → 100% | Instant — switch back |
| **Canary** | Route small % of traffic to new version; gradually increase | 5% → 25% → 100% | Easy — redirect % back |
| **Rolling** | Gradually replace old instances with new | Gradual instance-by-instance | Complex — some instances are already replaced |

```yaml
# Kubernetes — Blue-Green: two deployments, switch service selector
# Blue deployment (current production)
metadata:
  labels:
    version: blue

# Service selector
selector:
  version: blue  # ← change to 'green' for instant cutover

# Canary: split traffic
# Istio VirtualService
spec:
  http:
  - route:
    - destination:
        host: order-service
        subset: stable
      weight: 90
    - destination:
        host: order-service
        subset: canary
      weight: 10 # 10% canary traffic
```

---

### What metrics matter most for microservices in production?

**Core Explanation:**

Use **RED** and **USE** methods:

**RED (Request-centric):**
- **Rate**: Requests per second (RPS).
- **Errors**: Error rate (4xx/5xx per second).
- **Duration**: Request latency (p50, p95, p99).

**USE (Resource-centric):**
- **Utilization**: CPU %, memory %, connection pool %, thread pool %.
- **Saturation**: Queue depth, backpressure signals.
- **Errors**: Hardware/OS level errors.

**Key microservice metrics:**
```java
// Spring Boot Actuator + Micrometer auto-exposes:
// http.server.requests — count, error rate, duration
// jvm.memory.used — heap/non-heap usage
// hikaricp.connections.active — DB connection pool
// kafka.consumer.fetch-rate — Kafka consumer throughput

// Custom business metrics
@Component
public class OrderMetrics {
    private final Counter ordersPlaced;
    private final Timer checkoutDuration;

    public OrderMetrics(MeterRegistry registry) {
        ordersPlaced = registry.counter("orders.placed", "region", "eu");
        checkoutDuration = registry.timer("checkout.duration");
    }

    public void recordOrder() { ordersPlaced.increment(); }
    public void recordCheckout(long ms) { checkoutDuration.record(ms, MILLISECONDS); }
}
```

**Alert on:**
- Error rate > 1% → investigate.
- p99 latency > SLA threshold → scale or optimize.
- Connection pool utilization > 80% → increase pool or optimize queries.
- Consumer lag growing → scale consumers.

---

### How do you trace a single request across 10+ microservices?

**Core Explanation:**

**Distributed tracing** assigns a unique `trace-id` to each incoming request. Every service that handles the request creates a **span** tagged with the same `trace-id`. All spans are sent to a tracing backend (Zipkin, Jaeger) where they're assembled into a trace timeline.

```
Browser → API Gateway → Order Service → Inventory Service → Payment Service → Notification Service
trace-id: abc123

Spans:
[api-gateway    ] span=111 parent=null      200ms total
[order-service  ] span=222 parent=111       150ms
[inventory-svc  ] span=333 parent=222        50ms
[payment-svc    ] span=444 parent=222        80ms
[notification   ] span=555 parent=222        20ms (async)
```

**Log correlation:**
```java
// Add trace-id to every log line (auto with Spring Sleuth/Micrometer Tracing)
2024-01-15 [traceId=abc123,spanId=222] OrderService - Processing order 456
2024-01-15 [traceId=abc123,spanId=333] InventoryService - Reserving 2 units of P001
// Search all logs: traceId=abc123 → see the entire request flow
```

**Tools:** Jaeger, Zipkin, AWS X-Ray, Datadog APM, Grafana Tempo.
# Section 11: SQL & Database

---

## Basic Questions

---

### What is the difference between `WHERE` and `HAVING`?

**Core Explanation:**

- **`WHERE`**: Filters **rows** before grouping. Cannot use aggregate functions.
- **`HAVING`**: Filters **groups** after `GROUP BY`. Can use aggregate functions.

```sql
-- WHERE: filter individual rows before grouping
SELECT department, AVG(salary) as avg_salary
FROM employees
WHERE hire_date > '2020-01-01'  -- filter rows first
GROUP BY department;

-- HAVING: filter groups after aggregation
SELECT department, AVG(salary) as avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 75000;  -- filter groups

-- Combined
SELECT department, AVG(salary) as avg_salary
FROM employees
WHERE status = 'ACTIVE'       -- row filter
GROUP BY department
HAVING AVG(salary) > 75000;  -- group filter
```

---

### What is the difference between `INNER JOIN`, `LEFT JOIN`, and `RIGHT JOIN`?

**Core Explanation:**

| Join | Returns |
|---|---|
| `INNER JOIN` | Only rows with matches in BOTH tables |
| `LEFT JOIN` | All rows from LEFT table + matching rows from right (NULLs for no match) |
| `RIGHT JOIN` | All rows from RIGHT table + matching rows from left |
| `FULL OUTER JOIN` | All rows from BOTH tables (NULLs where no match) |

```sql
-- INNER JOIN — only customers who have orders
SELECT c.name, o.id
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- LEFT JOIN — all customers, including those with no orders
SELECT c.name, o.id
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;
-- Customers with no orders: name = "Bob", o.id = NULL

-- Find customers with NO orders (anti-join)
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;
```

---

### What is a primary key? Can a table have multiple primary keys?

**Core Explanation:**

A **primary key** uniquely identifies each row in a table. It must be:
- **Unique**: No two rows have the same PK value.
- **NOT NULL**: Cannot be null.
- **Immutable**: Should not change once set.

**Can a table have multiple primary keys?** No — a table has exactly **one** primary key. However, the primary key can be **composite** (spanning multiple columns).

```sql
-- Simple primary key
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL
);

-- Composite primary key
CREATE TABLE order_items (
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    PRIMARY KEY (order_id, product_id) -- composite PK
);
```

---

### What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?

**Core Explanation:**

| | `DELETE` | `TRUNCATE` | `DROP` |
|---|---|---|---|
| What it removes | Rows (filtered or all) | All rows | Entire table structure |
| WHERE clause | Yes | No | No |
| Rollback | Yes (DML, logged) | Database-dependent | No (DDL) |
| Triggers | Fires `DELETE` triggers | Does not fire triggers | No |
| Performance | Slow (row by row, logged) | Fast (deallocates pages) | Instant |
| Auto-increment | Reset? No | Yes (resets sequence) | N/A |

```sql
DELETE FROM orders WHERE status = 'CANCELLED';  -- selective delete
DELETE FROM temp_orders;                         -- delete all, slow, logged

TRUNCATE TABLE temp_orders;                      -- fast, resets auto-increment

DROP TABLE orders;                               -- removes table entirely — irreversible!
```

---

### What are ACID properties? Explain each.

**Core Explanation:**

| Property | Description |
|---|---|
| **Atomicity** | All operations in a transaction succeed, or all fail ("all or nothing") |
| **Consistency** | Transaction brings DB from one valid state to another (constraints respected) |
| **Isolation** | Concurrent transactions execute as if sequential (no interference) |
| **Durability** | Committed transactions survive system failures (written to disk) |

```sql
-- Classic bank transfer — requires all four ACID properties
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit
    -- If debit succeeds but credit fails: ROLLBACK (Atomicity)
    -- Balance can't go negative: CHECK constraint (Consistency)
    -- Concurrent transfers don't interfere: isolation level (Isolation)
    -- After COMMIT, power outage doesn't lose data (Durability)
COMMIT;
```

---

### What is normalization? Why is 3NF commonly used?

**Core Explanation:**

**Normalization** is organizing database tables to reduce redundancy and improve data integrity.

- **1NF**: Each column holds atomic values; no repeating groups.
- **2NF**: 1NF + every non-key column depends on the **entire** primary key (no partial dependencies).
- **3NF**: 2NF + no transitive dependencies (non-key column depends only on PK, not on another non-key column).

**Why 3NF:** It eliminates most common data anomalies (update, insert, delete anomalies) while remaining practical. Going beyond 3NF (BCNF, 4NF) often sacrifices query performance with little real-world benefit.

```sql
-- Violation: zip_code → city (transitive dependency in 3NF)
-- orders(order_id, customer_id, city, zip_code)
-- city depends on zip_code, not order_id

-- 3NF fix: extract to addresses table
-- orders(order_id, customer_id, address_id)
-- addresses(address_id, zip_code, city, street)
```

---

## Intermediate Questions

---

### What are window functions? Explain `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()`.

**Core Explanation:**

Window functions perform calculations over a **window** (a set of rows related to the current row) without collapsing rows into groups (unlike `GROUP BY`).

| Function | Tie handling |
|---|---|
| `ROW_NUMBER()` | Unique sequential number (no ties) |
| `RANK()` | Tied rows get same rank; next rank skipped |
| `DENSE_RANK()` | Tied rows get same rank; next rank NOT skipped |

```sql
SELECT
    name, department, salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank
FROM employees;

/*
name    dept  salary  row_num  rank  dense_rank
Alice   Eng   90000   1        1     1
Bob     Eng   90000   2        1     1     ← tie
Charlie Eng   80000   3        3     2     ← RANK skips to 3, DENSE_RANK goes to 2
*/

-- Find top 3 highest salaries per department
WITH ranked AS (
    SELECT *, DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) dr
    FROM employees
)
SELECT * FROM ranked WHERE dr <= 3;
```

---

### What is a CTE (Common Table Expression)? When would you use it over a subquery?

**Core Explanation:**

A CTE (WITH clause) is a named temporary result set valid for one query. It improves readability and enables recursion.

**Use CTE over subquery when:**
- The result is used **multiple times** in the query (CTE computed once, subquery may be computed multiple times).
- The logic is **complex** and benefits from a named step.
- You need **recursive** queries (tree traversal).

```sql
-- CTE — readable, named steps
WITH
monthly_sales AS (
    SELECT DATE_TRUNC('month', order_date) AS month, SUM(total) AS revenue
    FROM orders
    GROUP BY 1
),
ranked_months AS (
    SELECT *, RANK() OVER (ORDER BY revenue DESC) AS rank
    FROM monthly_sales
)
SELECT * FROM ranked_months WHERE rank <= 3;

-- Recursive CTE — employee hierarchy
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL  -- root (CEO)

    UNION ALL

    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT level, name FROM org_chart ORDER BY level, name;
```

---

### What is an index? What are clustered vs non-clustered indexes?

**Core Explanation:**

An **index** is a B-tree data structure that allows fast lookups without scanning every row.

| | Clustered Index | Non-Clustered Index |
|---|---|---|
| Data storage | Table data physically sorted by index key | Separate structure with pointers to rows |
| Count per table | One (it IS the table organization) | Many |
| Performance | Extremely fast for range scans | One extra lookup (pointer) for non-covering |
| Default | Primary key creates clustered index (MySQL InnoDB) | All other indexes |

```sql
-- Clustered index: primary key (InnoDB sorts the actual table rows by PK)
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,  -- clustered index on MySQL InnoDB
    customer_id BIGINT,
    status VARCHAR(50)
);

-- Non-clustered index: for fast lookups on non-PK columns
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status); -- composite
```

---

### What is a composite index? Does column order matter?

**Core Explanation:**

A **composite index** covers multiple columns. Column order is **critical** due to the **leftmost prefix rule**: the index can be used for queries that start with the leftmost column(s).

```sql
-- Index on (customer_id, status, created_at)
CREATE INDEX idx_orders_search ON orders (customer_id, status, created_at);

-- Uses index:
WHERE customer_id = 5                           -- uses leftmost column ✓
WHERE customer_id = 5 AND status = 'PENDING'   -- uses first two columns ✓
WHERE customer_id = 5 AND status = 'PENDING'
  AND created_at > '2024-01-01'                -- uses all three columns ✓

-- DOES NOT use index efficiently:
WHERE status = 'PENDING'                        -- skips customer_id ✗
WHERE created_at > '2024-01-01'                -- skips both ✗

-- Guideline: put high-cardinality equality columns first, range columns last
CREATE INDEX idx ON orders (customer_id, status, created_at);
-- customer_id (equality) → status (equality) → created_at (range)
```

---

### When should you NOT create an index?

**Core Explanation:**

Avoid indexes when:
1. **Low-cardinality columns**: `status` with 3 values — index barely helps (query scans ~33% anyway).
2. **Small tables**: Full scan of 1000 rows is faster than index lookup + random I/O.
3. **Write-heavy tables**: Each write updates all indexes — too many indexes → write bottleneck.
4. **Rarely queried columns**: Index never used → wasted space and write overhead.
5. **Columns frequently updated**: Reindexing overhead on every update.

---

### What is an execution plan? How do you read it?

**Core Explanation:**

An execution plan shows how the database engine executes a query — which indexes it uses, which joins, estimated costs.

```sql
-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 5 AND status = 'PENDING';

/*
Bitmap Heap Scan on orders  (cost=4.19..25.32 rows=10 width=100) (actual time=0.123..0.456 rows=8)
  Recheck Cond: ((customer_id = 5) AND (status = 'PENDING'))
  ->  Bitmap Index Scan on idx_orders_customer_status  (cost=0.00..4.19 rows=10)
        Index Cond: ((customer_id = 5) AND (status = 'PENDING'))
Planning Time: 0.234 ms
Execution Time: 0.512 ms
*/

-- Key indicators to look for:
-- "Seq Scan" on large tables → missing index
-- "rows=" estimate vs actual rows >> difference → stale statistics (run ANALYZE)
-- "cost=" high number → expensive operation
-- "loops=" high → index used inside nested loop (may indicate N+1)
-- "Filter:" after index scan → index not covering all conditions
```

---

### What is the difference between dirty read, non-repeatable read, and phantom read?

**Core Explanation:**

| Anomaly | Description | Prevented by |
|---|---|---|
| **Dirty read** | Read uncommitted data from another transaction (may be rolled back) | Read Committed |
| **Non-repeatable read** | Same row read twice returns different values (another tx updated it between reads) | Repeatable Read |
| **Phantom read** | Same query returns different rows (another tx inserted/deleted rows) | Serializable |

**Isolation levels:**
| Level | Dirty Read | Non-Repeatable | Phantom |
|---|---|---|---|
| `READ UNCOMMITTED` | Possible | Possible | Possible |
| `READ COMMITTED` (PostgreSQL default) | No | Possible | Possible |
| `REPEATABLE READ` (MySQL default) | No | No | Possible |
| `SERIALIZABLE` | No | No | No |

---

### How do you optimize a slow SQL query? List five techniques.

**Core Explanation:**

1. **Add or fix indexes**: Check execution plan for Seq Scans on large tables. Add composite index following access patterns.
2. **Use EXPLAIN ANALYZE**: Identify the most expensive node. Fix it before optimizing others.
3. **Avoid `SELECT *`**: Only fetch needed columns → less I/O, fits in memory.
4. **Rewrite correlated subqueries as JOINs**: Correlated subqueries execute once per row; JOINs are much more efficient.
5. **Paginate large result sets**: Use keyset pagination instead of `LIMIT/OFFSET` for large offsets.
6. **Update statistics**: `ANALYZE table_name` refreshes planner estimates.
7. **Partition large tables**: Range/list partitioning to limit scan scope.
8. **Covering indexes**: Include all needed columns in the index to avoid table lookups.

```sql
-- Slow: correlated subquery (executes per row)
SELECT name FROM customers c
WHERE (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) > 5;

-- Fast: JOIN + GROUP BY
SELECT c.name FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
HAVING COUNT(*) > 5;
```

---

## Advanced Questions

---

### How does pagination work with `LIMIT` and `OFFSET`? Why is it slow for large tables?

**Core Explanation:**

```sql
-- Offset pagination
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- Problem: DB must read and discard 10,000 rows before returning 20!
-- Cost grows linearly with page number → page 500 of 1000: reads 10,000 rows
```

**Keyset (cursor-based) pagination:**
```sql
-- First page
SELECT * FROM orders ORDER BY created_at DESC, id DESC LIMIT 20;
-- Returns rows; save the last row's (created_at, id) as cursor

-- Next page — use cursor values, no OFFSET
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-15 10:30:00', 456)  -- last seen
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- DB uses index starting from the cursor — O(log n) not O(n)
```

**Trade-offs of keyset:**
- Cannot jump to arbitrary page numbers.
- Cursor must be passed to the client.
- Ideal for infinite scroll, feeds, event logs.

---

### What is database sharding? How do you decide the shard key?

**Core Explanation:**

**Sharding** partitions data horizontally across multiple database instances, each holding a subset of rows.

**Shard key selection criteria:**
1. **High cardinality**: Many possible values → data distributed evenly.
2. **Even distribution**: Keys must distribute data without hotspots.
3. **Frequently used in queries**: Most queries should target a single shard (avoid cross-shard queries).
4. **Immutable**: Never changes — changing shard key requires data migration.

**Common shard key strategies:**
```
By tenant ID:    → good for multi-tenant SaaS (all tenant data on one shard)
By user ID:      → good for social apps (user's data co-located)
By date range:   → good for time-series (recent data on fast SSDs)
By geography:    → good for compliance (EU data stays in EU shards)
Hash-based:      → uniform distribution but no range queries
```

**Pitfalls:**
- "Hot shard": One user/tenant generates disproportionate traffic.
- Cross-shard joins: Extremely expensive — avoid in schema design.

---

### What is the difference between ACID and BASE consistency models?

**Core Explanation:**

| | ACID | BASE |
|---|---|---|
| Full name | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventually consistent |
| Focus | Strong consistency | Availability and partition tolerance |
| Used by | RDBMS (PostgreSQL, MySQL) | NoSQL (Cassandra, DynamoDB, MongoDB) |
| Trade-offs | Lower availability, less scalable | Eventual consistency — stale reads possible |
| When | Financial transactions, inventory | User profiles, social feeds, analytics |

---

### What is SQL injection? How do you prevent it in Java?

**Core Explanation:**

SQL injection occurs when user input is concatenated directly into SQL queries, allowing attackers to execute arbitrary SQL.

```java
// VULNERABLE
String query = "SELECT * FROM users WHERE username = '" + username + "'";
// username = "' OR '1'='1" → returns ALL users!
// username = "'; DROP TABLE users;--" → drops the table!

// SAFE: Parameterized queries
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ?");
ps.setString(1, username); // parameter is never interpreted as SQL
ResultSet rs = ps.executeQuery();

// SAFE: Spring Data JPA (parameterized by default)
@Query("SELECT u FROM User u WHERE u.username = :username")
Optional<User> findByUsername(@Param("username") String username);

// SAFE: Criteria API (type-safe, no string SQL)
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> user = cq.from(User.class);
cq.where(cb.equal(user.get("username"), username));
```

---

## SQL Coding Questions

---

### Write a query to find the Nth highest salary per department using window functions.

```sql
-- Find 2nd highest salary in each department
WITH ranked AS (
    SELECT
        name,
        department,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT name, department, salary
FROM ranked
WHERE salary_rank = 2; -- change to N for Nth highest
```

---

### Write a query to find the 2nd highest salary without using `LIMIT` or window functions.

```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- More robust: handles ties
SELECT DISTINCT salary
FROM employees e1
WHERE 1 = (
    SELECT COUNT(DISTINCT salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
);
```

---

### Write a query to find duplicate records and their count.

```sql
-- Find duplicate emails and their count
SELECT email, COUNT(*) AS count
FROM users
GROUP BY email
HAVING COUNT(*) > 1
ORDER BY count DESC;

-- Show all duplicate rows (not just the summary)
SELECT *
FROM users
WHERE email IN (
    SELECT email FROM users GROUP BY email HAVING COUNT(*) > 1
)
ORDER BY email;
```

---

### Write a query to calculate a running total using window functions.

```sql
SELECT
    order_date,
    order_id,
    amount,
    SUM(amount) OVER (ORDER BY order_date, order_id
                      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders
ORDER BY order_date, order_id;

-- Running total per customer
SELECT
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (PARTITION BY customer_id
                      ORDER BY order_date
                      ROWS UNBOUNDED PRECEDING) AS customer_running_total
FROM orders;
```

---

### Write a query to find the average salary per department where avg > 50000.

```sql
SELECT
    department,
    ROUND(AVG(salary), 2) AS avg_salary,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000
ORDER BY avg_salary DESC;
```

---

### Write a query to find employees who earn more than their manager (self join).

```sql
SELECT
    e.name AS employee,
    e.salary AS employee_salary,
    m.name AS manager,
    m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary
ORDER BY e.salary DESC;
```

---

### Write a query to fetch paginated records (page 3, 20 records per page).

```sql
-- OFFSET-based (simple but slow for large pages)
SELECT * FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 40;  -- page 3: skip (3-1)*20 = 40 rows

-- Keyset pagination (fast — use last page's cursor)
-- After page 2, you know the last row had: created_at='2024-01-15', id=100
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-15 10:00:00', 100)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Uses composite index (created_at, id) → O(log n) regardless of page number
```
# Sections 12 & 13: REST API Design + Kafka & Messaging

---

# Section 12: REST API Design

---

## Basic Questions

---

### What is a REST API? What are its key constraints?

**Core Explanation:**

**REST (Representational State Transfer)** is an architectural style for distributed hypermedia systems. A REST API uses HTTP to expose resources.

**Six constraints (Roy Fielding's definition):**
1. **Client-server**: Separation of UI and data storage concerns.
2. **Stateless**: Each request contains all information needed — server stores no session state.
3. **Cacheable**: Responses can be cached by clients/intermediaries.
4. **Uniform interface**: Consistent conventions (resource-based URLs, HTTP verbs, standard formats).
5. **Layered system**: Clients don't know if they're talking to the server or a proxy.
6. **Code on demand** (optional): Server can send executable code (e.g., JavaScript).

---

### What is the difference between `GET`, `POST`, `PUT`, `PATCH`, and `DELETE`?

**Core Explanation:**

| Method | Purpose | Idempotent | Safe |
|---|---|---|---|
| `GET` | Read resource | Yes | Yes |
| `POST` | Create new resource | No | No |
| `PUT` | Replace entire resource | Yes | No |
| `PATCH` | Partial update | No (by spec) | No |
| `DELETE` | Delete resource | Yes | No |

**Safe** = no side effects (read-only). **Idempotent** = same result no matter how many times called.

```
GET    /orders/123           → fetch order 123
POST   /orders               → create new order (returns 201 + Location header)
PUT    /orders/123           → replace entire order (all fields required)
PATCH  /orders/123           → partial update (only send changed fields)
DELETE /orders/123           → delete order (returns 204 No Content)
```

---

### What does "stateless" mean in REST?

**Core Explanation:**

Each HTTP request must contain **all information** necessary to understand and fulfill it. The server does not store client session state between requests.

**Implication:** No server-side sessions. Authentication (JWT/OAuth token) must be sent with every request. This enables:
- Horizontal scaling (any server can handle any request).
- Simpler failure recovery.
- Better cacheability.

```http
# Stateful (BAD for REST): Server stores session
POST /login → server stores session; subsequent requests rely on session cookie

# Stateless (GOOD): All context in each request
GET /orders/123
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
# Server validates token, extracts user identity from token itself
```

---

## Intermediate Questions

---

### What is idempotency? Which HTTP methods are idempotent?

**Core Explanation:**

An operation is **idempotent** if making the same request multiple times produces the same result as making it once. Critical for retry safety in distributed systems.

**Idempotent methods:** `GET`, `PUT`, `DELETE`, `HEAD`, `OPTIONS`

**NOT idempotent:** `POST` (each call creates a new resource), `PATCH` (applying patch multiple times may differ)

```http
# Idempotent — safe to retry
DELETE /orders/123   → first call: 204 No Content (deleted)
DELETE /orders/123   → second call: 404 Not Found (already deleted — safe outcome)

PUT /orders/123 {status: "SHIPPED"}  → same result on repeat

# Not idempotent
POST /orders {productId: 5}  → creates order #1 on first call
POST /orders {productId: 5}  → creates order #2 on second call (different result!)
```

---

### How do you design idempotent APIs in practice (idempotency key)?

**Core Explanation:**

For non-idempotent operations (especially POST), clients send an **Idempotency-Key** header. The server caches the result by this key and returns the cached response on duplicate requests.

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> charge(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody ChargeRequest request) {

    // Check Redis cache for this key
    PaymentResponse cached = redis.get("idem:" + idempotencyKey);
    if (cached != null) {
        return ResponseEntity.ok(cached); // return cached response
    }

    // First time — process and cache
    PaymentResponse response = paymentService.charge(request);

    // Cache with TTL (24h)
    redis.set("idem:" + idempotencyKey, response, Duration.ofHours(24));

    return ResponseEntity.status(201).body(response);
}
```

**Best practices:**
- Client generates UUID idempotency key per logical operation.
- Server returns the same HTTP status code on duplicates.
- Key should expire after a reasonable TTL (24h-7 days).
- Store: Redis, distributed cache, or database table.

---

### How do you implement API versioning?

**Core Explanation:**

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| **URI path** | `/api/v1/orders` | Visible, cacheable, simple | Pollutes URLs |
| **Query param** | `/orders?version=1` | Simple | Not restful, caching issues |
| **Header** | `API-Version: 1` | Clean URLs | Less visible, harder to test |
| **Content-type** | `Accept: application/vnd.company.v1+json` | True REST | Complex, less common |

**URI versioning (most common):**
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 { ... }
```

**Header versioning:**
```java
@GetMapping(value = "/orders/{id}",
            headers = "API-Version=2")
public OrderV2DTO getOrderV2(@PathVariable Long id) { ... }

@GetMapping(value = "/orders/{id}",
            headers = "API-Version=1")
public OrderV1DTO getOrderV1(@PathVariable Long id) { ... }
```

---

### How do you implement rate limiting for APIs (token bucket vs leaky bucket)?

**Core Explanation:**

**Token Bucket:**
- Bucket has capacity N tokens.
- Tokens replenished at rate R per second.
- Each request consumes 1 token. If empty → 429 Too Many Requests.
- Allows **burst** traffic (consume accumulated tokens).

**Leaky Bucket:**
- Requests enter a queue (bucket). Queue processes at fixed rate.
- Excess requests overflow → 429.
- **Smooth** output — no bursts allowed.

**Sliding Window** (common in practice):
- Track request count in a rolling time window.
- More accurate than fixed windows (no boundary burst).

```java
// Spring + Redis token bucket (using Bucket4j)
@Component
public class RateLimiter {
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    public boolean tryConsume(String clientId) {
        Bucket bucket = buckets.computeIfAbsent(clientId, k ->
            Bucket.builder()
                .addLimit(Bandwidth.classic(100,        // 100 tokens max
                           Refill.greedy(100, Duration.ofMinutes(1)))) // 100/minute
                .build()
        );
        return bucket.tryConsume(1);
    }
}

// Filter
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String clientId = req.getHeader("X-Client-Id");
        if (!rateLimiter.tryConsume(clientId)) {
            res.setStatus(429);
            res.addHeader("Retry-After", "60");
            return;
        }
        chain.doFilter(req, res);
    }
}
```

---

### What is the difference between HTTP/1.1, HTTP/2, and HTTP/3?

**Core Explanation:**

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | QUIC (UDP-based) |
| Multiplexing | No (one request/connection; pipelining problematic) | Yes — multiple streams in one connection | Yes — per-stream, no HOL blocking |
| Header compression | None | HPACK | QPACK |
| Server push | No | Yes | Yes |
| TLS | Optional | Effectively required | Always (built into QUIC) |
| Head-of-line blocking | TCP HOL blocking | TCP HOL blocking (still) | Eliminated |
| Performance | Baseline | ~2x faster | Best, especially on lossy networks |

**Practical impact for Java backends:**
- Spring Boot 3 with Tomcat supports HTTP/2 out of the box (with SSL).
- Reactive applications (WebFlux) benefit most from HTTP/2 multiplexing.
- HTTP/3 support is emerging (Quiche/netty-incubator).

---

### How do you handle long-running operations in REST APIs?

**Core Explanation:**

Three patterns:
1. **Polling**: Endpoint starts job, returns `202 Accepted` + polling URL. Client polls for status.
2. **Webhooks**: Client provides callback URL; server calls it on completion.
3. **Server-Sent Events / WebSocket**: Real-time push for status updates.

```java
// Polling pattern
@PostMapping("/reports")
public ResponseEntity<Void> generateReport(@RequestBody ReportRequest request) {
    String jobId = reportService.startGeneration(request); // async
    return ResponseEntity
        .accepted()
        .location(URI.create("/reports/" + jobId + "/status"))
        .build(); // 202 Accepted
}

@GetMapping("/reports/{jobId}/status")
public ResponseEntity<JobStatus> getStatus(@PathVariable String jobId) {
    JobStatus status = reportService.getStatus(jobId);
    if (status.isCompleted()) {
        return ResponseEntity.ok()
            .header("Location", "/reports/" + jobId + "/result")
            .body(status); // 200 with link to result
    }
    return ResponseEntity.ok(status); // 200 with "PROCESSING" status
}

@GetMapping("/reports/{jobId}/result")
public ResponseEntity<Report> getResult(@PathVariable String jobId) {
    return ResponseEntity.ok(reportService.getResult(jobId));
}
```

---

## Advanced Questions

---

### What are the trade-offs between REST and gRPC? When would you choose gRPC?

**Core Explanation:**

| | REST/JSON | gRPC |
|---|---|---|
| Protocol | HTTP/1.1 or 2 + JSON | HTTP/2 + Protocol Buffers |
| Performance | Slower (JSON parsing, text overhead) | ~5-10x faster (binary, compressed) |
| Schema | Optional (OpenAPI) | Required (.proto files) |
| Streaming | Polling or SSE | Bidirectional streaming built-in |
| Browser support | Native | Requires gRPC-Web proxy |
| Debugging | Easy — curl, Postman | Needs tools (grpcurl, BloomRPC) |
| Code generation | Optional | Automatic from .proto |
| Versioning | URL or header | Proto field numbers (backward compat) |

**Choose gRPC when:**
- Internal microservice-to-microservice communication (no browser clients).
- Performance-critical paths (10,000+ RPS with low latency).
- Need bidirectional streaming (real-time sensor data, chat).
- Polyglot services (gRPC generates clients in Go, Java, Python, etc.).

**Choose REST when:**
- Public APIs (developer experience, tooling, browser access).
- Webhooks and event callbacks.
- Simple CRUD with low performance requirements.

---

### How do you implement a custom rate limiter using Redis and the sliding window algorithm?

**Core Explanation:**

```java
@Component
public class SlidingWindowRateLimiter {
    @Autowired StringRedisTemplate redis;

    private static final int WINDOW_SECONDS = 60;
    private static final int MAX_REQUESTS = 100;

    public boolean isAllowed(String clientId) {
        String key = "rate:" + clientId;
        long now = System.currentTimeMillis();
        long windowStart = now - (WINDOW_SECONDS * 1000L);

        // Use Redis sorted set: score = timestamp, member = requestId
        String requestId = UUID.randomUUID().toString();

        return (Boolean) redis.execute(new SessionCallback<>() {
            @Override
            public Boolean execute(RedisOperations ops) {
                ops.multi();
                // Remove old requests outside the window
                ops.opsForZSet().removeRangeByScore(key, 0, windowStart);
                // Add current request
                ops.opsForZSet().add(key, requestId, now);
                // Count requests in window
                ops.opsForZSet().zCard(key);
                // Set TTL
                ops.expire(key, Duration.ofSeconds(WINDOW_SECONDS + 1));
                List<?> results = ops.exec();

                Long count = (Long) results.get(2);
                return count != null && count <= MAX_REQUESTS;
            }
        });
    }
}
```

---

# Section 13: Kafka & Messaging

---

## Basic Questions

---

### What is Apache Kafka? What problems does it solve?

**Core Explanation:**

Apache Kafka is a **distributed event streaming platform** designed for:
1. **High-throughput messaging**: Millions of events per second.
2. **Durability**: Events persisted to disk, replicated across brokers.
3. **Scalability**: Horizontal scaling via partitions.
4. **Real-time stream processing**: Kafka Streams API.
5. **Decoupling**: Producers and consumers are independent — producer doesn't wait for consumer.

**Problems it solves:**
- Decouples microservices (async communication).
- Handles backpressure (consumer processes at its own pace).
- Event replay: consumers can re-read historical events.
- Fan-out: one event to multiple consumers.

---

### What is the difference between a message queue and an event stream?

**Core Explanation:**

| | Message Queue (RabbitMQ) | Event Stream (Kafka) |
|---|---|---|
| Consumption model | Point-to-point — message consumed once | Log-based — multiple consumers, each at own offset |
| Retention | Deleted after consumption | Retained by time/size policy |
| Replay | No | Yes — consumers can re-read |
| Order | Per-queue FIFO | Per-partition |
| Throughput | Moderate | Very high |
| Routing | Complex (exchanges, bindings) | Simple (topics, consumer groups) |

---

### What are Kafka's core concepts?

**Core Explanation:**

- **Producer**: Publishes events to topics.
- **Consumer**: Reads events from topics.
- **Broker**: Kafka server instance. Cluster = multiple brokers.
- **Topic**: Named stream of events (like a category/table).
- **Partition**: Topics are divided into partitions. Ordered within a partition. Enables parallelism.
- **Offset**: Sequential position of an event within a partition. Consumers track their offset.
- **Consumer Group**: Multiple consumer instances sharing a topic. Each partition assigned to one consumer in the group.

```
Topic: "orders"
Partition 0: [offset 0: OrderA] [offset 1: OrderB] [offset 2: OrderC]
Partition 1: [offset 0: OrderD] [offset 1: OrderE]

Consumer Group "order-processor":
Consumer 1 → Partition 0 (reads offsets 0, 1, 2...)
Consumer 2 → Partition 1 (reads offsets 0, 1...)
```

---

### What is a consumer group?

**Core Explanation:**

A **consumer group** is a set of consumers that collectively consume a topic, with each partition assigned to exactly one consumer in the group. This enables:
- **Parallel consumption**: Multiple consumers divide the work.
- **Fault tolerance**: If one consumer fails, its partitions are reassigned.
- **Multiple subscriptions**: Different consumer groups can independently consume the same topic.

```java
@KafkaListener(topics = "orders", groupId = "order-processor")
public void processOrder(OrderEvent event) { /* each partition assigned to one instance */ }

// 3 instances of order-service + topic with 6 partitions:
// Instance 1 → Partitions 0, 1
// Instance 2 → Partitions 2, 3
// Instance 3 → Partitions 4, 5

// Second consumer group — independent consumption
@KafkaListener(topics = "orders", groupId = "analytics")
public void analyzeOrder(OrderEvent event) { /* gets ALL events independently */ }
```

---

## Intermediate Questions

---

### What delivery guarantees does Kafka provide?

**Core Explanation:**

| Guarantee | Description | Configuration |
|---|---|---|
| **At-most-once** | May lose messages, no duplicates | `acks=0`; auto-commit offset before processing |
| **At-least-once** | No message loss, may have duplicates | `acks=all`; commit offset after processing |
| **Exactly-once** | No loss, no duplicates | Transactional producer + `enable.idempotence=true` |

```java
// At-least-once (most common in practice)
@KafkaListener(topics = "orders", groupId = "processor")
public void process(OrderEvent event, Acknowledgment ack) {
    try {
        orderService.process(event); // process first
        ack.acknowledge(); // commit offset AFTER processing (at-least-once)
    } catch (Exception e) {
        // don't acknowledge → will redeliver
    }
}

// Exactly-once producer
@Bean
public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> factory) {
    KafkaTransactionManager<String, Object> tm = new KafkaTransactionManager<>(factory);
    return new KafkaTemplate<>(factory);
}

// application.yml for exactly-once
spring:
  kafka:
    producer:
      acks: all
      properties:
        enable.idempotence: true
        transactional.id: order-producer-1
```

---

### What happens if a consumer crashes after reading but before committing an offset?

**Core Explanation:**

With **manual offset commit** and at-least-once semantics:
1. Consumer reads message at offset 5.
2. Consumer processes message (e.g., saves to DB).
3. Consumer crashes before committing offset 5.
4. On restart: consumer reads from last committed offset (4) → receives message 5 again.
5. Result: message 5 processed **twice** (at-least-once).

**Solution for at-least-once:** Make consumers **idempotent** — processing the same message twice has no harmful side effect:
```java
@Transactional
public void processOrder(OrderEvent event) {
    // Check if already processed (idempotent)
    if (processedEventRepo.existsByEventId(event.getEventId())) {
        log.info("Duplicate event, skipping: {}", event.getEventId());
        return;
    }
    // Process and record
    orderService.save(event.toOrder());
    processedEventRepo.save(new ProcessedEvent(event.getEventId()));
}
```

---

### What is consumer lag? What causes it and how do you resolve it?

**Core Explanation:**

**Consumer lag** = difference between the latest message offset in a partition and the consumer's committed offset. Growing lag means consumers can't keep up with producers.

**Causes:**
1. Slow processing per message.
2. Insufficient consumer instances.
3. Network issues between consumer and broker.
4. Downstream service is slow (DB writes, external APIs).
5. GC pauses causing session timeouts and rebalances.

**Resolution:**
```java
// 1. Increase partitions (allows more consumer parallelism)
// Note: can't decrease partitions after creation

// 2. Scale consumer instances (auto-scaling based on lag metric)

// 3. Optimize processing (batch DB inserts, async I/O)

// 4. Increase batch size for throughput
spring:
  kafka:
    consumer:
      max-poll-records: 500         # process 500 messages per poll
      fetch-max-bytes: 52428800     # 50MB fetch
    listener:
      concurrency: 3                # 3 threads per consumer instance

// 5. Monitor lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processor
# Shows: TOPIC, PARTITION, CURRENT-OFFSET, LOG-END-OFFSET, LAG
```

---

### How do you handle poison messages?

**Core Explanation:**

A **poison message** is one that always fails processing (bad format, missing required field, business rule violation). Without handling, it blocks the consumer at that offset.

```java
@KafkaListener(topics = "orders")
public void process(OrderEvent event, Acknowledgment ack) {
    try {
        orderService.process(event);
        ack.acknowledge();
    } catch (ProcessingException e) {
        // Non-retriable error → send to DLQ
        kafkaTemplate.send("orders.DLQ", event);
        ack.acknowledge(); // acknowledge original to move past it
    }
}

// Spring Kafka — built-in dead letter publishing
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLQ", record.partition()));

    FixedBackOff backOff = new FixedBackOff(1000L, 3); // retry 3 times, 1s apart
    return new DefaultErrorHandler(recoverer, backOff);
}
```

---

## Advanced Questions

---

### What is exactly-once semantics in Kafka? How does the transactional API enable it?

**Core Explanation:**

**Exactly-once semantics (EOS)** ensures each message is processed exactly once end-to-end, even with retries and failures.

**Three components:**
1. **Idempotent producer** (`enable.idempotence=true`): Deduplicates producer retries using sequence numbers.
2. **Transactional API**: Atomically produces to multiple partitions and/or commits consumer offsets.
3. **Read committed isolation** on consumer (`isolation.level=read_committed`): Consumers only read committed transactions.

```java
// Exactly-once consume-transform-produce (Kafka Streams or manual)
@Transactional
public void processAndPublish(ConsumerRecord<String, OrderEvent> record) {
    // All of this is atomic:
    orderService.process(record.value());                    // DB write
    kafkaTemplate.send("processed-orders", transform(record.value())); // Kafka produce
    // Consumer offset committed as part of the transaction
}

// application.yml
spring:
  kafka:
    producer:
      acks: all
      properties:
        enable.idempotence: true
        transactional.id: service-tx-1
    consumer:
      isolation-level: read_committed
      enable-auto-commit: false
```

---

### How does Kafka's log compaction work? When would you use it?

**Core Explanation:**

**Log compaction** retains only the **latest value for each key** in a partition, discarding old values. Regular deletion: old segments deleted by time/size. Compaction: keeps latest per key, tombstones (null value) delete a key.

```
Before compaction:
[key=user1, value="Alice"] [key=user2, value="Bob"] [key=user1, value="Alicia"]

After compaction:
[key=user2, value="Bob"] [key=user1, value="Alicia"]
```

**Use cases:**
- **Change data capture (CDC)**: Keep current state of database rows.
- **User preferences**: Only need the latest preference value.
- **Configuration updates**: Only current config matters.
- **Event sourcing snapshots**: Latest state per entity.

```properties
# Enable on a topic
log.cleanup.policy=compact
log.retention.ms=-1  # infinite retention (compaction manages storage)
```

---

### What is Kafka's ISR (In-Sync Replicas) mechanism?

**Core Explanation:**

Each partition has a **leader** and zero or more **follower replicas**. The **ISR (In-Sync Replica set)** is the set of replicas that are fully caught up with the leader.

**How it works:**
1. Producer sends message to leader.
2. Leader writes to its log.
3. Followers fetch and replicate the message.
4. Once all ISR followers have replicated, the message is considered "committed."
5. `acks=all` means: wait for all ISR replicas to acknowledge.

**Failure handling:**
- If a follower falls behind (lag > `replica.lag.time.max.ms`), it's removed from ISR.
- If the leader fails, Kafka elects a new leader from the ISR.
- `min.insync.replicas=2`: Refuse writes if fewer than 2 replicas are in-sync → prevents data loss at the cost of availability.

```properties
# Durability configuration
acks=all                      # wait for all ISR replicas
min.insync.replicas=2         # at least 2 ISR replicas required
replication.factor=3          # 3 total replicas per partition
```

---

### What is the Outbox pattern with Kafka? How does it solve the dual-write problem?

**Core Explanation:**

**Dual-write problem:** If you save to DB AND publish to Kafka, either can fail independently → inconsistency.

```java
@Transactional
public Order placeOrder(CreateOrderRequest req) {
    // ATOMIC: both writes in same DB transaction
    Order order = orderRepository.save(new Order(req));

    outboxRepository.save(OutboxEvent.builder()
        .aggregateId(order.getId().toString())
        .aggregateType("Order")
        .eventType("OrderPlaced")
        .payload(serialize(order))
        .topic("orders")
        .build());

    return order;
    // If Kafka is down or the app crashes:
    // → DB transaction rolls back (both order and outbox event)
    // → OR DB commits, outbox event saved, publisher retries later
    // → No inconsistency possible
}

// Debezium (CDC) — reads DB transaction log, publishes to Kafka automatically
// No polling loop needed; captures all committed transactions
```

---

### How do you handle schema evolution without breaking consumers?

**Core Explanation:**

Use **Schema Registry** (Confluent) with Avro or Protobuf:

1. **Register schema**: Producer registers schema with registry.
2. **Schema ID in message**: Messages contain schema ID, not the full schema.
3. **Compatibility rules**: Schema Registry enforces backward/forward compatibility.

**Compatibility strategies:**
- **Backward compatible**: New consumers can read old messages. Add fields with defaults; never remove required fields.
- **Forward compatible**: Old consumers can read new messages. New consumers ignore unknown fields.
- **Full compatible**: Both backward and forward.

```avro
// v1 schema
{"type": "record", "name": "Order",
 "fields": [{"name": "id", "type": "long"},
             {"name": "status", "type": "string"}]}

// v2 schema — backward compatible (new field with default)
{"type": "record", "name": "Order",
 "fields": [{"name": "id", "type": "long"},
             {"name": "status", "type": "string"},
             {"name": "priority", "type": "string", "default": "NORMAL"}]}
// Old consumers: read v2 message, ignore "priority" field
// New consumers: read v1 message, "priority" defaults to "NORMAL"
```

```java
// Spring + Avro + Schema Registry
spring:
  kafka:
    producer:
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
    consumer:
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
    properties:
      schema.registry.url: http://schema-registry:8081
```
# Section 14: System Design

---

## Basic Questions

---

### What is the CAP theorem? Can you have all three?

**Core Explanation:**

**CAP theorem** states that a distributed system can provide at most **two** of the following three guarantees:

- **Consistency (C)**: Every read receives the most recent write (all nodes see the same data at the same time).
- **Availability (A)**: Every request receives a response (though not necessarily the latest data).
- **Partition Tolerance (P)**: System continues operating despite network partitions (message drops between nodes).

**Can you have all three?** No. In a distributed system, **network partitions are unavoidable** (P must always be chosen). So the real choice is:
- **CP** (Consistency + Partition Tolerance): When a partition occurs, refuse requests rather than return stale data. Examples: HBase, ZooKeeper, etcd.
- **AP** (Availability + Partition Tolerance): Respond with potentially stale data. Examples: Cassandra, CouchDB, DynamoDB (with eventual consistency).

```
Network partition example:
[Node A] ←✗→ [Node B]  (partition — they can't communicate)

CP system: Refuse reads/writes until partition heals (consistent but unavailable)
AP system: Both nodes accept reads/writes independently → may diverge (available but inconsistent)
```

---

### What is the difference between horizontal and vertical scaling?

**Core Explanation:**

| | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
|---|---|---|
| Method | Add more CPU/RAM/storage to existing server | Add more servers |
| Limit | Hardware limit — one server can only be so large | Near-infinite — add more nodes |
| Cost | Expensive (diminishing returns) | Cost-effective with commodity hardware |
| Single point of failure | Yes | Distributed — resilient |
| Complexity | Simple — no distributed concerns | Complex — load balancing, data consistency |
| Downtime | Usually requires restart | Add nodes without downtime |

**In practice:** Most modern systems start vertical (simpler) and move horizontal as load grows. Stateless application servers scale horizontally easily; stateful systems (databases) require sharding/replication for horizontal scaling.

---

### What is a load balancer? What are the common algorithms?

**Core Explanation:**

A **load balancer** distributes incoming requests across multiple backend servers to improve throughput, availability, and scalability.

**Algorithms:**
| Algorithm | Description | Best for |
|---|---|---|
| **Round-Robin** | Distribute requests in sequence | Equal-capacity servers |
| **Weighted Round-Robin** | More requests to higher-capacity servers | Mixed server capabilities |
| **Least Connections** | Route to server with fewest active connections | Long-running requests |
| **IP Hash** | Hash client IP → same server every time | Session affinity (stateful apps) |
| **Random** | Random server selection | Uniform servers, short requests |

**Layer 4 vs Layer 7:**
- **Layer 4 (transport)**: Route based on TCP/UDP — fast, no content inspection. (HAProxy, AWS NLB)
- **Layer 7 (application)**: Route based on HTTP headers, URL, content — smart routing. (Nginx, AWS ALB, Kong)

---

### What is caching? At which levels can you cache?

**Core Explanation:**

**Caching** stores frequently accessed data in a faster storage layer to reduce latency and load on the origin.

**Caching levels (fastest to slowest):**
1. **Client-side**: Browser cache, mobile app cache. HTTP `Cache-Control`, `ETag`.
2. **CDN**: Edge servers cache static content globally. AWS CloudFront, Cloudflare.
3. **API Gateway**: Cache API responses for identical requests.
4. **Application**: In-memory (HashMap, Caffeine) or distributed (Redis, Memcached).
5. **Database**: Query cache, result set cache. PostgreSQL shared buffers, MySQL InnoDB buffer pool.

---

## Intermediate Questions

---

### How do you design a URL shortener?

**Core Explanation:**

**Components:**
1. **Write API**: `POST /shorten {url: "https://..."}` → returns `{shortUrl: "https://short.ly/abc123"}`
2. **Read API**: `GET /{code}` → `301/302 Redirect` to original URL
3. **Database**: Store `code → original_url` mapping
4. **Hash generation**: Generate short unique codes

**Design decisions:**

```java
// Hash generation
public String generateCode(String url) {
    // Option 1: MD5/SHA hash, take first 6-8 chars
    String hash = DigestUtils.md5Hex(url + System.nanoTime()).substring(0, 8);

    // Option 2: Base62 encoding of auto-increment ID
    // ID 1234567 → "5Hla" (Base62: a-z, A-Z, 0-9)
    return base62Encode(idGenerator.nextId());
}

// Collision handling: check DB, retry with next ID
// Redirect: 301 (permanent, cached by browser) vs 302 (temporary, tracks clicks)

// Analytics: store click_count, user_agent, geo in separate table
// Cache hot URLs: Redis with TTL
// Rate limiting: prevent abuse

// Scale:
// Read-heavy (99% redirects, 1% shortens) → cache extensively
// Horizontally scale read replicas
// Partition by code hash for DB sharding
```

---

### How do you design an LRU cache?

**Core Explanation:**

LRU (Least Recently Used) evicts the least recently accessed item when capacity is full.

**Data structures:** `HashMap` + `Doubly Linked List`
- HashMap: O(1) key lookup.
- Doubly Linked List: O(1) move-to-head (on access) and O(1) remove-tail (on eviction).

```java
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null); // sentinel head (MRU)
    private final Node<K, V> tail = new Node<>(null, null); // sentinel tail (LRU)

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToHead(node); // mark as recently used
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);
        if (existing != null) {
            existing.value = value;
            moveToHead(existing);
            return;
        }
        Node<K, V> node = new Node<>(key, value);
        map.put(key, node);
        addToHead(node);

        if (map.size() > capacity) {
            Node<K, V> lru = tail.prev; // least recently used
            removeNode(lru);
            map.remove(lru.key);
        }
    }

    private void addToHead(Node<K, V> node) {
        node.prev = head; node.next = head.next;
        head.next.prev = node; head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next; node.next.prev = node.prev;
    }

    private void moveToHead(Node<K, V> node) { removeNode(node); addToHead(node); }
}
```

**Production:** Use `LinkedHashMap(capacity, 0.75f, true)` with `removeEldestEntry()` for simplicity. Use Redis for distributed LRU.

---

### What are caching strategies?

**Core Explanation:**

**Cache Aside (Lazy Loading):**
```java
// Read: check cache first, fall back to DB
User getUser(Long id) {
    User cached = redis.get("user:" + id);
    if (cached != null) return cached;
    User user = db.findById(id);
    redis.set("user:" + id, user, Duration.ofMinutes(10));
    return user;
}
// Write: invalidate cache (let next read reload)
void updateUser(User user) {
    db.save(user);
    redis.delete("user:" + user.getId()); // invalidate
}
```

**Write Through:**
```java
// Write to cache AND DB synchronously
void updateUser(User user) {
    db.save(user);                              // write to DB
    redis.set("user:" + user.getId(), user);   // write to cache
}
// Cache always fresh; more write latency
```

**Write Behind (Write Back):**
```java
// Write to cache, async write to DB
void updateUser(User user) {
    redis.set("user:" + user.getId(), user);  // write cache immediately
    writeQueue.add(user);                      // async DB write
}
// Fast writes; risk of data loss on cache failure
```

**Read Through:**
- Cache itself fetches from DB on miss (transparent to application).
- Library-level (e.g., JCache with read-through configured).

---

### What is cache invalidation? Why is it considered one of the hardest problems?

**Core Explanation:**

Cache invalidation — ensuring cached data is removed or updated when the source data changes — is hard because:

1. **Distributed state**: Cache and DB are separate systems; updates can fail partially.
2. **Race conditions**: Between invalidation and re-load, another thread may cache stale data.
3. **Cache stampede**: On invalidation, many threads simultaneously fetch from DB.
4. **Cascading invalidation**: One entity's update may require invalidating many related cache keys.

**Strategies:**
```java
// TTL-based: accept eventual consistency
redis.set("user:1", user, Duration.ofMinutes(5)); // auto-expires

// Event-driven invalidation
@TransactionalEventListener(phase = AFTER_COMMIT)
void onUserUpdated(UserUpdatedEvent event) {
    redis.delete("user:" + event.getUserId());
    redis.delete("user:list:*"); // pattern delete (careful — expensive!)
}

// Cache stampede prevention — mutex lock
User getUser(Long id) {
    User cached = redis.get("user:" + id);
    if (cached != null) return cached;

    String lockKey = "lock:user:" + id;
    if (redis.setIfAbsent(lockKey, "1", Duration.ofSeconds(5))) {
        try {
            User user = db.findById(id);
            redis.set("user:" + id, user, Duration.ofMinutes(10));
            return user;
        } finally {
            redis.delete(lockKey);
        }
    } else {
        Thread.sleep(100);
        return getUser(id); // retry — someone else is loading
    }
}
```

---

## Advanced Questions

---

### How do you design a scalable payment system?

**Core Explanation:**

**Key requirements:** Reliability (no lost payments), idempotency (no double charges), consistency, fraud detection, audit trail.

**Architecture:**
```
Client → API Gateway → Payment API → Payment Processor (Stripe/Adyen)
                              ↓
                         Orders DB (with Outbox table)
                              ↓ (CDC via Debezium)
                         Kafka: payment-events topic
                              ↓
                         Notification Service
                         Accounting Service
                         Analytics Service
```

**Key design decisions:**
```java
// 1. Idempotency key — prevent duplicate charges
@PostMapping("/payments")
public PaymentResponse charge(
        @RequestHeader("Idempotency-Key") String key,
        @RequestBody ChargeRequest request) {
    return idempotencyService.execute(key, () -> {
        Payment payment = paymentGateway.charge(request);
        paymentRepo.save(payment);
        return payment;
    });
}

// 2. Payment state machine
enum PaymentStatus { PENDING, PROCESSING, COMPLETED, FAILED, REFUNDED }

// 3. Optimistic locking on payment records
@Version Long version; // prevent concurrent state updates

// 4. Reconciliation job — compare internal records vs payment processor
@Scheduled(cron = "0 0 1 * * *") // nightly
void reconcile() { reconciliationService.reconcileWithProvider(); }

// 5. Circuit breaker around payment provider
@CircuitBreaker(name = "paymentProvider")
public PaymentResponse charge(ChargeRequest req) { return stripeClient.charge(req); }
```

---

### What is consistent hashing? How does it help in distributed caching?

**Core Explanation:**

**Problem:** With N servers, naive `hash(key) % N` routing breaks when adding/removing servers — almost all keys remap to different servers (cache misses).

**Consistent hashing:** Places servers and keys on a circular ring. Each key maps to the nearest server clockwise. Adding/removing a server only remaps `1/N` of keys.

```
Ring (0 to 2^32):
Server A at position 100
Server B at position 200
Server C at position 300

Key "user:1" hashes to position 150 → assigned to Server B (nearest clockwise)
Key "user:2" hashes to position 250 → assigned to Server C
Key "user:3" hashes to position 50  → assigned to Server A

Add Server D at position 175:
- Key "user:1" (pos 150) now goes to Server D instead of B
- Only keys between A (100) and D (175) are remapped — ~1/4 of total

Virtual nodes: Each server has 150 virtual positions on the ring → more even distribution
```

**Use in Redis Cluster:** Keys automatically distributed across shards using consistent hashing with hash slots (0-16383).

---

### How do you design a notification system for millions of users?

**Core Explanation:**

**Requirements:** Push notifications (mobile), email, SMS, real-time (WebSocket). Millions of users, some with device preferences.

**Architecture:**
```
Event source (order service, etc.)
    ↓
Kafka: notification-events topic (partitioned by user_id)
    ↓
Notification Service (consumer)
    ↓ fan-out per channel
[Push Worker] → FCM/APNS (mobile push)
[Email Worker] → SendGrid/SES
[SMS Worker] → Twilio
[WebSocket Worker] → WebSocket server (real-time)
```

```java
// Notification preference routing
@KafkaListener(topics = "notification-events")
public void processNotification(NotificationEvent event) {
    User user = userService.findById(event.getUserId());
    NotificationPrefs prefs = prefsService.getPrefs(user.getId());

    if (prefs.isPushEnabled() && user.hasDeviceToken()) {
        pushWorker.send(user.getDeviceToken(), event);
    }
    if (prefs.isEmailEnabled()) {
        emailWorker.send(user.getEmail(), event);
    }
    // Async, parallel workers — don't block on slow channels
}

// WebSocket fan-out
// Use Redis pub/sub: when user is online, their WebSocket server receives via Redis channel
// Horizontally scalable: any server can push to any user via Redis

// Rate limiting per user: max 5 notifications/hour
// DLQ for failed deliveries
// Retry with exponential backoff
```

---

### How do you handle thundering herd / cache stampede problems?

**Core Explanation:**

**Cache stampede**: When a popular cache entry expires, thousands of requests simultaneously go to the DB.

**Solutions:**

1. **Mutex/Lock**: Only one request refreshes; others wait.
```java
// Only one thread regenerates; others get lock and see cached value
String value = cache.get(key);
if (value == null) {
    if (lock.tryLock(key, 1, TimeUnit.SECONDS)) {
        try {
            value = db.fetch(key); // only one thread does this
            cache.set(key, value, ttl);
        } finally {
            lock.unlock(key);
        }
    } else {
        value = cache.get(key); // others wait and re-read
    }
}
```

2. **Probabilistic Early Expiration (PER)**:
```java
// Refresh cache slightly before TTL expires based on probability
// More likely to refresh as TTL approaches zero
double beta = 1.0;
double delta = System.currentTimeMillis() - lastComputeTime; // time to compute
double remainingTtl = ttl - System.currentTimeMillis();
if (-beta * delta * Math.log(Math.random()) >= remainingTtl) {
    refreshCache(key); // refresh early with increasing probability as TTL approaches
}
```

3. **Background refresh**: Asynchronously refresh before expiry (stale-while-revalidate).
```java
// Serve stale data while refreshing in background
if (cache.isExpiringSoon(key, Duration.ofSeconds(10))) {
    asyncExecutor.submit(() -> refreshCache(key)); // don't block request
}
return cache.get(key); // return current (possibly slightly stale) value
```

---

### How do you design a distributed chat system?

**Core Explanation:**

**Requirements:** Real-time messages, presence (online/offline), message ordering, history, group chats.

**Architecture:**
```
Client (WebSocket) → Chat Server cluster → Message Broker (Kafka)
                                       ↓
                               Message DB (Cassandra - append-heavy)
                               Presence Service (Redis - TTL-based)
```

**Key design decisions:**
```java
// WebSocket server with Redis pub/sub fan-out
// User connects to any Chat Server instance
// Each Chat Server subscribes to Redis channel for that user's rooms

// Message ordering: use unique monotonic IDs
// Cassandra table: (room_id, message_id) → message_content
// Partition by room_id — all messages for a room on same partition
// Cluster by message_id DESC — recent messages first

// Presence: heartbeat every 30s, Redis key with 60s TTL
// If heartbeat stops → key expires → user appears offline

// Message delivery guarantee:
// ACK from receiver → message marked delivered
// If receiver offline → buffer in Kafka → deliver when reconnects

// Group chat fan-out:
// Write once to Kafka partition for the group
// Consumer delivers to each member's WebSocket or notification
```

---

### How do you identify and fix performance bottlenecks in a Java backend system?

**Core Explanation:**

**Systematic approach:**

```
1. Identify the bottleneck:
   - High latency → profile with JFR/async-profiler
   - High CPU → CPU flame graph (hot methods)
   - High memory → heap dump + MAT
   - I/O bound → check DB slow query log, Kafka lag

2. Common bottlenecks and fixes:

   N+1 queries:
   - Symptom: High DB query count, TTFB high
   - Fix: JOIN FETCH, EntityGraph, batch loading

   Connection pool exhaustion:
   - Symptom: HikariCP timeout, "Connection not available" errors
   - Fix: Tune pool size, reduce tx duration, fix missing @Transactional(readOnly=true)

   GC pressure:
   - Symptom: High GC pause % in JFR, OOM
   - Fix: Reduce allocations in hot paths, use primitives, tune heap

   Inefficient serialization:
   - Symptom: High CPU on Jackson operations
   - Fix: Reuse ObjectMapper, use streaming API, pre-compute serialization

   Blocking I/O in async handlers:
   - Symptom: Thread pool exhaustion in WebFlux app
   - Fix: Move blocking calls to bounded elastic scheduler
```

```java
// JFR profiling (Java Flight Recorder)
java -XX:+FlightRecorder -XX:StartFlightRecording=duration=60s,filename=app.jfr ...
// Analyze with JDK Mission Control

// Async-profiler — CPU flame graph
./profiler.sh -d 30 -f flamegraph.html PID
```
# Sections 15 & 16: DevOps & Cloud + Testing

---

# Section 15: DevOps & Cloud

---

## Basic Questions

---

### What is Docker? What is the difference between a Docker image and a container?

**Core Explanation:**

**Docker** is a platform for building, shipping, and running applications in **containers** — lightweight, isolated environments.

| | Docker Image | Docker Container |
|---|---|---|
| Definition | Read-only blueprint (layers of filesystem changes) | Running instance of an image |
| State | Immutable | Mutable (runtime state) |
| Stored | Registry (Docker Hub, ECR) | Host machine |
| Analogy | Class definition | Object instance |

```bash
# Image: built from Dockerfile
docker build -t my-app:1.0 .
docker images  # list images

# Container: running instance
docker run -d -p 8080:8080 --name app my-app:1.0
docker ps      # list running containers
docker stop app
docker rm app
```

---

### What is the difference between Docker and a virtual machine?

**Core Explanation:**

| | Docker (Container) | Virtual Machine |
|---|---|---|
| Isolation | Process-level (shares host OS kernel) | Full OS isolation |
| OS overhead | None — no guest OS | Full guest OS per VM |
| Startup time | Milliseconds | Minutes |
| Size | Tens of MB | GB |
| Performance | Near-native | Some overhead (hypervisor) |
| Security | Weaker isolation (shared kernel) | Stronger (separate kernel) |
| Use case | Microservices, CI/CD | Multi-OS, strong isolation requirements |

---

### What is Kubernetes? Why is it used?

**Core Explanation:**

**Kubernetes (K8s)** is a container orchestration platform that automates:
- **Deployment**: Deploy containers across a cluster of nodes.
- **Scaling**: Scale up/down automatically (HPA/VPA).
- **Self-healing**: Restart failed containers, replace unhealthy pods.
- **Load balancing**: Distribute traffic to healthy pods.
- **Rolling updates**: Deploy new versions without downtime.
- **Service discovery**: DNS-based service-to-service communication.
- **Secret management**: Encrypted storage of credentials.

**Core concepts:**
- **Pod**: Smallest deployable unit (one or more containers).
- **Deployment**: Manages desired state of pods (replica count, image version).
- **Service**: Stable network endpoint for a set of pods.
- **Namespace**: Virtual cluster within a cluster.

---

### What is the difference between blue-green and canary deployments?

**Core Explanation:**

**Blue-Green:** Two identical environments (blue = current, green = new). Traffic switches 100% from blue to green. Instant rollback by switching back.

**Canary:** New version receives a small % of traffic (5-10%), gradually increased as confidence grows. Slower rollback.

**Rolling:** Replace pods one at a time. No extra environment needed. Rollback requires downward rolling update.

---

## Intermediate Questions

---

### How do you Dockerize a Spring Boot application? Write a sample Dockerfile.

**Core Explanation:**

**Multi-stage Dockerfile (best practice):**
```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Extract layers (Spring Boot layertools)
FROM eclipse-temurin:21-jdk-alpine AS extractor
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 3: Runtime — minimal image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy layers in order of change frequency (most stable first)
COPY --from=extractor /app/dependencies/ ./
COPY --from=extractor /app/spring-boot-loader/ ./
COPY --from=extractor /app/snapshot-dependencies/ ./
COPY --from=extractor /app/application/ ./

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.JarLauncher"]
```

**Key best practices:**
- Multi-stage build: final image doesn't include JDK or build tools.
- Non-root user for security.
- Layered JARs: Docker caches dependency layer (rarely changes), only re-pushes application layer.
- `UseContainerSupport`: JVM respects container memory limits.

---

### What are liveness and readiness probes? How do they differ?

**Core Explanation:**

| | Liveness Probe | Readiness Probe |
|---|---|---|
| Purpose | Is the app alive? (should it be restarted?) | Is the app ready to receive traffic? |
| Failure action | Kubernetes restarts the pod | Kubernetes removes pod from Service endpoints |
| Use case | Detect deadlock, hung app | Wait for startup, circuit breaker open |

```yaml
# Kubernetes deployment spec
spec:
  containers:
  - name: order-service
    image: order-service:1.2
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      initialDelaySeconds: 60  # wait for JVM startup
      periodSeconds: 10
      failureThreshold: 3       # restart after 3 consecutive failures

    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 5
      failureThreshold: 3       # stop routing traffic after 3 failures
```

```yaml
# Spring Boot Actuator health groups
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, redis
```

---

### How does Kubernetes handle auto-scaling (HPA, VPA)?

**Core Explanation:**

**HPA (Horizontal Pod Autoscaler):** Scales pod count based on metrics (CPU, memory, custom).
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60  # scale up when avg CPU > 60%
  - type: Pods
    pods:
      metric:
        name: kafka_consumer_lag  # custom metric from Prometheus
      target:
        type: AverageValue
        averageValue: "1000"
```

**VPA (Vertical Pod Autoscaler):** Adjusts CPU/memory requests per pod. Doesn't add pods — optimizes resource allocation. Requires restart when adjusting (limitation).

---

### How do you secure secrets in cloud environments?

**Core Explanation:**

```yaml
# BAD: Hardcoded in environment variables (visible in K8s dashboard)
env:
- name: DB_PASSWORD
  value: "mysupersecretpassword"  # NEVER!

# GOOD option 1: Kubernetes Secrets (base64, not encrypted by default)
kubectl create secret generic db-secret --from-literal=password=mysecret
# In pod spec:
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password

# GOOD option 2: External Secrets Operator + AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  secretStoreRef:
    name: aws-secretsmanager
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: prod/db/password  # pulled from AWS Secrets Manager

# GOOD option 3: HashiCorp Vault + Vault Agent Sidecar
# Vault injects secrets as files or env vars after authentication via Kubernetes SA token
```

**Best practices:**
- Encrypt K8s secrets at rest (etcd encryption).
- Use AWS Secrets Manager / GCP Secret Manager for managed secret rotation.
- Never log secrets; use environment variables, not config files.
- Rotate secrets regularly; use IAM roles (no long-lived credentials).

---

### What is the difference between AWS EC2, ECS, EKS, and Lambda?

**Core Explanation:**

| Service | Abstraction | Best for |
|---|---|---|
| **EC2** | Virtual machines | Full control, stateful apps, custom OS |
| **ECS** | Container orchestration (AWS-native) | Simpler K8s alternative, Fargate serverless containers |
| **EKS** | Managed Kubernetes | Kubernetes workloads, portable, multi-cloud |
| **Lambda** | Serverless functions | Event-driven, short-lived tasks, no traffic mgmt |

**Decision tree:**
- Event-driven, <15min execution, variable load → **Lambda**
- Container, no K8s expertise, AWS-native → **ECS Fargate**
- Kubernetes, portability, complex orchestration → **EKS**
- Custom OS, persistent workloads, specialized hardware → **EC2**

---

# Section 16: Testing

---

## Basic Questions

---

### What is the difference between unit tests and integration tests?

**Core Explanation:**

| | Unit Tests | Integration Tests |
|---|---|---|
| Scope | Single class/method in isolation | Multiple components working together |
| Dependencies | Mocked (Mockito) | Real (DB, messaging, services) |
| Speed | Very fast (milliseconds) | Slower (seconds to minutes) |
| Count | Many (pyramid base) | Moderate |
| Failure diagnosis | Pinpoint specific method | Broader — harder to diagnose |
| Tools | JUnit, Mockito | Spring Boot Test, Testcontainers |

```java
// Unit test — mock everything external
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;
    @Mock EmailService emailService;
    @InjectMocks OrderService service;

    @Test
    void shouldPlaceOrder() {
        Order order = new Order("P1", 2);
        when(repo.save(any())).thenReturn(order);
        Order result = service.placeOrder(new CreateOrderRequest("P1", 2));
        verify(emailService).sendConfirmation(result);
        assertThat(result.getStatus()).isEqualTo("PENDING");
    }
}

// Integration test — real Spring context, real DB
@SpringBootTest
@Transactional
class OrderServiceIntegrationTest {
    @Autowired OrderService service;
    @Autowired OrderRepository repo;

    @Test
    void shouldPersistOrderToDatabase() {
        Order result = service.placeOrder(new CreateOrderRequest("P1", 2));
        Order fromDb = repo.findById(result.getId()).orElseThrow();
        assertThat(fromDb.getStatus()).isEqualTo("PENDING");
    }
}
```

---

### What is Mockito? What is the difference between `@Mock` and `@InjectMocks`?

**Core Explanation:**

**Mockito** creates mock objects that replace real dependencies in unit tests, allowing controlled behavior.

- **`@Mock`**: Creates a mock of a class/interface. All methods return default values (0, null, empty collections) unless stubbed.
- **`@InjectMocks`**: Creates an instance of the class under test and injects all `@Mock` fields into it (via constructor, setter, or field injection).

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {
    @Mock PaymentGateway gateway;    // mock — no real network calls
    @Mock AuditRepository auditRepo; // mock

    @InjectMocks PaymentService service; // real PaymentService with mocked deps injected

    @Test
    void shouldChargeSuccessfully() {
        // Arrange
        when(gateway.charge("user1", 100.0))
            .thenReturn(new ChargeResult("txn123", "SUCCESS"));

        // Act
        PaymentResult result = service.processPayment("user1", 100.0);

        // Assert
        assertThat(result.getTransactionId()).isEqualTo("txn123");
        verify(auditRepo).save(any(AuditLog.class)); // verify interaction
    }

    @Test
    void shouldHandleGatewayFailure() {
        when(gateway.charge(any(), anyDouble()))
            .thenThrow(new GatewayException("Network error"));

        assertThatThrownBy(() -> service.processPayment("user1", 100.0))
            .isInstanceOf(PaymentException.class)
            .hasMessageContaining("Payment failed");
    }
}
```

---

### What is the difference between `@MockBean` and `@Mock`?

**Core Explanation:**

| | `@Mock` | `@MockBean` |
|---|---|---|
| Framework | Mockito | Spring Test |
| Context | No Spring context needed | Requires Spring context |
| Scope | Single test class | Spring `ApplicationContext` |
| Bean replacement | No — bypasses Spring | Replaces bean in Spring context |
| Use with | `@ExtendWith(MockitoExtension.class)` | `@SpringBootTest`, `@WebMvcTest` |

```java
// @MockBean — use in Spring integration tests
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService; // replaces the real bean in Spring context

    @Test
    void shouldReturnOrder() throws Exception {
        when(orderService.findById(1L)).thenReturn(new OrderDTO(1L, "PENDING"));

        mockMvc.perform(get("/api/orders/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("PENDING"));
    }
}
```

---

## Intermediate Questions

---

### What is Testcontainers? When would you use it over H2?

**Core Explanation:**

**Testcontainers** starts real Docker containers (PostgreSQL, MySQL, Kafka, Redis) during tests, providing a production-identical environment.

**Why prefer over H2:**
- H2 has different SQL dialect, different behavior for constraints, window functions, triggers.
- Real bugs only appear on the real database.
- Testcontainers tests catch: dialect differences, index behavior, deadlocks, connection pool issues.

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired OrderRepository repo;

    @Test
    void shouldFindOrdersByStatus() {
        repo.save(new Order("PENDING"));
        repo.save(new Order("SHIPPED"));

        List<Order> pending = repo.findByStatus("PENDING");
        assertThat(pending).hasSize(1);
    }
}

// Shared container across tests (faster)
@TestConfiguration
class TestDatabaseConfig {
    @Bean
    @ServiceConnection // Spring Boot 3.1+ — auto-configures datasource
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16");
    }
}
```

---

### What is the test pyramid? How do you balance unit, integration, and E2E tests?

**Core Explanation:**

```
          /\
         /  \
        / E2E \           Few — expensive, slow, brittle
       /--------\
      / Integration\       Moderate — real components, medium speed
     /--------------\
    /   Unit Tests    \    Many — fast, cheap, isolated
   /------------------\
```

**Guidelines:**
- **Unit (70%)**: Pure logic, algorithms, transformations — no I/O.
- **Integration (20%)**: Repository + DB, Controller + Service, Kafka producers/consumers.
- **E2E (10%)**: Critical user journeys (signup, checkout, payment).

**Pitfalls:**
- Inverted pyramid (too many E2E): Slow, flaky, expensive CI.
- Testing pyramid but not coverage (good numbers, poor quality): Tests don't catch bugs.

---

### How do you test REST APIs (MockMvc, RestAssured)?

**Core Explanation:**

```java
// MockMvc — lightweight, no running server
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @MockBean OrderService service;

    @Test
    void shouldCreateOrder() throws Exception {
        CreateOrderRequest req = new CreateOrderRequest("P1", 2);
        OrderDTO response = new OrderDTO(100L, "PENDING");
        when(service.create(any())).thenReturn(response);

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", containsString("/orders/100")))
            .andExpect(jsonPath("$.id").value(100))
            .andExpect(jsonPath("$.status").value("PENDING"));
    }

    @Test
    void shouldReturnValidationErrorForInvalidRequest() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"productId\": null, \"quantity\": -1}"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
    }
}
```

---

## Advanced Questions

---

### What is contract testing (Pact, Spring Cloud Contract)? How does it prevent integration failures?

**Core Explanation:**

**Contract testing** verifies that consumer expectations and provider implementations agree — without deploying both services simultaneously.

**Problem:** Service A (consumer) calls Service B (provider). If B changes its API, A breaks — but you only discover this in E2E tests or production.

**Solution:**
1. Consumer writes a **pact** (contract): "I expect GET /users/1 to return `{id, name, email}`"
2. Pact is published to **Pact Broker**.
3. Provider tests verify it honors all published pacts.
4. CI fails if provider breaks a consumer contract.

```java
// Consumer side (Pact test)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "user-service")
class UserServicePactTest {

    @Pact(consumer = "order-service")
    public RequestResponsePact getUserPact(PactDslWithProvider builder) {
        return builder
            .given("user 1 exists")
            .uponReceiving("GET user by id")
                .path("/users/1").method("GET")
            .willRespondWith()
                .status(200)
                .body(new PactDslJsonBody()
                    .numberType("id", 1)
                    .stringType("name", "Alice")
                    .stringType("email", "alice@example.com"))
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getUserPact")
    void shouldFetchUser(MockServer mockServer) {
        UserClient client = new UserClient(mockServer.getUrl());
        UserDTO user = client.findById(1L);
        assertThat(user.getName()).isEqualTo("Alice");
    }
}

// Provider side — verify contract
@Provider("user-service")
@PactBroker(url = "https://your-pact-broker.io")
class UserServicePactVerificationTest {
    @TestTemplate @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }
}
```

---

### How do you test Kafka producers and consumers?

**Core Explanation:**

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"orders"})
class OrderKafkaTest {

    @Autowired KafkaTemplate<String, OrderEvent> kafkaTemplate;
    @Autowired OrderEventConsumer consumer;

    @Test
    void shouldConsumeOrderEvent() throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(1);
        consumer.setLatch(latch);

        OrderEvent event = new OrderEvent("ORDER-001", "PLACED");
        kafkaTemplate.send("orders", "ORDER-001", event).get();

        boolean consumed = latch.await(10, TimeUnit.SECONDS);
        assertThat(consumed).isTrue();
        assertThat(consumer.getLastEvent().getOrderId()).isEqualTo("ORDER-001");
    }
}

// Consumer with test latch
@Component
class OrderEventConsumer {
    private CountDownLatch latch;
    private OrderEvent lastEvent;

    @KafkaListener(topics = "orders")
    public void consume(OrderEvent event) {
        this.lastEvent = event;
        if (latch != null) latch.countDown();
    }
    // setters...
}

// With Testcontainers Kafka
@Container
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

@DynamicPropertySource
static void kafkaProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
}
```

---

### What is mutation testing? How does it improve test quality beyond code coverage?

**Core Explanation:**

**Mutation testing** automatically introduces small bugs ("mutations") into your code (change `>` to `>=`, flip `true` to `false`, delete a line) and runs tests. If a test catches the mutation → test is effective ("killed"). If not → test is weak ("survived").

**Why code coverage alone is insufficient:**
```java
// 100% coverage but weak test
public boolean isAdult(int age) { return age >= 18; }

@Test
void testIsAdult() {
    assertThat(isAdult(25)).isTrue(); // passes! 100% coverage
    // But: mutation age >= 18 → age > 18 → test still passes with 25!
    // Better test:
    assertThat(isAdult(18)).isTrue();  // boundary
    assertThat(isAdult(17)).isFalse(); // boundary - 1
}
```

**PITest (PIT)** — Java mutation testing framework:
```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <configuration>
        <targetClasses>com.example.service.*</targetClasses>
        <targetTests>com.example.*Test</targetTests>
        <mutationThreshold>75</mutationThreshold> <!-- fail if mutation score < 75% -->
    </configuration>
</plugin>
```

---

### What are the best practices for writing maintainable test suites?

**Core Explanation:**

1. **Arrange-Act-Assert (AAA)**: Clear structure in every test.
2. **One assertion concept per test**: Tests that fail for one clear reason.
3. **Descriptive names**: `shouldThrowExceptionWhenQuantityIsNegative` not `testOrder`.
4. **No test interdependence**: Tests must be runnable in any order.
5. **Use test fixtures**: `@BeforeEach` for common setup; test data builders for complex objects.
6. **Fast test suite**: Separate slow integration tests (`@Tag("integration")`) from fast unit tests.
7. **Test behavior, not implementation**: Don't test private methods or internals — test via public API.
8. **Avoid over-mocking**: Too many mocks → tests don't actually test the system.

```java
// Builder pattern for test data
class OrderTestBuilder {
    private String productId = "P001";
    private int quantity = 1;
    private String status = "PENDING";

    static OrderTestBuilder anOrder() { return new OrderTestBuilder(); }
    OrderTestBuilder withProductId(String id) { this.productId = id; return this; }
    OrderTestBuilder withQuantity(int qty) { this.quantity = qty; return this; }
    Order build() { return new Order(productId, quantity, status); }
}

// Clean test using builder
@Test
void shouldRejectOrderWithNegativeQuantity() {
    Order order = anOrder().withQuantity(-1).build();
    assertThatThrownBy(() -> orderService.validate(order))
        .isInstanceOf(ValidationException.class)
        .hasMessageContaining("Quantity must be positive");
}
```
# Section 17: Coding & Problem Solving

---

## Data Structures & Algorithms

---

### Find the second highest number in an array (single pass)

```java
public static int findSecondHighest(int[] arr) {
    int first = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int n : arr) {
        if (n > first) { second = first; first = n; }
        else if (n > second && n != first) { second = n; }
    }
    if (second == Integer.MIN_VALUE) throw new IllegalArgumentException("No second highest");
    return second;
}
// Time: O(n), Space: O(1)
// Test: [5,1,4,2,3] → 4;  [5,5,3] → 3;  [5] → exception
```

---

### Find the first non-repeating character in a string

```java
public static char firstNonRepeating(String s) {
    Map<Character, Integer> freq = new LinkedHashMap<>(); // preserve insertion order
    for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);
    return freq.entrySet().stream()
        .filter(e -> e.getValue() == 1)
        .map(Map.Entry::getKey)
        .findFirst()
        .orElse('\0');
}
// Time: O(n), Space: O(1) — at most 26 chars for lowercase letters
// "aabbcd" → 'c';  "aabb" → '\0'
```

---

### Find the longest substring without repeating characters

```java
public static int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> charIndex = new HashMap<>();
    int maxLen = 0, left = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (charIndex.containsKey(c)) {
            left = Math.max(left, charIndex.get(c) + 1); // shrink window
        }
        charIndex.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// Sliding window — Time: O(n), Space: O(min(n, charset))
// "abcabcbb" → 3 ("abc");  "bbbbb" → 1;  "pwwkew" → 3 ("wke")
```

---

### Check if a number is a palindrome without converting to String

```java
public static boolean isPalindrome(int x) {
    if (x < 0 || (x % 10 == 0 && x != 0)) return false;
    int reversed = 0;
    while (x > reversed) {
        reversed = reversed * 10 + x % 10;
        x /= 10;
    }
    return x == reversed || x == reversed / 10; // handle odd-length numbers
}
// Time: O(log n), Space: O(1)
// 121 → true;  123 → false;  10 → false;  -121 → false
```

---

### Find the maximum sum subarray (Kadane's Algorithm)

```java
public static int maxSubarraySum(int[] arr) {
    int maxSum = arr[0], currentSum = arr[0];
    for (int i = 1; i < arr.length; i++) {
        currentSum = Math.max(arr[i], currentSum + arr[i]); // extend or restart
        maxSum = Math.max(maxSum, currentSum);
    }
    return maxSum;
}
// Time: O(n), Space: O(1)
// [-2,1,-3,4,-1,2,1,-5,4] → 6 (subarray [4,-1,2,1])

// Bonus: also return the subarray
public static int[] maxSubarrayWithIndices(int[] arr) {
    int maxSum = arr[0], currentSum = arr[0];
    int start = 0, end = 0, tempStart = 0;
    for (int i = 1; i < arr.length; i++) {
        if (arr[i] > currentSum + arr[i]) {
            currentSum = arr[i]; tempStart = i;
        } else {
            currentSum += arr[i];
        }
        if (currentSum > maxSum) {
            maxSum = currentSum; start = tempStart; end = i;
        }
    }
    return Arrays.copyOfRange(arr, start, end + 1);
}
```

---

### Find the maximum sum of K consecutive elements (sliding window)

```java
public static int maxSumKWindow(int[] arr, int k) {
    if (arr.length < k) throw new IllegalArgumentException("Array smaller than window");
    int windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += arr[i]; // initial window
    int maxSum = windowSum;
    for (int i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k]; // slide: add new, remove old
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
// Time: O(n), Space: O(1)
// arr=[1,4,2,10,2,3,1,0,20], k=4 → 24 ([2,10,2,3] or [3,1,0,20])
```

---

### Find all pairs whose sum equals a target (Two Sum)

```java
public static List<int[]> twoSum(int[] arr, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    List<int[]> result = new ArrayList<>();
    for (int i = 0; i < arr.length; i++) {
        int complement = target - arr[i];
        if (map.containsKey(complement)) {
            result.add(new int[]{arr[map.get(complement)], arr[i]});
        }
        map.put(arr[i], i);
    }
    return result;
}
// Time: O(n), Space: O(n)
// arr=[2,7,11,15], target=9 → [[2,7]]
```

---

### Find all triplets whose sum equals a target

```java
public static List<List<Integer>> threeSum(int[] arr, int target) {
    Arrays.sort(arr);
    List<List<Integer>> result = new ArrayList<>();
    for (int i = 0; i < arr.length - 2; i++) {
        if (i > 0 && arr[i] == arr[i-1]) continue; // skip duplicates
        int left = i + 1, right = arr.length - 1;
        while (left < right) {
            int sum = arr[i] + arr[left] + arr[right];
            if (sum == target) {
                result.add(List.of(arr[i], arr[left], arr[right]));
                while (left < right && arr[left] == arr[left+1]) left++;
                while (left < right && arr[right] == arr[right-1]) right--;
                left++; right--;
            } else if (sum < target) left++;
            else right--;
        }
    }
    return result;
}
// Time: O(n²), Space: O(1) excluding output
```

---

### Reverse a linked list iteratively and recursively

```java
class ListNode { int val; ListNode next; ListNode(int v) { val = v; } }

// Iterative — O(n) time, O(1) space
public static ListNode reverseIterative(ListNode head) {
    ListNode prev = null, curr = head;
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}

// Recursive — O(n) time, O(n) space (call stack)
public static ListNode reverseRecursive(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode newHead = reverseRecursive(head.next); // reverse rest
    head.next.next = head; // point next node back to current
    head.next = null;      // break forward link
    return newHead;
}
```

---

### Detect a cycle in a linked list (Floyd's algorithm)

```java
public static boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;         // move 1 step
        fast = fast.next.next;    // move 2 steps
        if (slow == fast) return true; // cycle detected
    }
    return false;
}
// Time: O(n), Space: O(1)

// Bonus: find start of cycle
public static ListNode findCycleStart(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next; fast = fast.next.next;
        if (slow == fast) { // cycle exists
            slow = head; // reset one pointer to head
            while (slow != fast) { slow = slow.next; fast = fast.next; }
            return slow; // meeting point is cycle start
        }
    }
    return null;
}
```

---

### Implement binary search

```java
public static int binarySearch(int[] sorted, int target) {
    int left = 0, right = sorted.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // avoid overflow: (left+right)/2
        if (sorted[mid] == target) return mid;
        else if (sorted[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1; // not found
}
// Time: O(log n), Space: O(1)

// Find first occurrence (leftmost)
public static int binarySearchFirst(int[] sorted, int target) {
    int left = 0, right = sorted.length - 1, result = -1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (sorted[mid] == target) { result = mid; right = mid - 1; } // keep searching left
        else if (sorted[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return result;
}
```

---

### Find the top K frequent elements

```java
public static int[] topKFrequent(int[] nums, int k) {
    // Count frequencies
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    // Min-heap of size k
    PriorityQueue<Map.Entry<Integer, Integer>> pq =
        new PriorityQueue<>(Comparator.comparingInt(Map.Entry::getValue));

    for (Map.Entry<Integer, Integer> entry : freq.entrySet()) {
        pq.offer(entry);
        if (pq.size() > k) pq.poll(); // remove least frequent
    }

    return pq.stream().mapToInt(Map.Entry::getKey).toArray();
}
// Time: O(n log k), Space: O(n)
// [1,1,1,2,2,3], k=2 → [1,2]
```

---

### Implement an LRU cache

```java
public class LRUCache {
    private final int capacity;
    private final Map<Integer, Integer> map;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        // access-order=true: most recently used at tail
        this.map = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;
            }
        };
    }

    public int get(int key) {
        return map.getOrDefault(key, -1);
    }

    public void put(int key, int value) {
        map.put(key, value);
    }
}
```

---

### Implement the Producer-Consumer problem using `BlockingQueue`

```java
public class ProducerConsumerExample {
    static final BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

    static class Producer implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    queue.put(i); // blocks if queue is full
                    System.out.println("Produced: " + i);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }

    static class Consumer implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    int item = queue.take(); // blocks if queue is empty
                    System.out.println("Consumed: " + item);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new Producer()).start();
        new Thread(new Consumer()).start();
    }
}
```

---

### Implement a thread-safe counter using `AtomicInteger`

```java
public class ThreadSafeCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() { count.incrementAndGet(); }
    public void decrement() { count.decrementAndGet(); }
    public int get() { return count.get(); }

    // For high contention — LongAdder is faster
    private final LongAdder adder = new LongAdder();
    public void add(long n) { adder.add(n); }
    public long total() { return adder.sum(); }
}
```

---

### Design a custom `Queue` using two stacks

```java
public class QueueFromTwoStacks<T> {
    private final Deque<T> inStack = new ArrayDeque<>();
    private final Deque<T> outStack = new ArrayDeque<>();

    public void enqueue(T item) {
        inStack.push(item); // always push to inStack
    }

    public T dequeue() {
        if (outStack.isEmpty()) {
            if (inStack.isEmpty()) throw new NoSuchElementException();
            while (!inStack.isEmpty()) outStack.push(inStack.pop()); // transfer
        }
        return outStack.pop();
    }

    public T peek() {
        if (outStack.isEmpty()) {
            while (!inStack.isEmpty()) outStack.push(inStack.pop());
        }
        return outStack.peek();
    }
}
// Amortized O(1) per operation — each element moved at most once from inStack to outStack
```

---

## Java Stream API Coding

---

### Find the second highest salary

```java
record Employee(String name, String department, double salary) {}

Optional<Double> secondHighest = employees.stream()
    .mapToDouble(Employee::salary)
    .boxed()
    .distinct()
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst();
```

---

### Group employees by department

```java
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department));
```

---

### Find the highest-paid employee in each department

```java
Map<String, Optional<Employee>> topEarnerByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.maxBy(Comparator.comparingDouble(Employee::salary))
    ));

// Without Optional:
Map<String, Employee> topEarner = employees.stream()
    .collect(Collectors.toMap(
        Employee::department,
        e -> e,
        BinaryOperator.maxBy(Comparator.comparingDouble(Employee::salary))
    ));
```

---

### Filter the top 3 highest salaries per department

```java
Map<String, List<Employee>> top3ByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.collectingAndThen(
            Collectors.toList(),
            list -> list.stream()
                .sorted(Comparator.comparingDouble(Employee::salary).reversed())
                .limit(3)
                .collect(Collectors.toList())
        )
    ));
```

---

### Count word occurrences in a sentence

```java
String sentence = "the cat sat on the mat";
Map<String, Long> wordCount = Arrays.stream(sentence.split("\\s+"))
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
// {the=2, cat=1, sat=1, on=1, mat=1}
```

---

### Calculate average, min, and max salary using IntSummaryStatistics

```java
DoubleSummaryStatistics stats = employees.stream()
    .mapToDouble(Employee::salary)
    .summaryStatistics();
System.out.println("Min: " + stats.getMin());
System.out.println("Max: " + stats.getMax());
System.out.println("Avg: " + stats.getAverage());
System.out.println("Count: " + stats.getCount());
System.out.println("Sum: " + stats.getSum());
```

---

### Partition a list into even and odd numbers

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6);
Map<Boolean, List<Integer>> partitioned = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
List<Integer> evens = partitioned.get(true);   // [2, 4, 6]
List<Integer> odds = partitioned.get(false);   // [1, 3, 5]
```

---

### Flatten a List<List<String>> into List<String>

```java
List<List<String>> nested = List.of(List.of("a","b"), List.of("c","d"), List.of("e"));
List<String> flat = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList());
// ["a","b","c","d","e"]
```

---

### Convert List<Employee> to Map<Integer, String> (id -> name) handling duplicates

```java
Map<Long, String> idToName = employees.stream()
    .collect(Collectors.toMap(
        Employee::id,
        Employee::name,
        (existing, duplicate) -> existing // keep first on duplicate
    ));
```

---

### Sort employees by salary descending, then name ascending

```java
List<Employee> sorted = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::salary).reversed()
                      .thenComparing(Employee::name))
    .collect(Collectors.toList());
```

---

### Remove duplicates from a list while preserving order

```java
List<String> list = List.of("a", "b", "a", "c", "b");
List<String> deduplicated = list.stream()
    .distinct() // maintains encounter order
    .collect(Collectors.toList());
// ["a","b","c"]
```

---

### Count characters and return Map<Character, Long>

```java
String s = "hello world";
Map<Character, Long> charCount = s.chars()
    .filter(c -> c != ' ')
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
// {h=1, e=1, l=3, o=2, w=1, r=1, d=1}
```

---

### Print index positions of all vowels using IntStream

```java
String s = "hello world";
Set<Character> vowels = Set.of('a','e','i','o','u');
IntStream.range(0, s.length())
    .filter(i -> vowels.contains(s.charAt(i)))
    .forEach(i -> System.out.println("Vowel '" + s.charAt(i) + "' at index " + i));
// Vowel 'e' at index 1
// Vowel 'o' at index 4
// Vowel 'o' at index 7
```

---

## SQL Coding

---

### Find the Nth highest salary per department using DENSE_RANK()

```sql
WITH ranked AS (
    SELECT
        name, department, salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dr
    FROM employees
)
SELECT name, department, salary FROM ranked WHERE dr = 2; -- replace with N
```

---

### Find the 2nd highest salary without LIMIT or window functions

```sql
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

---

### Find employees earning more than their manager

```sql
SELECT e.name, e.salary, m.name as manager, m.salary as manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

---

### Calculate a running total of sales by date

```sql
SELECT
    sale_date,
    amount,
    SUM(amount) OVER (ORDER BY sale_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM sales
ORDER BY sale_date;
```

---

### Find duplicate records and their count

```sql
SELECT email, COUNT(*) AS count
FROM users
GROUP BY email
HAVING COUNT(*) > 1
ORDER BY count DESC;
```

---

### Fetch paginated records — page 3, 20 records per page (offset and keyset)

```sql
-- Offset-based
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 40;

-- Keyset pagination (fast for large pages)
-- After page 2, cursor is last seen id = 40
SELECT * FROM orders WHERE id > 40 ORDER BY id LIMIT 20;
```

---

### Find average salary per department where avg > 50000

```sql
SELECT department, ROUND(AVG(salary), 2) as avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000
ORDER BY avg_salary DESC;
```
# Section 18: Scenario-Based & Production Debugging

---

## Application-Level Scenarios

---

### Your Spring Boot app works locally but fails after deployment. What are the first five things you check?

**Core Explanation:**

Deployment failures fall into five root-cause categories: configuration drift, environment mismatch, resource constraints, network/dependency differences, and packaging errors.

**Systematic Checklist:**

**1. Startup logs and exception stack trace**
The first and fastest signal. Look for `ApplicationContext` failure, `BeanCreationException`, or `ConnectException`. Read the full stack trace — the root cause is almost always buried in "Caused by".

```bash
# In a container environment
kubectl logs <pod-name> --previous   # logs from crashed container
kubectl describe pod <pod-name>      # OOMKilled, CrashLoopBackOff reasons

# On a VM / EC2
journalctl -u myapp.service -n 200
tail -f /var/log/myapp/application.log
```

**2. External configuration not applied / wrong profile**

`application.properties` may be overridden by environment variables, config servers, or the active profile may be wrong.

```bash
# Check which profile is active
echo $SPRING_PROFILES_ACTIVE

# Verify Kubernetes env vars
kubectl exec <pod> -- env | grep SPRING

# Actuator endpoint — requires management.endpoints.web.exposure.include=env
curl http://host/actuator/env | jq '.propertySources'
```

**3. External dependencies unreachable (DB, Redis, Kafka, third-party APIs)**

Deployed environments often have stricter network policies. Check DNS resolution, port reachability, and credentials.

```bash
# From inside the container
kubectl exec -it <pod> -- /bin/sh
nslookup postgres-service
curl -v telnet://postgres-service:5432

# Check connection pool at startup
# HikariCP logs: "HikariPool-1 - Connection is not available, request timed out"
```

**4. Environment-specific secrets and credentials**

Passwords, API keys, JWT secrets, or TLS certificates may differ between environments. Common mistakes: using local `.env` values hardcoded in code, wrong Vault path, missing K8s Secret binding.

```yaml
# Verify K8s secret is mounted correctly
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

**5. JVM memory, CPU limits, and OS-level constraints**

Containers default to low memory limits. JVM without `-XX:+UseContainerSupport` (Java 10+) reads host memory, not container limits, causing OOMKilled.

```bash
# Symptoms: pod gets OOMKilled
kubectl describe pod <pod> | grep -i oom

# Fix in Dockerfile / K8s
JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

# Also check: open file descriptors, ulimit, disk space
df -h
ulimit -a
```

**Edge Cases/Pitfalls:**
- Classpath resources that exist locally but weren't packaged (check `src/main/resources` inclusion in Maven/Gradle)
- SSL certificate issues — local JVM trusts self-signed cert, production does not
- Time zone differences (`TZ` env var) causing timestamp parsing failures
- Spring Cloud Config Server not reachable on deploy → app fails to start

**Best Practices:**
- Always use structured logging with correlation IDs so production logs are parseable
- Implement health checks (`/actuator/health`) that verify database and downstream connectivity
- Use `spring.config.import` and externalize ALL environment-specific config
- Test the production Docker image locally with production-like env vars before deploying

---

### APIs are fast locally but slow only in production. How do you debug this systematically?

**Core Explanation:**

Performance differences between local and production almost always come from: data volume, network latency, resource contention, missing indexes, or environment-specific configuration. Debug systematically — measure before changing anything.

**Step-by-Step Debugging Approach:**

**Step 1 — Establish baseline measurements**

```bash
# HTTP response time distribution
curl -w "@curl-format.txt" -s -o /dev/null https://prod/api/endpoint

# curl-format.txt
time_namelookup:  %{time_namelookup}\n
time_connect:     %{time_connect}\n
time_appconnect:  %{time_appconnect}\n
time_pretransfer: %{time_pretransfer}\n
time_starttransfer: %{time_starttransfer}\n
time_total:       %{time_total}\n
```

**Step 2 — Add distributed tracing to find the slow span**

```java
// With Micrometer + Zipkin/Jaeger, every span shows its own duration
// Look for: which service/layer is slow

// If not yet instrumented, add timing logs:
@Around("@annotation(org.springframework.web.bind.annotation.GetMapping)")
public Object timeEndpoint(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        return pjp.proceed();
    } finally {
        log.info("Method {} took {}ms", pjp.getSignature(), System.currentTimeMillis() - start);
    }
}
```

**Step 3 — Check database query times (most common culprit)**

```sql
-- PostgreSQL: slow query log
SET log_min_duration_statement = 100; -- log queries > 100ms

-- Check pg_stat_statements for worst offenders
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- Check for missing indexes (sequential scans on large tables)
EXPLAIN (ANALYZE, BUFFERS) SELECT ...
```

```java
// Enable Hibernate SQL logging + execution time
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
# Or use p6spy for timing:
spring.datasource.url=jdbc:p6spy:postgresql://...
```

**Step 4 — Rule out N+1 and connection pool exhaustion**

```
# N+1 symptom: 1 slow API call generates 100+ DB queries
# Fix: JOIN FETCH / @EntityGraph (see section 9)

# Connection pool exhaustion: requests wait in pool queue
# HikariCP metric: hikaricp.connections.pending > 0
# Fix: increase pool size or find slow transactions holding connections
```

**Step 5 — Check garbage collection pauses**

```bash
# JVM GC logs
-Xlog:gc*:file=/var/log/gc.log:time,level,tags

# Look for: long GC pause times > 200ms
# Common cause: high allocation rate, large heap with old GC
# Fix: switch to G1 or ZGC, tune heap size
```

**Step 6 — Check thread pool saturation**

```java
// Actuator thread dump
GET /actuator/threaddump

// Look for: threads in WAITING state stuck on:
// - Lock.lock() → contention
// - Socket.read() → slow downstream service
// - Connection.getConnection() → pool exhaustion
```

**Edge Cases/Pitfalls:**
- Local uses H2 in-memory; production uses PostgreSQL with network latency
- Local dataset is 100 rows; production has 10 million rows — index matters only at scale
- Docker networking adds ~1ms per hop; multiple hops in service mesh add up
- CPU throttling in containers (Kubernetes CPU limits) causes JIT compilation to be slow

**Best Practices:**
- Always use production-like data volumes in staging
- Implement SLO-based alerting: alert when p99 latency crosses threshold
- Use `spring.jpa.properties.hibernate.generate_statistics=true` in staging to detect N+1 early
- Profile with async-profiler attached to the running JVM (zero restart, low overhead)

---

### You updated `application.properties` but changes are not reflecting. What are the possible causes?

**Core Explanation:**

Spring Boot property loading has a well-defined precedence order. A value set in one source silently overrides another, and packaged `.properties` files inside the JAR may be stale.

**Possible Causes (ordered by frequency):**

**1. Environment variables or system properties override the file**

Spring Boot property precedence (highest to lowest):
```
1. Command-line arguments (--server.port=9090)
2. SPRING_APPLICATION_JSON env var
3. OS environment variables (SERVER_PORT=9090)
4. JVM system properties (-Dserver.port=9090)
5. application.properties in /config/ external dir
6. application.properties in current dir
7. application.properties in classpath /config/
8. application.properties in classpath root
```

```bash
# Check what's actually active
curl http://localhost:8080/actuator/env | jq '.propertySources[] | select(.name | contains("systemEnvironment")) | .properties'
```

**2. Wrong active profile — editing the wrong file**

```properties
# You edited application.properties but app runs with --spring.profiles.active=prod
# and application-prod.properties has a different value
# Profile-specific properties override the base file
```

**3. Application not rebuilt / old JAR still running**

```bash
# The packaged JAR contains the old application.properties
# You edited the source file but didn't rebuild and redeploy
mvn clean package -DskipTests
# Then verify contents of the JAR:
jar tf target/myapp.jar | grep application.properties
jar xf target/myapp.jar BOOT-INF/classes/application.properties
cat application.properties
```

**4. External Config Server (Spring Cloud Config) overrides local file**

```properties
# bootstrap.properties / bootstrap.yml takes precedence
spring.cloud.config.uri=http://config-server:8888
# Config server returns a value that overrides local file
```

**5. `@Value` field with hardcoded default**

```java
// Hardcoded default overrides missing property, but if the property
// was always there and now you change it, check:
@Value("${cache.ttl:300}")   // ← always falls back to 300 even if property is wrong
private int cacheTtl;

// Verify via /actuator/env — the property key must match exactly (case-sensitive)
```

**6. `@RefreshScope` not used / `/actuator/refresh` not called**

For runtime property updates without restart, you need `@RefreshScope` and to POST to `/actuator/refresh`. Without this, beans cache the value at startup.

```java
@RefreshScope
@Component
public class FeatureFlags {
    @Value("${feature.dark-mode:false}")
    private boolean darkMode;
}

// After changing property in Config Server:
curl -X POST http://localhost:8080/actuator/refresh
```

**Edge Cases/Pitfalls:**
- YAML key structure: `spring.datasource.url` vs `spring.datasource: url:` indentation error → property silently not set
- Property names with dots in YAML need quoting: `"[my.key]": value`
- `@ConfigurationProperties` classes need `@EnableConfigurationProperties` or `@Component`

**Best Practices:**
- Always verify with `/actuator/env` after changing properties
- Use the Config Server with Git backend for auditability
- Keep environment-specific values in environment variables, not in committed files

---

### CPU usage is low but requests are timing out. What could be blocking threads?

**Core Explanation:**

Low CPU + high latency means threads are waiting, not working. Threads wait on I/O (database, network, file system), locks, or are exhausted in the thread pool queue.

**Diagnostic Approach:**

**Step 1 — Take a thread dump**

```bash
# JVM thread dump (no restart needed)
kill -3 <pid>           # prints to stdout/log
jstack <pid>            # prints to console
# Or via Actuator:
curl http://localhost:8080/actuator/threaddump
```

**Step 2 — Interpret the thread dump**

```
# Look for threads WAITING or TIMED_WAITING on:
"http-nio-8080-exec-10" #45 daemon prio=5 os_prio=0 tid=0x... nid=0x... waiting for monitor entry
    java.lang.Thread.State: BLOCKED (on object monitor)
    at com.example.service.OrderService.processOrder(OrderService.java:87)
    - waiting to lock <0x00000000d8e14e00> (a com.example.service.OrderService)
```

**Common Causes:**

**1. Database connection pool exhausted**
```
# Thread dump shows all threads waiting at:
HikariPool.getConnection() → all HTTP threads blocked waiting for a DB connection
# Because: some queries are slow / transactions not closing / connection leak
```

**2. Synchronized block / method contention**
```java
// Bad: coarse-grained synchronized on shared object
@Service
public class CounterService {
    private int count = 0;

    public synchronized void increment() { // all threads serialize here
        count++;
        doSlowWork(); // holding lock while doing slow work!
    }
}

// Fix: use AtomicInteger, reduce lock scope, or use ReentrantLock with tryLock
```

**3. Downstream HTTP call without timeout**
```java
// RestTemplate without timeout → thread blocks indefinitely waiting for response
RestTemplate rt = new RestTemplate(); // no timeout!

// Fix:
SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
factory.setConnectTimeout(2000);
factory.setReadTimeout(5000);
RestTemplate rt = new RestTemplate(factory);

// Or with WebClient (non-blocking):
webClient.get()
    .uri("/api")
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(5));
```

**4. Thread pool too small**
```properties
# Tomcat default: 200 threads
# If all 200 are blocked on slow DB calls, new requests queue and time out
server.tomcat.threads.max=200
server.tomcat.accept-count=100  # queue size before rejecting

# Fix: don't block threads on I/O — use async processing
# Or: increase pool size (temporary fix, addresses symptom not cause)
```

**5. Deadlock between two locks**
```
# Thread dump shows circular wait:
Thread A: waiting to acquire lock B (holds lock A)
Thread B: waiting to acquire lock A (holds lock B)

# Java deadlock detection:
jstack <pid> | grep -A 20 "deadlock"
# JVM also prints "Found one Java-level deadlock:" in thread dump
```

**6. External service not responding (no circuit breaker)**
```java
// Every request spawns a call to slow-service.internal
// Threads pile up waiting for TCP timeout (default 2 minutes!)
// Fix: always set connection/read timeouts + circuit breaker (Resilience4j)
```

**Edge Cases/Pitfalls:**
- `@Transactional` methods calling external HTTP APIs while holding a DB transaction → connection held for entire duration of HTTP call
- Thread-local MDC or SecurityContext not cleared → memory leak + stale context in pooled threads
- Virtual threads (Java 21) still block on `synchronized` — must use ReentrantLock

**Best Practices:**
- Always set connect + read timeouts on all HTTP clients
- Never hold a database connection while making external network calls
- Implement circuit breakers for all downstream dependencies
- Monitor HikariCP metrics: `hikaricp.connections.pending`, `hikaricp.connections.active`

---

### Application memory usage keeps increasing over time. How do you investigate?

**Core Explanation:**

Increasing memory over time is a memory leak. Java objects are accumulating faster than GC can collect them — usually because something is holding references that prevent collection.

**Investigation Steps:**

**Step 1 — Confirm it's a real leak (not just GC not running)**
```bash
# Check GC behavior — memory should drop after Full GC
jstat -gcutil <pid> 5000   # print GC stats every 5 seconds
# S0  S1   E   O   M   CCS  YGC   YGCT  FGC  FGCT  GCT
# If Old Gen (O) keeps growing and doesn't drop after FGC → memory leak
```

**Step 2 — Heap dump analysis**
```bash
# Capture heap dump (app continues running)
jcmd <pid> GC.heap_dump /tmp/heapdump.hprof
# Or trigger on OOM:
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/

# Analyze with Eclipse MAT or VisualVM
# MAT: "Leak Suspects" report → shows objects with most retained heap
# Look for: Dominator tree — what's holding the most memory?
```

**Step 3 — Common leak patterns to look for in heap dump:**

**Static collections accumulating data:**
```java
// Classic leak: static Map never cleaned
public class SessionCache {
    private static final Map<String, UserData> cache = new HashMap<>();

    public static void put(String key, UserData data) {
        cache.put(key, data);  // never removed → grows forever
    }
}
// Fix: use WeakHashMap, Caffeine cache with TTL/size limits, or explicit eviction
```

**ThreadLocal not removed:**
```java
// Tomcat reuses threads; if ThreadLocal not cleared, references accumulate
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<>();

// Fix: always call remove() in finally block
try {
    connectionHolder.set(conn);
    doWork();
} finally {
    connectionHolder.remove();  // CRITICAL
}
```

**Event listeners and callbacks not deregistered:**
```java
// Spring ApplicationEvent listeners in prototype-scoped beans
// Guava EventBus subscriptions never unregistered
// Fix: implement DisposableBean / @PreDestroy and unregister
@PreDestroy
public void cleanup() {
    eventBus.unregister(this);
}
```

**Cache without size/TTL bounds:**
```java
// Unbounded Caffeine / Guava cache with no eviction policy
Cache<String, byte[]> cache = Caffeine.newBuilder()
    // MISSING: .maximumSize(1000).expireAfterWrite(1, TimeUnit.HOURS)
    .build();
```

**Step 4 — Use JFR for continuous monitoring**
```bash
# Java Flight Recorder (zero overhead profiling)
jcmd <pid> JFR.start duration=60s filename=/tmp/recording.jfr
jfr print --events jdk.ObjectAllocationInNewTLAB /tmp/recording.jfr
# Open in JDK Mission Control for visual analysis
```

**Edge Cases/Pitfalls:**
- Metaspace leak: class loaders not GC'd (common in hot-reload / OSGi / classloader-per-tenant architectures)
- Direct memory leak: `ByteBuffer.allocateDirect()` not freed — not visible in heap dump
- Off-heap memory: JVM native libraries (Netty, Caffeine) using off-heap allocation

**Best Practices:**
- Set JVM flags `-XX:+HeapDumpOnOutOfMemoryError` in all production environments
- Monitor `jvm.memory.used` and `jvm.gc.pause` via Micrometer/Prometheus
- Use Caffeine for all application-level caches with explicit `maximumSize` and `expireAfterWrite`
- Review all `static` collections in code review — ask "what removes from this?"

---

### `@Transactional` is present but rollback doesn't happen. List all possible causes.

**Core Explanation:**

Spring `@Transactional` rollback has many silent failure modes. Each requires a different fix.

**Complete List of Causes:**

**1. Exception type is not a RuntimeException (checked exception)**
```java
// Default: only rolls back on RuntimeException and Error
// Checked exceptions (IOException, SQLException) do NOT trigger rollback by default
@Transactional
public void process() throws IOException {
    repo.save(entity);
    throw new IOException("oops");  // NO rollback!
}

// Fix:
@Transactional(rollbackFor = IOException.class)
// Or: @Transactional(rollbackFor = Exception.class)
```

**2. Self-invocation (calling @Transactional method from same class)**
```java
@Service
public class OrderService {

    public void placeOrder() {
        this.processPayment();  // NOT proxied! Calls real method, no transaction
    }

    @Transactional
    public void processPayment() {
        // transaction never started
    }
}
// Fix: inject self, use AspectJ mode, or restructure into two classes
```

**3. Exception caught and not re-thrown**
```java
@Transactional
public void save(Entity e) {
    try {
        repo.save(e);
        externalService.call();
    } catch (Exception ex) {
        log.error("Error", ex);  // exception swallowed! Rollback never triggered
    }
}
// Fix: re-throw or use TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()
```

**4. Method is not public**
```java
// Spring AOP proxies only intercept public methods
@Transactional
protected void internalUpdate() { ... }  // annotation ignored!
// Fix: make it public
```

**5. Bean is not managed by Spring (instantiated with `new`)**
```java
OrderService service = new OrderService();  // not a Spring proxy
service.processOrder();  // @Transactional has no effect
// Fix: always inject via @Autowired / constructor injection
```

**6. Wrong transaction propagation**
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void audit() {
    // Runs in its own transaction — rollback here doesn't affect outer transaction
    // and vice versa
}
```

**7. Database/storage doesn't support transactions**
```java
// MyISAM tables in MySQL don't support transactions
// MongoDB (pre 4.0) — single-document ops are atomic, multi-document requires explicit transaction
// Fix: use InnoDB, or switch to transactional storage engine
```

**8. `@Transactional` on wrong layer (not on the Spring bean)**
```java
// Annotation on an interface method works only with JDK proxy (if bean implements interface)
// With CGLIB proxy (no interface), annotate the concrete class
```

**9. TransactionManager bean misconfigured**
```java
// Multiple data sources → multiple transaction managers
// @Transactional without specifying which one uses the primary
@Transactional("secondaryTransactionManager")  // must be explicit
```

**Edge Cases/Pitfalls:**
- `@Transactional` on `@Async` method: async methods run in a new thread where the calling transaction is not visible. Each needs its own transaction.
- Exception thrown in `@PreDestroy` or shutdown hook after transaction commits → nothing to roll back

**Best Practices:**
- Keep transactions short — never make external HTTP calls inside `@Transactional`
- Use `@Transactional(rollbackFor = Exception.class)` when in doubt
- Test rollback behavior with integration tests, not unit tests (unit tests mock the repo)

---

### Database connection pool gets exhausted under load. How do you identify and fix it?

**Core Explanation:**

HikariCP connection pool exhaustion means more threads need DB connections than the pool can provide. New requests wait up to `connectionTimeout` (default 30 seconds) then throw `SQLTransientConnectionException`.

**Identification:**

```bash
# Symptoms in logs:
HikariPool-1 - Connection is not available, request timed out after 30000ms

# Metrics to monitor:
hikaricp.connections.active       # connections currently in use
hikaricp.connections.pending      # threads waiting for a connection
hikaricp.connections.idle         # connections available
hikaricp.connections.timeout.total # number of timeouts (alert on this!)
```

**Step 1 — Take a thread dump to find who's holding connections**

All HTTP threads should show their current stack. Look for:
- Threads in `@Transactional` methods doing slow work
- Threads making external HTTP calls inside a transaction
- Threads holding connections while waiting on locks

**Step 2 — Enable HikariCP leak detection**

```properties
spring.datasource.hikari.leak-detection-threshold=2000  # warn if connection held > 2 seconds
# Prints stack trace of where connection was acquired — invaluable for finding leaks
```

**Step 3 — Find the root causes:**

**Cause A: Pool size too small for load**
```properties
# Default: 10 connections
spring.datasource.hikari.maximum-pool-size=10

# Formula: pool_size = (core_count * 2) + effective_spindle_count
# But first, find if slow queries are the real problem
```

**Cause B: Long-running transactions**
```java
// @Transactional method that:
// - Makes HTTP calls to external services
// - Does file I/O
// - Runs complex in-memory processing
// All while holding a DB connection!

// Fix: move non-DB work outside the @Transactional boundary
public void processOrder(OrderRequest req) {
    byte[] pdf = pdfService.generate(req);   // ← do this FIRST, outside transaction

    orderService.saveOrder(req, pdf);         // ← @Transactional here, short and fast
}
```

**Cause C: Connection not returned (resource leak)**
```java
// Using DataSource directly without try-with-resources
Connection conn = dataSource.getConnection();
doWork(conn);
// If doWork() throws, connection never returned → pool gradually drains
conn.close();  // ← never reached!

// Fix:
try (Connection conn = dataSource.getConnection()) {
    doWork(conn);
}  // always closed
```

**Cause D: Misconfigured connection pool size < actual concurrency**
```properties
# If you have 200 Tomcat threads all needing DB at the same time,
# 10 pool connections = 190 threads always waiting
spring.datasource.hikari.maximum-pool-size=50
server.tomcat.threads.max=100
# Rule: pool size should match realistic DB concurrency, not HTTP concurrency
```

**Edge Cases/Pitfalls:**
- Connection pool fine at p50 latency, exhausts at p99 (slow query causes cascading waits)
- Multiple `DataSource` beans with separate pools — if one is misconfigured it fails silently

**Best Practices:**
- Never make external HTTP/RPC calls inside `@Transactional`
- Set `leak-detection-threshold` in all environments
- Monitor `hikaricp.connections.pending` — alert when > 0 sustained
- Use separate connection pools for read replicas vs primary

---

### Scheduled background jobs start affecting API latency. How do you isolate them?

**Core Explanation:**

By default, Spring `@Scheduled` uses a single-threaded `ThreadPoolTaskScheduler`. If background jobs consume CPU, I/O, or DB connections, they compete with API request threads.

**Isolation Strategies:**

**1. Run scheduled tasks on a dedicated thread pool**

```java
@Configuration
public class SchedulerConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(4);
        scheduler.setThreadNamePrefix("scheduler-");
        scheduler.setThreadPriority(Thread.MIN_PRIORITY); // lower priority than API threads
        return scheduler;
    }
}
```

**2. Use a separate DataSource / connection pool for batch jobs**

```java
@Configuration
public class BatchDataSourceConfig {

    @Bean("batchDataSource")
    @ConfigurationProperties("spring.datasource.batch")
    public DataSource batchDataSource() {
        return DataSourceBuilder.create().build();
    }
}

// spring.datasource.batch.hikari.maximum-pool-size=5
// This prevents batch jobs from consuming API connections
```

**3. Offload batch processing to @Async with bounded executor**

```java
@Component
public class ReportJob {

    @Scheduled(cron = "0 0 2 * * ?")  // 2 AM daily
    public void generateReports() {
        reportService.generateAsync();  // fire and forget
    }
}

@Service
public class ReportService {

    @Async("batchExecutor")
    public void generateAsync() {
        // runs on dedicated batch thread pool
    }
}

// Config:
@Bean("batchExecutor")
public Executor batchExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(2);
    exec.setMaxPoolSize(4);
    exec.setQueueCapacity(50);
    return exec;
}
```

**4. Use CPU and I/O priority controls (Linux)**

```bash
# Lower I/O priority of Java process
ionice -c 3 -p <pid>   # idle class: only uses I/O when no other process needs it

# Nice level for CPU
renice +10 -p <pid>
```

**5. Move to a separate service / worker process**

For heavy batch jobs, deploy a separate Spring Batch application that shares the same database but runs independently.

**Edge Cases/Pitfalls:**
- `@Scheduled` with `fixedRate` on a slow job: tasks pile up in queue if execution exceeds interval
- Solution: use `fixedDelay` (waits for previous to finish) or distributed locking (Shedlock)

```java
// Prevent overlapping execution with ShedLock
@Scheduled(cron = "*/10 * * * * *")
@SchedulerLock(name = "myJob", lockAtMostFor = "PT9S", lockAtLeastFor = "PT5S")
public void myJob() { ... }
```

**Best Practices:**
- Always configure a named, bounded thread pool for `@Scheduled` tasks
- Use `@ConditionalOnProperty` to disable scheduled jobs in test environments
- Implement job monitoring: log start/end time, record duration as a metric

---

### Application behaves differently in Docker vs locally. What are the common causes?

**Core Explanation:**

Docker introduces OS-level, resource, and networking differences that don't exist when running locally.

**Common Causes:**

**1. JVM doesn't respect container memory limits (Java < 10)**
```bash
# Old JVM reads host memory (e.g., 64GB) and sizes heap accordingly
# Container limit is 512MB → OOMKilled
# Fix: Java 10+ has -XX:+UseContainerSupport by default
# Also set:
JAVA_OPTS="-XX:MaxRAMPercentage=75.0"
```

**2. File path separator and case sensitivity**
```
# macOS filesystem: case-insensitive (File("README.md") finds "readme.md")
# Linux filesystem (Docker): case-sensitive
# Fix: use exact case in all file references
```

**3. Timezone differences**
```bash
# Docker container default TZ is UTC
# Local machine may be America/New_York
# Fix: set TZ explicitly
ENV TZ=UTC
# Or in Java:
-Duser.timezone=UTC
```

**4. Network resolution (localhost vs service name)**
```
# Local: database is at localhost:5432
# Docker: database is at db:5432 (service name in docker-compose)
# spring.datasource.url=jdbc:postgresql://db:5432/mydb
```

**5. Port already in use / binding to wrong interface**
```
# Local: app binds to 0.0.0.0:8080
# Docker: if EXPOSE not set or port not mapped, app unreachable
# Fix: always -p 8080:8080 and EXPOSE 8080 in Dockerfile
```

**6. Environment variable name mapping**
```
# Spring Boot maps SPRING_DATASOURCE_URL → spring.datasource.url
# But: underscore vs dot matters
# Fix: use exact Spring relaxed binding names
```

**7. CPU architecture (Apple Silicon M1/M2 → ARM vs x86)**
```bash
# Build on ARM Mac, deploy to x86 Linux
# Fix:
docker buildx build --platform linux/amd64 -t myapp:latest .
```

**8. Non-root user can't bind to ports < 1024**
```dockerfile
# You run app as non-root (best practice), port 80 requires root
# Fix: use port 8080 in container, map to 80 externally
USER appuser
EXPOSE 8080  # not 80
```

**Best Practices:**
- Use `docker-compose.yml` for local development that mirrors production topology
- Keep Docker image lean and deterministic (pin base image versions)
- Always test with `docker run` locally before deploying

---

### Logs are missing in production but fine locally. Where do you look?

**Core Explanation:**

Log configuration, log routing, and container log drivers differ between environments.

**Checklist:**

**1. Log level is higher in production**
```properties
# Local: application.properties
logging.level.com.example=DEBUG

# Production: application-prod.properties or env var
LOGGING_LEVEL_COM_EXAMPLE=WARN  # overwrites local → DEBUG logs hidden

# Fix: check /actuator/loggers
curl http://host/actuator/loggers/com.example
```

**2. Asynchronous appender dropped logs**
```xml
<!-- Logback AsyncAppender drops events when queue is full -->
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <discardingThreshold>20</discardingThreshold>  <!-- drops INFO+ when queue 80% full -->
    <queueSize>512</queueSize>
    <appender-ref ref="FILE"/>
</appender>

<!-- Fix: increase queueSize or set discardingThreshold=0 -->
<discardingThreshold>0</discardingThreshold>
```

**3. Container stdout not captured / wrong log driver**
```bash
# Docker logs by default capture stdout/stderr
# But if app writes to a file, docker logs shows nothing
# Fix: log to stdout (Spring Boot default)
logging.file.name=  # empty = stdout only

# Check Docker log driver:
docker inspect <container> | jq '.[].HostConfig.LogConfig'
```

**4. Log file rotation deleted old logs**
```xml
<!-- Logback: max history in days -->
<maxHistory>7</maxHistory>
<!-- Logs older than 7 days deleted — you're looking at last week's incident -->
```

**5. Multiple instances — looking at wrong pod**
```bash
# K8s: 10 replicas, you're tailing 1 pod
kubectl logs -l app=myapp --all-pods  # aggregate all pods
# Or use centralized logging (ELK, Loki) to search across all instances
```

**6. Structured logging vs plain text — log parser drops malformed lines**
```json
// ELK/Loki filter that expects JSON drops plain text lines
// Fix: use consistent structured logging (logstash-logback-encoder)
// Add to pom.xml:
// net.logstash.logback:logstash-logback-encoder
```

**Best Practices:**
- Always log to stdout in containerized apps — let the platform capture and route
- Use centralized log aggregation (ELK, Grafana Loki, Datadog)
- Never rely on file-based logs in ephemeral containers

---

### A REST API returns correct data but response time is inconsistent. What are the possible causes?

**Core Explanation:**

Inconsistent latency (high variance between requests) points to resource contention, GC pauses, cache hit/miss variance, or variable-cost operations.

**Possible Causes:**

**1. JVM GC stop-the-world pauses**
```bash
# G1GC or CMS can pause all threads for 100-500ms unpredictably
# Check: -Xlog:gc* logs, or JFR recording
# Fix: switch to ZGC/Shenandoah (sub-millisecond pauses)
-XX:+UseZGC  # Java 15+
```

**2. Cache hit/miss variance**
```
# Cache hit: 2ms
# Cache miss: 200ms (goes to DB)
# If cache has 90% hit rate, 10% of requests are 100x slower
# Fix: pre-warm cache, use probabilistic early expiration to avoid stampede
```

**3. HikariCP connection wait time**
```
# Sometimes all connections busy → request waits in queue
# Symptom: bimodal distribution — fast requests and slow requests
# Fix: see connection pool exhaustion section
```

**4. External API dependency with variable latency**
```
# If your API calls another service, your p99 = their p99
# Fix: circuit breaker + timeout + fallback
```

**5. Thread scheduling jitter (OS-level)**
```
# Java thread scheduler on a loaded VM/container
# Threads sometimes wait longer to be scheduled
# Fix: ensure sufficient CPU resources, don't over-subscribe pods
```

**6. Input-dependent processing time**
```java
// Large request bodies take longer to process/deserialize
// Complex queries based on request parameters
// Fix: cap input size, add parameter-based caching
```

**7. DNS resolution (not cached / TTL expired)**
```java
// If your service resolves external hostnames on every request
// DNS lookup can add 10-100ms intermittently
// Fix: use connection pooling (connections already established, no DNS per request)
// Or: configure DNS caching: -Dsun.net.inetaddr.ttl=300
```

**Best Practices:**
- Track latency as a histogram (p50/p95/p99), not just average
- Use distributed tracing to find which span has variable duration
- Implement request logging with duration for every API call

---

### Service restarts automatically without clear errors. What could trigger this?

**Core Explanation:**

Automatic restarts in production come from: OOMKilled (K8s/Docker), failed health checks, uncaught exceptions in Spring lifecycle, or watchdog/supervisor processes.

**Causes and Diagnosis:**

**1. OOMKilled — JVM heap or native memory exhaustion**
```bash
kubectl describe pod <pod> | grep -A5 "Last State"
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137  ← signal 9 (SIGKILL from kernel)

# Fix: increase memory limit, add -XX:+HeapDumpOnOutOfMemoryError, investigate leak
```

**2. Kubernetes liveness probe failing**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 3
  periodSeconds: 10
# If /actuator/health returns 503 three times → K8s kills and restarts pod

# Diagnosis: check probe endpoint manually
curl http://pod-ip:8080/actuator/health/liveness
# Common cause: thread pool exhaustion causes health check to also time out
```

**3. Spring Boot DevTools in production**
```properties
# DevTools has file watcher that restarts the app on classpath changes
# Never include spring-boot-devtools in production builds
# pom.xml: exclude from production profile
```

**4. Supervisor / systemd restart policy**
```bash
# If using systemd:
systemctl status myapp.service
journalctl -u myapp.service -n 50

# If process exits with non-zero code, systemd restarts it
# Exit code 1 = JVM startup failure, exit code 137 = SIGKILL
```

**5. Uncaught exception in non-daemon thread**
```java
// If a non-daemon thread throws an uncaught exception, JVM exits
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    log.error("Uncaught exception in thread {}", thread.getName(), throwable);
    // Don't exit here — log and recover if possible
});
```

**Best Practices:**
- Separate liveness probe from readiness probe — liveness should only fail for truly unrecoverable states
- Set `restartPolicy: OnFailure` in K8s instead of `Always` to avoid infinite restart loops
- Alert on `kube_pod_container_status_restarts_total` metric

---

### After enabling parallel streams, response time increased. Why?

**Core Explanation:**

Parallel streams use the common `ForkJoinPool` (shared across the entire JVM). In a web server, this pool competes with every other parallel stream operation, causing contention.

**Root Cause Explanation:**

```java
// ForkJoinPool.commonPool() has parallelism = Runtime.availableProcessors() - 1
// e.g., 4-core machine → 3 threads in common pool

// Every parallel stream in the JVM shares these 3 threads:
// - Your API endpoint
// - Other API endpoints serving concurrent requests
// - Background jobs using parallel streams
// = All threads fighting for 3 ForkJoinPool threads

list.parallelStream()
    .filter(...)
    .collect(Collectors.toList());
// This looks fast in isolation but is slow under load
```

**Why parallelism can hurt in web servers:**

```
Sequential stream: uses 1 Tomcat thread, no ForkJoinPool contention
Parallel stream: spawns tasks to ForkJoinPool, but if pool is busy → waits
                 Thread context switch overhead + synchronization cost
                 For small datasets: overhead > savings
```

**Correct use of parallel streams:**

```java
// WRONG: parallel stream in web request handler
@GetMapping("/report")
public List<ReportLine> generateReport() {
    return hugeList.parallelStream()  // ← contends with all other requests
        .map(this::processItem)
        .collect(toList());
}

// BETTER: use custom ForkJoinPool
@GetMapping("/report")
public List<ReportLine> generateReport() throws Exception {
    ForkJoinPool customPool = new ForkJoinPool(4);
    try {
        return customPool.submit(() ->
            hugeList.parallelStream()
                .map(this::processItem)
                .collect(toList())
        ).get();
    } finally {
        customPool.shutdown();
    }
}
// But creating a ForkJoinPool per request is also expensive

// BEST: for truly parallel CPU work, pre-create a bounded executor
// Or: process asynchronously and return immediately with a task ID
```

**When parallel streams help vs hurt:**
- Help: CPU-bound operations, large datasets (> 10,000 elements), batch jobs
- Hurt: Small datasets, I/O-bound operations, web request handlers, operations with shared state

**Best Practices:**
- Never use `parallelStream()` blindly — benchmark first with realistic concurrent load
- Use `parallelStream()` only in batch/offline processing, not in API handlers
- Monitor ForkJoinPool: `ForkJoinPool.commonPool().getActiveThreadCount()`

---

### Excessive logging was added for debugging and production performance degraded severely. Why?

**Core Explanation:**

Logging has multiple performance costs that compound under high throughput: string concatenation, I/O, synchronization in appenders, and log level evaluation overhead.

**Root Causes:**

**1. String concatenation even when log level is disabled**
```java
// BAD: string is built even if DEBUG is disabled
logger.debug("Processing order: " + order.toString() + " for user: " + user.getId());
// order.toString() called regardless of log level

// GOOD: lazy evaluation with parameterized logging
logger.debug("Processing order: {} for user: {}", order, user.getId());
// toString() only called if DEBUG is enabled

// ALSO GOOD: guard check for expensive operations
if (logger.isDebugEnabled()) {
    logger.debug("State: {}", expensiveToCompute());
}
```

**2. Synchronous file I/O in the hot path**
```xml
<!-- Synchronous file appender: every log call blocks on disk I/O -->
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <immediateFlush>true</immediateFlush>  <!-- every write flushed to disk immediately -->
</appender>

<!-- Fix: AsyncAppender wraps sync appender -->
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>8192</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <neverBlock>false</neverBlock>
    <appender-ref ref="FILE"/>
</appender>
```

**3. Log rotation under load**
```
# When log file rolls over at midnight or size limit:
# - Rename file
# - Create new file
# - Brief spike in latency
# Fix: async appender absorbs the spike
```

**4. JSON serialization of complex objects**
```java
// Logging entire HTTP request/response bodies in JSON format
logger.info("Request: {}", objectMapper.writeValueAsString(request));
// ObjectMapper serialization is expensive — especially for large nested objects
// Fix: log only relevant fields, use lazy supplier
logger.info("Request: {}", (Supplier<String>) () -> objectMapper.writeValueAsString(request));
```

**5. Log output going to stdout in container (which goes to Docker log driver)**
```
# Docker JSON file log driver buffers and writes logs to disk
# Under extreme logging, disk I/O becomes the bottleneck
# Fix: use a log aggregator (Fluentd, Logstash) with async pipeline
```

**Best Practices:**
- Always use parameterized logging: `logger.debug("x={}", x)` never `"x=" + x`
- Always use `AsyncAppender` in production
- Remove excessive debug logging after troubleshooting — use log levels properly
- Log only structured, relevant fields; avoid serializing full objects

---

### Lazy loading works locally but fails in production with `LazyInitializationException`. What is missing?

**Core Explanation:**

`LazyInitializationException` means a lazily-loaded association was accessed after the Hibernate Session was closed. It works locally but not in production usually because of a configuration difference or architectural difference.

**Common Causes and Fixes:**

**1. `Open Session in View` is enabled locally but disabled in production**
```properties
# OpenSessionInView keeps the Hibernate session open for the entire HTTP request
# Spring Boot default: TRUE (bad practice, but common)
spring.jpa.open-in-view=true   # ← local default that masks the problem

# Production config (correct):
spring.jpa.open-in-view=false  # ← exposes lazy loading problems

# Fix: don't use OSIV as a crutch. Instead:
# - Initialize all needed associations in the service layer (inside @Transactional)
# - Use JOIN FETCH or @EntityGraph
# - Use DTOs / projections that don't carry uninitialized proxies
```

**2. Accessing lazy association after transaction commits**
```java
@Service
public class OrderService {

    @Transactional
    public Order findOrder(Long id) {
        return orderRepo.findById(id).orElseThrow();
        // Transaction commits here when method returns
        // The returned Order has uninitialized lazy collections
    }
}

// In controller:
Order order = orderService.findOrder(1L);
order.getItems().size();  // ← LazyInitializationException: session is closed
```

**Fix A — Initialize inside the transaction:**
```java
@Transactional
public Order findOrderWithItems(Long id) {
    Order order = orderRepo.findById(id).orElseThrow();
    Hibernate.initialize(order.getItems());  // explicit initialization
    return order;
}
```

**Fix B — JOIN FETCH:**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

**Fix C — Use DTO projection (best for read-only APIs):**
```java
public record OrderDto(Long id, String status, List<ItemDto> items) {}

@Transactional(readOnly = true)
public OrderDto findOrder(Long id) {
    Order order = orderRepo.findByIdWithItems(id);
    return OrderMapper.toDto(order);  // map inside transaction
}
```

**3. `@Async` method accessing entity passed from calling thread**
```java
@Async
public void sendNotification(Order order) {
    // order was loaded in a different thread/transaction
    // session is closed here
    order.getCustomer().getEmail();  // LazyInitializationException!
}
// Fix: pass only the primitive values needed, not the full entity
@Async
public void sendNotification(String customerEmail, String orderStatus) { ... }
```

**Best Practices:**
- Set `spring.jpa.open-in-view=false` from day one — fail fast on lazy loading issues
- Design service methods to return DTOs, not managed entities, to the presentation layer
- Use `@Transactional(readOnly = true)` for read operations — better performance

---

## Microservices-Level Scenarios

---

### A downstream service is slow and your service also starts failing. How do you protect it?

**Core Explanation:**

Without protection, a slow downstream service causes thread pool exhaustion (bulkhead), which causes your service to fail completely. The cascade failure pattern. You need: timeouts, circuit breakers, bulkheads, and fallbacks.

**Layer-by-Layer Protection:**

**Layer 1 — Timeouts (always the first line of defense)**
```java
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
        .responseTimeout(Duration.ofSeconds(5));

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
```

**Layer 2 — Circuit Breaker (stops calling a failing service)**
```java
@Service
public class PaymentClient {

    @CircuitBreaker(name = "payment", fallbackMethod = "fallback")
    @TimeLimiter(name = "payment")
    public CompletableFuture<PaymentResult> charge(PaymentRequest req) {
        return webClient.post()
            .uri("/charge")
            .bodyValue(req)
            .retrieve()
            .bodyToMono(PaymentResult.class)
            .toFuture();
    }

    public CompletableFuture<PaymentResult> fallback(PaymentRequest req, Throwable t) {
        log.warn("Payment service unavailable, using fallback: {}", t.getMessage());
        return CompletableFuture.completedFuture(PaymentResult.pending(req.getOrderId()));
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        failureRateThreshold: 50          # open if 50% of last 10 calls fail
        waitDurationInOpenState: 30s       # wait 30s before trying again
        slidingWindowSize: 10
  timelimiter:
    instances:
      payment:
        timeoutDuration: 5s
```

**Layer 3 — Bulkhead (limits concurrent calls to downstream)**
```yaml
resilience4j:
  bulkhead:
    instances:
      payment:
        maxConcurrentCalls: 10       # max 10 threads calling payment service
        maxWaitDuration: 100ms       # others wait 100ms then fail fast
```

**Layer 4 — Retry with exponential backoff and jitter**
```yaml
resilience4j:
  retry:
    instances:
      payment:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        # Add jitter to prevent retry storms
        randomizedWaitFactor: 0.5
```

**Best Practices:**
- Order of decorators: `Retry(CircuitBreaker(RateLimiter(Bulkhead(TimeLimiter(fn)))))`
- Always implement fallbacks — degraded service is better than complete failure
- Expose circuit breaker state via `/actuator/health` so load balancers can route around failing instances

---

### Scaling instances didn't improve performance. What are the possible reasons?

**Core Explanation:**

Horizontal scaling (adding instances) only helps if the bottleneck is CPU/memory in the app layer. If the bottleneck is elsewhere, scaling does nothing or makes things worse.

**Possible Reasons:**

**1. Bottleneck is at the database (most common)**
```
Multiple app instances → all hit the same database
Database CPU/connection limit reached → adding more app instances just adds more DB pressure

Diagnosis:
- Monitor DB CPU: > 80% sustained
- Monitor DB connections: near max_connections
- EXPLAIN ANALYZE slow queries

Fix:
- Read replicas for read-heavy workloads
- Connection pooling via PgBouncer (at DB level, not app level)
- Query optimization / caching
- Database sharding (last resort)
```

**2. Shared external resource is the bottleneck**
```
- Single Redis instance (not cluster) — 100k ops/sec limit
- Single Kafka broker (not enough partitions)
- Shared NFS file system
- Rate-limited third-party API

Fix: scale the bottleneck resource, not the app
```

**3. Amdahl's Law — non-parallelizable portion limits scaling**
```
If 20% of work is serial (single-threaded, DB locks, distributed locks):
Max speedup = 1 / (0.20) = 5x, regardless of how many instances you add
```

**4. Sessions are not stateless — sticky sessions required**
```
# HttpSession stored in-memory → requests must go to the same instance
# Load balancer routes some requests to "wrong" instance
# Fix: use session replication (Spring Session + Redis) for stateless routing
```

**5. Load balancer misconfiguration**
```
# All traffic going to one instance (round-robin not working)
# Health checks failing on new instances → they never receive traffic
# Fix: verify load balancer target group health
```

**6. CPU throttling in Kubernetes**
```yaml
resources:
  limits:
    cpu: "100m"   # ← 0.1 CPU! Application is CPU-throttled
  requests:
    cpu: "100m"
# Fix: increase CPU limits or remove them and rely on requests only
```

**Best Practices:**
- Always identify the bottleneck before scaling (metrics-driven decision)
- Profile: is your bottleneck CPU, memory, I/O, network, or a shared dependency?
- USE method: Utilization, Saturation, Errors for every resource

---

### Circuit breaker is open even when the downstream service is healthy. What could be wrong?

**Core Explanation:**

A circuit breaker that won't close (stays open or Half-Open) when the downstream is healthy indicates a misconfiguration in failure thresholds, health check configuration, or probe logic.

**Possible Causes:**

**1. Half-Open state: permitted calls still failing**
```
Circuit breaker transitions: Open → Half-Open → (if probe calls succeed) → Closed
If the probe calls in Half-Open state fail, it stays open.

Possible reason: the probe calls are failing due to a different reason than the original outage:
- Timeout too short (service is slow to warm up after restart)
- Health check endpoint is different from actual endpoint behavior
- Connection pool still exhausted from previous failures
```

**2. Failure threshold too sensitive**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      downstream:
        slidingWindowSize: 5          # only 5 calls in window
        failureRateThreshold: 20      # open at 20% → 1 failure in 5 calls opens it
        # Fix: increase window size and/or threshold
        slidingWindowSize: 20
        failureRateThreshold: 50
```

**3. Exception types being counted incorrectly**
```java
// TimeoutException from your code is counted as failure
// But the service is actually healthy — your timeout is too short!
timeoutDuration: 1s  // ← service actually needs 2s on occasion
// Fix: tune timeout to p99 latency of the service, not p50
```

**4. Half-Open probe calls not enough**
```yaml
permittedNumberOfCallsInHalfOpenState: 1  # only 1 probe call
# If that 1 call fails (transient error), circuit stays open
# Fix: increase to 3-5 probe calls, require majority success
permittedNumberOfCallsInHalfOpenState: 5
```

**5. waitDurationInOpenState too long**
```yaml
waitDurationInOpenState: 60s  # waits 60s before trying Half-Open
# During this window, the service recovered but circuit is still open
# Monitoring shows circuit open; ops team thinks downstream is down
# Fix: reduce waitDuration, add alerting on circuit breaker state transitions
```

**6. Circuit breaker state not shared across instances (in-memory)**
```
Each app instance has its own circuit breaker state
Instance A: circuit open (saw failures)
Instance B: circuit closed (happens to not have seen failures)
→ Instance A never tries the service even though it recovered
Fix: use distributed circuit breaker state (Redis-backed) or accept per-instance state
```

**Best Practices:**
- Expose circuit breaker state via Actuator: `management.health.circuitbreakers.enabled=true`
- Alert on state transitions (Closed → Open) — this is a real incident signal
- Tune failure thresholds based on production traffic patterns, not theory

---

### A cache improves performance initially, then degrades it. What could cause this?

**Core Explanation:**

A cache that helps at first then hurts is a sign of cache poisoning, memory pressure, cache thrashing, or stale data causing extra work.

**Possible Causes:**

**1. Cache is too large — causing GC pressure**
```
Cache consumes most of heap → frequent GC pauses → worse than no cache
Fix: bound cache size, use off-heap cache (Caffeine with soft references, or Redis)
```

**2. Cache thrashing (working set larger than cache size)**
```
Cache size: 1,000 entries
Active data: 100,000 distinct keys → every request is a miss
Cache overhead (lock acquisition, ConcurrentHashMap operations) added to every request
With 0% hit rate, cache is pure overhead

Fix: profile hit rate: cache.stats().hitRate()
If hit rate < 50%, cache is doing more harm than good for that use case
```

**3. Cache stampede after expiration**
```
Popular entry expires → 1,000 concurrent requests all miss and hit DB
DB overwhelmed → response time spikes for all requests
After cache warms up, it helps; when entry expires, it hurts

Fix: probabilistic early expiration (PER algorithm):
if (timeToExpire() < -Math.log(random()) / beta) {
    refresh();  // refresh slightly early to prevent stampede
}
// Or: background refresh before expiration (Caffeine refreshAfterWrite)
```

**4. Memory pressure causes cache eviction → useful entries expelled**
```java
// Caffeine with softValues: JVM may evict entries under GC pressure
Cache<String, BigObject> cache = Caffeine.newBuilder()
    .softValues()  // ← entries evicted when JVM needs memory
    .build();
// Eviction rate increases as heap usage grows → cache effectiveness drops over time
// Fix: use maximumSize instead of softValues for predictable behavior
```

**5. Stale data causes extra validation calls (double reads)**
```java
// Cache-Aside with validation: check cache, then validate freshness with DB
// Ends up doing 2 DB queries for every cache hit (defeating the purpose)
// Fix: trust TTL, don't validate freshness unless absolutely required
```

**6. Cache serialization/deserialization overhead**
```java
// Remote cache (Redis): object must be serialized to JSON/Protobuf before storage
// and deserialized on retrieval
// For small, frequently-accessed objects, serialization cost > DB query cost

// Fix: use local (in-process) cache for hot, small data; Redis for large or distributed data
// Two-level caching: L1 = Caffeine (in-process), L2 = Redis (shared)
```

**Best Practices:**
- Always monitor cache hit rate — below 70% hit rate, re-evaluate the caching strategy
- Set explicit maximum size bounds on all caches
- Use `refreshAfterWrite` in Caffeine for popular entries to avoid stampede

---

### A retry mechanism caused system overload (retry storm). What design mistake was made?

**Core Explanation:**

A retry storm happens when many clients simultaneously retry failed requests, multiplying the load on an already-struggling service — the opposite of what retries should achieve.

**The Design Mistakes:**

**1. Fixed retry interval without jitter**
```java
// BAD: all clients retry at t+1s, t+2s, t+3s simultaneously
@Retry(maxAttempts = 3, backoff = @Backoff(delay = 1000))
public Response call() { ... }

// If 1,000 clients fail at t=0, they all retry at t=1s → 1,000 simultaneous retries
// Service just recovered → overwhelmed again by synchronized retry wave
```

**2. Fix: Exponential backoff with random jitter**
```java
// Jitter spreads retries across time, preventing synchronized waves
resilience4j:
  retry:
    instances:
      downstream:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2      # 500ms, 1s, 2s
        randomizedWaitFactor: 0.5            # ±50% jitter
        # Actual wait times: 250-750ms, 500ms-1.5s, 1s-3s
        # Spreads 1000 retries across a 3-second window
```

**3. Retrying non-idempotent operations**
```java
// POST /orders is not idempotent — retrying creates duplicate orders!
// Only retry safe/idempotent operations:
// - GET, HEAD, OPTIONS: always safe to retry
// - PUT, DELETE: idempotent by design (check your implementation!)
// - POST: only with idempotency key

// Fix: add idempotency key for POST retries
@Retry(retryExceptions = {IOException.class}, ignoreExceptions = {BusinessException.class})
```

**4. No circuit breaker — retrying a dead service forever**
```
Without circuit breaker:
- Service is down for 5 minutes
- Every request retries 3 times = 3x the load
- Service trying to recover gets 3x traffic → can't recover

Fix: always pair Retry with CircuitBreaker
Order: Retry → CircuitBreaker (CB opens → retries stop)
```

**5. No retry budget (unlimited retries)**
```
// maxAttempts = 10 with 3 retries each = 10 retries per request
// 1,000 concurrent requests × 10 retries = 10,000 calls to downstream
// Fix: keep maxAttempts low (2-3), rely on circuit breaker for longer outages
```

**Best Practices:**
- Golden rule: `Retry(exponential backoff + jitter) + CircuitBreaker + Bulkhead`
- Only retry on transient errors (network timeout, 503) — never on 4xx client errors
- Add retry count as a header/metric — alert if retry rate is high (sign of instability)

---

### One microservice deployment causes system-wide latency spike. How do you investigate?

**Core Explanation:**

A single service deployment causing system-wide impact means the service is on the critical path for many other services, or the deployment introduced a performance regression that amplifies upstream.

**Investigation Steps:**

**Step 1 — Identify the affected services**
```bash
# Distributed tracing (Jaeger/Zipkin): find traces with high latency
# Filter by service = recently-deployed service
# Check: does high latency correlate with the deployment time?

# Kubernetes deployment time:
kubectl rollout history deployment/payment-service
kubectl describe deployment/payment-service | grep "Last update"
```

**Step 2 — Compare metrics before/after deployment**
```
Key metrics to compare (use Grafana with deployment annotation):
- p99 response time for the deployed service
- Error rate for the deployed service
- CPU/memory for the deployed service
- Downstream service impact (call graph)
```

**Step 3 — Check for common deployment-induced regressions**
```
1. New database query added without index → N+1 or full table scan at scale
   Fix: EXPLAIN the new queries, add indexes

2. New @Transactional covering too much (holds DB connection longer)
   Fix: reduce transaction scope

3. New synchronous call to a slow external service added to hot path
   Fix: make it async or cached

4. Startup warmup period: new pods not warmed up (JIT not compiled yet)
   → First 1-2 minutes after deployment are slower
   Fix: add readiness probe that delays traffic until JVM is warmed
   Or: use GraalVM native image for consistent startup performance

5. New logging added (see excessive logging section)

6. Library upgrade with performance regression
   Fix: diff dependencies between old and new deployment
```

**Step 4 — Rollback if necessary**
```bash
# Quick rollback
kubectl rollout undo deployment/payment-service

# After rollback, verify latency returns to normal
# Then investigate the issue in staging before re-deploying
```

**Step 5 — Canary deployment for future protection**
```yaml
# Only route 5% of traffic to new version
# Monitor latency, error rate for 15 minutes before full rollout
# Use Argo Rollouts or Flagger for automated canary analysis
```

**Best Practices:**
- All deployments should have a deployment marker in your metrics dashboard
- Implement automated rollback triggers: if p99 > threshold after deploy → auto-rollback
- Performance test new queries in staging with production data volumes

---

### Two services update data that must stay consistent but you cannot use distributed transactions. How do you solve it?

**Core Explanation:**

Distributed transactions (2PC) are rarely used in microservices due to availability concerns and tight coupling. The standard solutions are: Saga pattern, Outbox pattern, and CQRS with eventual consistency.

**Solution 1 — Saga Pattern (choreography)**

```
Order Service publishes OrderCreated event
  ↓
Payment Service listens, charges card, publishes PaymentProcessed or PaymentFailed
  ↓
Inventory Service listens to PaymentProcessed, reserves stock, publishes StockReserved
  ↓
Order Service listens to StockReserved, marks order as confirmed

Compensation flow (if StockReserved fails):
Inventory publishes StockReservationFailed
  ↓
Payment Service listens, issues refund, publishes PaymentRefunded
  ↓
Order Service marks order as failed
```

**Solution 2 — Outbox Pattern (guaranteed event delivery)**

```java
@Transactional
public void placeOrder(OrderRequest req) {
    Order order = orderRepo.save(new Order(req));

    // Write event to outbox IN SAME TRANSACTION as the business data
    OutboxEvent event = new OutboxEvent(
        "ORDER_CREATED",
        objectMapper.writeValueAsString(new OrderCreatedPayload(order.getId()))
    );
    outboxRepo.save(event);
    // If either save fails, both roll back — atomic local operation
}

// Separate poller (or Debezium CDC) reads outbox and publishes to Kafka
@Scheduled(fixedDelay = 1000)
public void publishPendingEvents() {
    List<OutboxEvent> pending = outboxRepo.findByPublishedFalse();
    for (OutboxEvent event : pending) {
        kafka.send(event.getTopic(), event.getPayload());
        event.setPublished(true);
        outboxRepo.save(event);
    }
}
```

**Solution 3 — Idempotent consumers (safe retries)**

```java
// Each consumer records what it has processed
// If Kafka delivers the same message twice (at-least-once delivery),
// the second processing is a no-op
@KafkaListener(topics = "payment-events")
public void handlePayment(PaymentEvent event) {
    if (processedEventRepo.existsById(event.getEventId())) {
        return; // already processed, skip
    }

    processPayment(event);
    processedEventRepo.save(new ProcessedEvent(event.getEventId()));
}
```

**Consistency Guarantees:**

| Approach | Consistency | Availability | Notes |
|---|---|---|---|
| Saga (choreography) | Eventual | High | Complex rollback logic |
| Saga (orchestration) | Eventual | High | Centralized coordinator |
| Outbox + CDC | Eventual | High | At-least-once delivery |
| Idempotent consumer | Eventual | High | Handles duplicates safely |

**Best Practices:**
- Always pair Saga with idempotent consumers
- Use Outbox pattern to guarantee event delivery (don't publish events outside the DB transaction)
- Design compensating transactions before implementing the happy path

---

### Autoscaling is configured but performance does not improve under load. Why?

**Core Explanation:**

Kubernetes HPA (Horizontal Pod Autoscaler) may not trigger, trigger too late, or add pods that don't help if the bottleneck is not the scaled metric.

**Possible Reasons:**

**1. HPA metric not reflecting actual load**
```yaml
# HPA based on CPU utilization
# But your service is I/O-bound (waits on DB) — CPU stays low even under load
# CPU metric never triggers scale-out

# Fix: use the right metric
# For I/O-bound: scale on request rate or response time
# For message processing: scale on Kafka consumer lag
spec:
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
      target:
        type: AverageValue
        averageValue: "1000"
```

**2. Scale-out takes too long (pod startup time)**
```
Load spike at t=0
HPA detects CPU > threshold at t=30s (1 scrape interval)
HPA decides to scale at t=90s (stabilization window default: 5 min for scale-up)
New pods scheduled + started at t=3m
JVM startup + warmup at t=3m-5m
Actual service from new pods at t=5m
Load spike already passed or users have given up

Fix:
- Reduce stabilization window: spec.behavior.scaleUp.stabilizationWindowSeconds: 30
- Pre-warm with min replicas > 0
- Reduce JVM startup time (virtual threads, GraalVM native)
- Use KEDA for event-driven scaling (reacts faster)
```

**3. Bottleneck is at the database, not app tier**
```
Adding app instances → all query same DB → DB saturated
More instances = more DB connections = more DB load = worse performance

Diagnosis: monitor DB CPU and connection count during scale-out event
Fix: scale the DB too (read replicas), optimize queries, add caching
```

**4. Resource requests/limits too low — pods get CPU-throttled**
```yaml
resources:
  requests:
    cpu: "100m"   # 0.1 CPU — Kubernetes schedules HPA based on this
    memory: "256Mi"
  limits:
    cpu: "200m"   # throttled to 0.2 CPU even under load

# Fix: set realistic requests, increase limits (or remove CPU limit, keep memory limit)
```

**5. Anti-affinity misconfigured — all pods on same node**
```yaml
# If all pods land on one overloaded node, adding more pods doesn't help
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
# This spreads pods across nodes
```

**Best Practices:**
- Test autoscaling with load tests before relying on it in production
- Use KEDA (Kubernetes Event-Driven Autoscaler) for non-CPU metrics
- Scale preemptively if traffic patterns are predictable (cron-based scaling)

---

### Your service works fine standalone but fails behind an API Gateway. What do you check?

**Core Explanation:**

API Gateways transform requests and add constraints that your standalone service doesn't experience.

**Checklist:**

**1. Request/response size limits**
```
Gateway max body size: 1MB (default in many gateways)
Your service handles 10MB file uploads
→ 413 Payload Too Large from gateway, not from your service

Fix in Spring Cloud Gateway:
spring.cloud.gateway.httpclient.max-header-size=10MB
spring.cloud.gateway.httpclient.websocket.max-frame-payload-length=65536
```

**2. Timeout settings (shorter at gateway than at service)**
```
Service timeout: 30s
Gateway timeout: 5s → 504 Gateway Timeout after 5s
Service is fine but clients see errors

Fix: align timeouts, or use async pattern for long-running operations
```

**3. Header stripping**
```
# Gateway may strip headers your service relies on:
# X-Forwarded-For (real client IP)
# Authorization (if gateway terminates auth)
# X-Correlation-ID (tracing)

# Fix: configure gateway to pass-through needed headers
# Or use gateway-set headers: X-User-ID after JWT validation
```

**4. Path rewriting issues**
```yaml
# Request to gateway: /api/v1/orders
# Rewritten to service: /orders  (stripped /api/v1 prefix)
# But service expects: /api/v1/orders
# → 404 Not Found

# Spring Cloud Gateway path rewriting:
filters:
  - RewritePath=/api/v1/(?<remaining>.*), /${remaining}
```

**5. TLS termination — service expects HTTPS context**
```java
// Gateway terminates TLS, forwards HTTP to service
// Service uses request.isSecure() to build redirect URLs
// → redirect URLs are http:// instead of https://

// Fix:
server.forward-headers-strategy=NATIVE  # trust X-Forwarded-Proto header
# Or: server.forward-headers-strategy=FRAMEWORK
```

**6. CORS — gateway adds CORS headers, service also adds them → duplicate**
```
Error: "The 'Access-Control-Allow-Origin' header contains multiple values"
Fix: handle CORS only in the gateway, disable it in individual services
```

**7. Rate limiting / throttling at gateway**
```
Service sees requests randomly fail with 429 Too Many Requests
Fix: implement request retry with backoff, increase gateway rate limit, or
distribute client request rate more evenly
```

**Best Practices:**
- Test your service with a gateway in your local Docker Compose setup
- Always log gateway-assigned correlation IDs and propagate them downstream
- Make timeout values explicit and aligned across gateway → service → downstream

---

### A Kafka consumer is falling behind (consumer lag is growing). How do you diagnose and fix it?

**Core Explanation:**

Consumer lag grows when the consumer processes messages slower than the producer produces them. The root causes are: slow processing, insufficient consumer instances, or resource contention.

**Diagnosis:**

```bash
# Check consumer lag per partition
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group my-consumer-group --describe

# Output:
# GROUP           TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
# my-group        orders  0          5000             15000           10000  consumer-1
# my-group        orders  1          4900             15000           10100  consumer-2
# my-group        orders  2          0                15000           15000  --  ← UNASSIGNED!

# Unassigned partitions → not enough consumer instances
# Lag growing equally on all partitions → consumer too slow
# Lag concentrated on one partition → one slow consumer (check that instance)
```

**Root Cause 1: Not enough consumer instances**
```
Topic has 12 partitions but only 3 consumer instances
Each consumer handles 4 partitions
Fix: scale to 12 consumer instances (max parallelism = number of partitions)
```

```yaml
# K8s HPA based on consumer lag (using KEDA)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: order-consumer
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-consumer-group
      topic: orders
      lagThreshold: "1000"   # scale when lag per partition > 1000
      activationLagThreshold: "100"
```

**Root Cause 2: Processing each message takes too long**
```java
// Each message calls an external service synchronously
@KafkaListener(topics = "orders")
public void handle(OrderEvent event) {
    paymentService.charge(event);     // 500ms per call
    inventoryService.reserve(event);  // 300ms per call
    // Total: 800ms per message → can only process 75 messages/minute per consumer
}

// Fix: parallelize within the consumer
@KafkaListener(topics = "orders", concurrency = "3")  // 3 threads per consumer
// Or: batch processing
@KafkaListener(topics = "orders", containerFactory = "batchListenerFactory")
public void handleBatch(List<OrderEvent> events) {
    // process all events concurrently
}
```

**Root Cause 3: Consumer doing DB work on every message**
```java
// Fetching data from DB for each message
@KafkaListener(topics = "product-updates")
public void handle(ProductUpdate update) {
    Product p = productRepo.findById(update.getId());  // DB call per message!
    // Fix: batch fetch, local cache, or denormalize data in the message
}
```

**Root Cause 4: Message processing errors causing endless retries**
```java
// Poison pill message: one malformed message retries forever, blocking the partition
@KafkaListener(topics = "orders")
public void handle(OrderEvent event) {
    processOrder(event);  // throws uncaught exception → Kafka retries infinitely
}

// Fix: dead letter queue
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<?, ?> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);
    var backoff = new FixedBackOff(1000L, 3);  // 3 retries, 1s apart
    return new DefaultErrorHandler(recoverer, backoff);
}
```

**Root Cause 5: max.poll.records too high (consumer rebalance timeout)**
```properties
# Default: 500 records per poll
# If processing 500 records takes > max.poll.interval.ms (default 5 min):
# → Consumer is kicked out of group → rebalance → lag spike
spring.kafka.consumer.max-poll-records=100  # reduce batch size
spring.kafka.consumer.properties.max.poll.interval.ms=300000
```

**Best Practices:**
- Monitor consumer lag as a key metric — alert when lag grows consistently
- Use KEDA for lag-based auto-scaling of consumer deployments
- Implement dead-letter queues to prevent poison pills from blocking entire partitions
- Benchmark consumer throughput before go-live to size the consumer group correctly

---

# Section 19: Behavioral & Leadership Questions

*Note: These questions assess experience, judgment, and interpersonal skills. The answers below provide frameworks and example structures. Adapt them to your specific real-world experiences.*

---

## Self & Experience

---

### Tell me about yourself and your current project/role.

**Framework (STARS: Situation, Task, Action, Results, Skills):**

Structure your answer in three parts: who you are, what you currently work on, and where you're headed.

**Example Structure:**
```
Opening — Professional identity (2 sentences):
"I'm a Senior Java Developer with 6 years of experience building high-throughput
backend systems. My focus has been microservices architecture with Spring Boot,
distributed data systems, and Kubernetes-based cloud deployments."

Current Role — What you build and its scale (2-3 sentences):
"Currently at [Company], I'm the technical lead for our order processing platform —
a system that handles 50,000 transactions per day across 5 microservices.
I own the architecture, lead a team of 4, and work closely with Product to
translate business requirements into technical design."

Key Wins — One or two concrete achievements with numbers:
"Recently, I led an initiative that cut our API p99 latency from 800ms to 120ms
by introducing caching and fixing N+1 queries. I also built our Kafka-based
event sourcing pipeline, which eliminated a dual-write problem we'd lived with for two years."

What's Next — Signals genuine motivation for this role:
"I'm looking for an opportunity to architect systems at a larger scale and work
on harder distributed systems challenges — which is why this role caught my attention."
```

**Follow-Up Questions:**
- "What was the biggest technical challenge you faced recently?"
- "How do you decide what to work on next?"

---

### Walk through a production issue you debugged. What was the root cause and how did you fix it?

**Framework (STAR: Situation, Task, Action, Result):**

Pick an incident with a non-obvious root cause. Demonstrates debugging methodology, not just luck.

**Strong Answer Structure:**
```
Situation (what went wrong, impact):
"Six months ago, our payment service started throwing 503 errors for about
5% of requests. This was causing failed checkouts — estimated $10K/hour in lost revenue."

Task (your role):
"I was on call and owned the investigation."

Action (step-by-step debugging — this is the core of the answer):
"Step 1: Checked the error logs — 'HikariPool connection timeout' repeated pattern.
 Step 2: Thread dump — showed 30% of threads blocked at PaymentService.processRefund().
 Step 3: Found that a new feature shipped that day made a synchronous HTTP call
         to a third-party fraud API INSIDE a @Transactional method.
 Step 4: The fraud API had a 3-second p99 latency, holding DB connections for 3+ seconds.
 Step 5: Under normal load this was fine. But Black Friday surge caused all 10
         pool connections to be held → timeout."

Root Cause:
"External I/O inside a database transaction held connections for 3+ seconds."

Result:
"Hotfix: extracted the fraud check to BEFORE the transaction.
 503 rate dropped to 0% within 5 minutes.
 Long-term fix: added circuit breaker and 1-second timeout on fraud API,
 moved fraud check to async pre-processing step."

What I learned:
"Never make external HTTP calls inside @Transactional. We now have a code review
checklist item for this."
```

---

### Describe a microservices project you designed. What trade-offs did you make?

**Framework — Cover: decomposition decision, communication patterns, consistency model, operational choices.**

**Example Answer:**
```
Context:
"At my current company, we migrated a monolith e-commerce platform to microservices.
I led the design of the Order, Payment, and Inventory services."

Decomposition decision:
"We chose domain-driven design as the decomposition strategy — each service owned
one bounded context. The main debate was whether Order and Inventory should be
one service or two. We split them because they have different scaling needs:
Inventory is read-heavy, Order is write-heavy."

Communication trade-off:
"We chose async event-driven communication (Kafka) for Order → Inventory
and synchronous REST for customer-facing reads. Trade-off: async gives us
resilience but adds eventual consistency complexity. We had to implement
saga-based compensation and idempotent consumers."

Consistency trade-off:
"We accepted eventual consistency for inventory reservation. An order can briefly
show as confirmed before inventory is reserved, which required us to handle
over-commitment scenarios. We added a compensation flow for the rare case
where inventory is actually unavailable after order confirmation."

What I would do differently:
"I'd use the Outbox pattern from day one. We initially published Kafka events
directly after DB commits, which caused lost events during failures.
Retrofitting the Outbox pattern took 2 sprints."
```

---

### What is the most impactful technical improvement you made at your current company?

**Framework — Focus on measurable business impact, not just technical elegance.**

**Example Answer:**
```
Problem:
"Our analytics pipeline was a nightly batch job that imported data into a
data warehouse. Business stakeholders couldn't see real-time data, which made
it impossible to react to trends same-day."

What I did:
"I proposed and implemented a change data capture (CDC) pipeline using Debezium
on our PostgreSQL database, streaming changes to Kafka, and consuming them with
a Kafka Streams application that updated a read model in ElasticSearch in near real-time."

Technical challenges:
"Main challenge: handling schema evolution in Avro without breaking consumers.
We implemented backward-compatible schema changes and tested with contract testing."

Impact:
"Data latency went from 24 hours to under 5 minutes. Business team started
using this for real-time promotions during flash sales, which they estimate
increased promotional revenue by 15%. Also eliminated the fragile nightly job
that failed ~3 times per month."
```

---

### Describe a time when you chose a new technology or approach over the traditional one. What was the outcome?

**Framework — Highlight: evaluation process, risk management, outcome, and retrospective honesty.**

**Example Answer:**
```
Context:
"We needed to add real-time notifications to our platform. Traditional approach:
polling from the client every 5 seconds. I proposed WebSocket with Redis Pub/Sub
for multi-instance support."

Decision process:
"I built a proof of concept in 3 days. Tested with 10,000 concurrent connections
in a load test. Compared latency, resource usage, and complexity against polling."

Trade-offs I communicated:
"WebSocket is stateful — trickier to scale than REST. Requires sticky sessions or
shared subscription state (Redis). Added operational complexity. But: massively
better user experience and reduced server load (no 5-second polling)."

Outcome:
"After 6 months in production: 60% reduction in API calls, user engagement with
notifications up 40%. Zero major incidents related to the WebSocket layer."

Honesty:
"The one thing I underestimated was the K8s ingress configuration for WebSocket
upgrade headers. That took an extra day to sort out. I now add 'WebSocket support
verification' to the deployment checklist."
```

---

## Teamwork & Leadership

---

### Describe a situation where you showed technical leadership.

**Framework — Technical leadership = influencing without authority, making hard decisions, unblocking others.**

**Example Answer:**
```
Situation:
"Our team was about to start a 3-month project to build a custom distributed
caching layer. A senior engineer had already designed it and was excited about it."

My concern:
"After reviewing the design, I realized we were about to reinvent Caffeine + Redis —
a combination we already used elsewhere in the company. The custom solution would
also be harder to maintain."

How I handled it:
"I didn't dismiss his work. Instead, I scheduled a 1-hour design review where I
compared the custom solution to Caffeine + Redis on 5 dimensions: performance,
operational overhead, time to delivery, team familiarity, and long-term maintenance.
I prepared benchmarks and referenced our existing Redis usage."

Outcome:
"The team agreed to use Caffeine + Redis. We delivered in 3 weeks instead of 3 months.
The original engineer appreciated that I treated his work seriously and engaged
with data rather than just saying 'don't reinvent the wheel'."

What this taught me:
"Technical leadership is about helping the team make the best decision,
not about being right. Present data, not opinions."
```

---

### How do you mentor junior developers?

**Core Approach:**

Effective mentoring combines structured guidance, deliberate challenge, and psychological safety.

**Frameworks and Practices:**

```
1. Start with understanding their goals
"Before anything else, I ask: what do you want to get better at?
What's frustrating you right now? I tailor my mentoring to their goals,
not just what I think they need."

2. Teach debugging methodology, not just answers
"When they come with a bug, I don't just tell them the answer.
I walk through my thought process: 'What does this error mean?
What can we confirm or eliminate? How would you verify your hypothesis?'
This builds the debugging muscle, which is more valuable than any one answer."

3. Code review as a teaching tool
"I write detailed code review comments with explanations, not just
'this is wrong'. I link to documentation, explain the why,
and for complex cases, I open a Slack thread to discuss rather than
just leave a comment."

4. Assign stretch tasks with a safety net
"I give juniors tasks slightly above their current level,
but I check in daily during the first week. If they're stuck for > 30 minutes,
I want them to ask — not spend 2 hours spinning."

5. Retrospective on their PRs
"After a junior's PR is merged, I review it with them:
'What went well? What would you do differently?
What did you learn?' Reflection accelerates learning."
```

---

### How do you handle conflicts within a team?

**Core Principle:** Conflicts are information — they reveal misaligned assumptions, priorities, or information. The goal is resolution, not winning.

**Framework:**

```
Step 1: Listen to understand, not to respond
"When there's a conflict, I first make sure I understand each person's
actual concern — not their stated position. Often the real concern is different
from what's being argued about."

Step 2: Separate the technical from the interpersonal
"Technical disagreements (which technology to use) need data and criteria.
Interpersonal conflicts need empathy and acknowledgment of feelings.
Mixing the two makes both worse."

Step 3: For technical conflicts — establish criteria first
"Before arguing solutions, agree on criteria:
'We'll choose the option that best satisfies X, Y, and Z.'
Then evaluate both options against the criteria.
This depersonalizes the debate."

Example:
"Two senior engineers disagreed on whether to use REST or gRPC for internal
service communication. I facilitated a session where we first listed evaluation
criteria (developer familiarity, performance needs, schema evolution, tooling).
Then we scored each option. gRPC won on performance and schema evolution;
REST won on familiarity. The team chose gRPC for the high-traffic path
and REST for the admin APIs. Both engineers felt heard."

Step 4: For persistent interpersonal conflicts
"If two team members are genuinely struggling to work together,
I have a private conversation with each individually first,
then a joint conversation focused on working agreements, not grievances."
```

---

### Describe a time you disagreed with your manager. How did you handle it?

**Framework — Show: reasoned disagreement, professional delivery, and ability to commit after decision.**

**Example Answer:**
```
Situation:
"My manager wanted to ship a feature that I believed had significant performance
risks — we hadn't load tested it, and I estimated it would cause issues under
our peak load of 5,000 requests/minute."

What I did:
"I didn't disagree in the standup. I scheduled 30 minutes with my manager,
came prepared with: my specific concern, what data I would need to validate it,
and a proposal: 1 day of load testing before shipping, or alternatively,
ship behind a feature flag with gradual rollout."

The outcome:
"My manager appreciated the structured approach. We agreed on the feature flag
approach — it took 2 hours to implement and let us roll out to 5% of users first.
In the 5% cohort, we did see a performance issue, fixed it, and shipped cleanly."

What I learned:
"Disagreements are fine; how you raise them matters.
Come with data and a solution, not just a problem.
And commit fully once a decision is made — even if it's not your preferred option."
```

---

### How do you handle critical feedback?

**Core Principle:** Critical feedback is information. Your initial reaction is not your final response.

**Framework:**

```
1. Don't react immediately
"When I get feedback that stings, I take a breath before responding.
My first emotional reaction is usually not useful."

2. Separate signal from delivery
"Feedback can be poorly delivered but still contain valid signal.
I try to extract the useful part regardless of how it was framed."

3. Ask clarifying questions
"Can you give me a specific example? What would better look like to you?"

4. Thank them and give yourself time to process
"Even if my gut reaction is defensive, I say: 'Thank you for telling me this.
I want to think about it.' Then I actually do think about it."

5. Act on the valid parts
"If there's truth in the feedback, I change my behavior and make it visible:
'You mentioned my code reviews were too focused on style.
I've been focusing more on architecture concerns — does this feel different?'"

Example:
"Early in my career, a tech lead told me my PRs were hard to review because
they were too large. My immediate reaction was defensiveness — 'I was working on
a complex feature!' But after thinking about it, he was right.
I started breaking work into smaller, reviewable slices (< 400 lines per PR).
PR review time dropped from 2 days to 4 hours. That feedback improved my entire workflow."
```

---

### How do you conduct code reviews? What do you look for?

**Core Principle:** Code reviews serve two purposes: catching issues and transferring knowledge. Both are valuable.

**What I Look For (in priority order):**

```
1. Correctness — Does it actually work correctly?
   - Edge cases (null inputs, empty collections, concurrent access)
   - Error handling (are exceptions caught at the right level?)
   - Logic bugs (off-by-one, wrong operator, missing branch)

2. Security — Any new vulnerabilities?
   - SQL injection (parameterized queries?)
   - Authentication/authorization checks
   - Sensitive data in logs
   - Input validation at boundaries

3. Performance — Any obvious regressions?
   - N+1 queries (lazy loading in loops)
   - Unbounded collections (load entire table?)
   - Synchronous blocking in async context

4. Design — Does it fit the architecture?
   - Single responsibility
   - Appropriate layer for the logic (business logic in controller?)
   - Appropriate abstractions (premature or missing?)

5. Testability — Is it tested? Can it be tested?
   - Test coverage for new code paths
   - Tests that would catch regressions

6. Readability — Will the next developer understand this?
   - Variable/method names
   - Complex logic commented
   - Method length (> 50 lines is a signal)
```

**How I Conduct Reviews:**

```
- Acknowledge good work: "Nice approach here — I learned something."
- Distinguish blocking issues from suggestions:
  BLOCKING: "This will cause a NullPointerException when userId is null — please fix."
  SUGGESTION: "Consider — you could simplify this with Optional.ofNullable().
               Not blocking, just for readability."
- Ask questions instead of stating flaws:
  "What happens here when the list is empty?"
  (Better than: "This is wrong when the list is empty.")
- Be timely — aim to review within 24 hours
```

---

## Decision Making & Planning

---

### How do you prioritize tasks under tight deadlines?

**Framework — Combine urgency/importance matrix with stakeholder communication and scope negotiation.**

```
Step 1: Understand the real deadline and real requirements
"Before prioritizing, I ask: what's driving this deadline?
Is it a hard customer commitment, a marketing date, or internal pressure?
Understanding this reveals whether the deadline is negotiable."

Step 2: Identify the MVP (minimum viable product)
"With a tight deadline, I start from the deadline and work backward:
what's the minimum we need to ship to meet the business goal?
I distinguish: must-have (launch blocker) vs nice-to-have (can follow up)."

Step 3: Remove blockers, don't just work harder
"I look for what might block delivery: unclear requirements, unresolved tech decisions,
dependencies on other teams. I address blockers proactively."

Step 4: Communicate proactively
"If I think we can't make the deadline with full scope, I say so early —
with a counter-proposal: 'We can ship X and Y by the deadline; Z will be 2 weeks later.'
Late surprises are worse than early scope negotiation."

Step 5: Protect quality on the critical path
"Under time pressure, there's a temptation to skip tests and reviews.
I skip non-critical tests (edge case coverage), but never core path tests
and never security reviews. The cost of a production incident outweighs a delayed launch."
```

---

### How do you estimate technical tasks?

**Framework — Three-point estimation + decomposition + historical calibration.**

```
1. Break down before estimating
"I never estimate tasks larger than 2 days without decomposing them.
Estimation accuracy improves dramatically when tasks are < 1 day."

2. Three-point estimation
"For any task, I estimate three scenarios:
- Optimistic (everything goes right): 2 days
- Most likely (realistic): 4 days
- Pessimistic (unexpected complexity): 8 days
Expected = (O + 4M + P) / 6 = (2 + 16 + 8) / 6 ≈ 4.3 days"

3. Account for hidden work
"A feature estimate must include: implementation + unit tests + integration tests +
code review iterations + deployment + monitoring setup.
Pure implementation is often < 50% of total time."

4. Identify unknowns explicitly
"I separate known work from unknowns. For unknowns, I add a spike task:
'2-day investigation to understand the legacy payment integration' before
estimating the actual work."

5. Use historical velocity
"If our team typically takes 20% longer than estimated, I adjust.
Estimation is calibrated by feedback loops."

6. Communicate confidence intervals
"I give ranges: '3-5 days, depending on the complexity of the fraud API integration.'
Point estimates are false precision."
```

---

### How do you decide when to take on technical debt vs paying it off?

**Core Principle:** Technical debt is a financial instrument. Sometimes it's a good investment; sometimes it's high-interest debt that must be paid.

**Framework:**

```
1. Not all technical debt is equal
"I categorize debt:
- Reckless/inadvertent: 'We didn't know any better then.' HIGH priority to fix.
- Prudent/deliberate: 'We knew it was suboptimal but shipped to meet deadline.'
  Schedule payoff.
- Acceptable: 'This is in a stable area that rarely changes.' Low priority."

2. When to take on new debt (acceptable):
- Hard deadline with real business consequence
- Experimental feature (validate hypothesis before building it right)
- Throwaway code (proof of concept, not going to production)
Always: document the debt, create a ticket, assign a timeframe.

3. When to pay it off (urgently):
- Debt is blocking new feature development (team velocity declining)
- Debt is in a frequently-changed area (every change is dangerous)
- Debt is causing production incidents
- New team members can't understand the code

4. The 'boy scout rule' for maintenance debt:
"Leave the code slightly better than you found it.
Don't refactor everything on every PR, but fix small things in your path."

5. Explicit negotiation with stakeholders:
"Technical debt has business consequences: slower delivery, more bugs.
I make this visible to product: 'This area of code is slowing us down by X days
per sprint. Investing 2 sprints now will save us 1 sprint per quarter going forward.'"
```

---

### How do you use AI tools in your development and testing workflow?

**Honest, Thoughtful Answer:**

```
How I use AI tools:
"I use GitHub Copilot and Claude regularly. They're most valuable for:

1. Boilerplate and scaffolding
   Generating Spring Boot controllers, JPA entities, test class structure.
   AI is fast for the parts that are repetitive and mechanical.

2. Exploring unfamiliar APIs
   'Show me how to configure Resilience4j CircuitBreaker with Spring Boot 3.'
   Faster than reading documentation for initial orientation.

3. Explaining complex code
   'What does this Hibernate query cache configuration do?'
   Good for quickly understanding legacy code.

4. Test generation
   Generating test cases for known inputs — useful for covering edge cases
   I might not have thought of.

Where I'm careful:
1. AI code needs review — it can be subtly wrong, especially for security-sensitive code.
   I never blindly accept AI-generated authentication or authorization logic.

2. AI can give confident but outdated answers about framework versions.
   Always verify against official docs.

3. I don't use AI to avoid understanding the code.
   If I can't explain what AI generated, I don't ship it.

My principle:
AI handles the 'what', I handle the 'why' and 'is this right?'
I'm responsible for every line in my commits, regardless of who generated it."
```

---

### Why are you leaving your current company?

**Framework — Be honest, professional, and forward-looking. Never burn bridges.**

```
Acceptable Reasons to Express:
- Seeking larger-scale technical challenges
- Limited growth or advancement opportunities
- Company direction changed (acquired, pivoted away from technical work)
- Relocation or remote work policy change
- Looking for a team with stronger engineering culture

Example Answer (positive framing):
"I've learned a lot at [current company] and built systems I'm proud of.
After 4 years, I feel I've mastered the challenges at this scale.
I'm looking for the opportunity to work on larger-scale distributed systems
and grow into a more architectural role — which this position offers."

What to avoid:
- Don't complain about your manager or colleagues
- Don't mention salary as the only reason (even if it's true)
- Don't speak disparagingly about the company's technology
```

---

### Why do you want to join this company?

**Framework — Research the company. Generic answers reveal lack of genuine interest.**

```
Structure your answer around three elements:
1. Company-specific reason (what this company does that others don't)
2. Role-specific reason (what this role offers your growth)
3. Culture/team reason (what you know about how they work)

Example:
"I've been following [Company]'s engineering blog for the past year —
particularly the post about how you scaled your search infrastructure
to handle 10M queries/second using consistent hashing. That's the kind
of distributed systems challenge I want to work on.

The role specifically appeals to me because it involves both hands-on
architecture work and technical leadership of a team — I've been doing both
informally at my current company and want to do it officially.

And from speaking with [Name] in the first interview, it sounds like the
engineering culture here values autonomy and technical rigor, which matches
how I work best."
```

---

### Where do you see yourself in 3–5 years?

**Framework — Be honest about your direction without over-committing to a title. Align with where the role can take you.**

```
Strong Answer:
"In 3–5 years, I want to be someone who can own the architecture of a complex
distributed system from early design through production operation — including
the organizational side of getting a team aligned on technical direction.

I see myself growing toward a Staff or Principal Engineer role: not managing
people, but having architectural influence across multiple teams. I want to
be the person who catches systemic issues before they become incidents,
and who helps junior engineers grow into seniors.

I think this role is well-aligned with that trajectory — it has the scale
and complexity to develop those skills, and the team seems like one where
that kind of technical growth is valued."

What interviewers want to hear:
- You have a direction, not just a destination (title)
- Your direction aligns with what this role can offer
- You're invested enough in your craft to have thought about this
- You're not planning to leave in 6 months

What to avoid:
- "I want to be a manager" (if the role doesn't have that path)
- "I want to start my own company" (signals short tenure)
- "I don't know" (signals lack of self-direction)
```

---
