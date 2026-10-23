# Java Module 1 --- Java Fundamentals and Runtime Execution

**Academy:** KRB Academy\
**Repository:** `K_KnowledgeBase`\
**Suggested repository path:**
`01_Java/01_Fundamentals/01_java_fundamentals_and_execution.md`\
**Session date:** 23 October 2026\
**Status:** Core introductory session covered; use this document as the
review checkpoint for Module 1.

------------------------------------------------------------------------

## 1. Learning Objective

Understand how Java source code becomes JVM bytecode and how the JVM
loads, links, initializes, and executes a class. Learn to inspect
bytecode and class-file metadata with `javap`, and distinguish
compilation failures from class-loading/linking failures and runtime
exceptions.

## 2. Problem Statement

Java source code is not normally executed directly by the CPU. The Java
compiler translates source into class files containing bytecode and
metadata. A Java Virtual Machine (JVM) loads and executes that bytecode.

A backend engineer should understand what `javac` produces, what the
`.class` file contains, how symbolic references work, how static
initialization differs from object construction, and where failures
occur in the execution pipeline.

## 3. Environment Verified

``` text
java --version
openjdk 25.0.4 2026-07-21 LTS
OpenJDK Runtime Environment Temurin-25.0.4+7 (build 25.0.4+7-LTS)
OpenJDK 64-Bit Server VM Temurin-25.0.4+7 (build 25.0.4+7-LTS, mixed mode, sharing)

javac --version
javac 25.0.4

echo '$JAVA_HOME'
$JAVA_HOME

echo "$JAVA_HOME"
$HOME/.sdkman/candidates/java/current

which java
$HOME/.sdkman/candidates/java/current/bin/java

which javac
$HOME/.sdkman/candidates/java/current/bin/javac
```

The active JDK is Eclipse Temurin 25.0.4 managed through SDKMAN.

**Shell quoting observation:** Single quotes preserve literal text
(`echo '$JAVA_HOME'` prints `$JAVA_HOME`). Double quotes allow variable
expansion (`echo "$JAVA_HOME"` prints the variable's value).

## 4. Source → Bytecode → Execution

``` text
HelloWorld.java
    | javac HelloWorld.java
    v
HelloWorld.class
    | java HelloWorld
    v
JVM loads, links, initializes, and executes the class
    v
Hello, KRB Academy!
```

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `javac HelloWorld.java`             Compiles source into class files

  `java HelloWorld`                   Launches the class through the JVM

  `javap -c HelloWorld`               Displays bytecode instructions

  `javap -v HelloWorld`               Displays detailed class-file
                                      metadata, including the constant
                                      pool
  -----------------------------------------------------------------------

Launch with the class name, without `.class`.

## 5. First Program: `HelloWorld`

`HelloWorld.java`:

``` java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, KRB Academy!");
    }
}
```

Commands:

``` bash
javac HelloWorld.java
java HelloWorld
```

Observed output:

``` text
Hello, KRB Academy!
```

-   `main` is the entry-point method used by the Java launcher.
-   `static` means `main` can be invoked without constructing a
    `HelloWorld` object.
-   `String[] args` receives command-line arguments.
-   `System.out` is the standard output stream; `println` prints a line.

## 6. JDK, JVM, and JRE

-   **JVM:** Loads and executes Java class files according to the JVM
    specification.
-   **JDK:** Development kit containing tools such as `javac` and
    `java`, plus runtime components.
-   **JRE:** A term for the environment needed to run Java applications.
    Modern JDK distributions do not necessarily provide a separate
    independently installed JRE.

The verified environment is a JDK distribution: Eclipse Temurin 25.0.4.

## 7. Inspecting `HelloWorld.class`

Commands:

``` bash
javap -c HelloWorld
javap -v HelloWorld
```

The verbose output showed: - class-file major version `69` (Java 25); -
superclass `java/lang/Object`; - zero fields and two methods
(constructor and `main`); - constant-pool entries for `System.out`, the
string literal, and `PrintStream.println`; - method descriptors, flags,
code attributes, and line-number metadata.

### Relevant bytecode

``` text
public HelloWorld();
  Code:
     0: aload_0
     1: invokespecial #1 // Method java/lang/Object."<init>":()V
     4: return

public static void main(java.lang.String[]);
  Code:
     0: getstatic #7 // Field java/lang/System.out:Ljava/io/PrintStream;
     3: ldc #13 // String Hello, KRB Academy!
     5: invokevirtual #15 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     8: return
```

### Compiler-supplied constructor

Because the source declares no constructor, the compiler supplies a
default no-argument constructor. It calls the superclass constructor
before returning.

-   `aload_0`: loads local slot zero (`this`) in the instance
    constructor.
-   `invokespecial`: invokes `java.lang.Object.<init>()`.
-   `return`: exits the constructor.

If a class declares a constructor, the compiler does not also supply the
default constructor.

### `main` instructions

-   `getstatic #7`: obtains `System.out`.
-   `ldc #13`: loads the string constant.
-   `invokevirtual #15`: invokes `PrintStream.println(String)`.
-   `return`: exits `main`.

`#7`, `#13`, and `#15` are constant-pool indices---not memory addresses
or source line numbers.

### Constant pool and symbolic references

A `Methodref` entry describes a method symbolically through related
class and name-and-type entries. In this case, it identifies
`java.io.PrintStream.println` with the relevant descriptor. A `Fieldref`
similarly describes a field such as `java.lang.System.out`.

The JVM links symbolic references to runtime entities as needed. Class
files do not need to embed fixed native memory addresses for these
references.

### Method descriptors

  Descriptor                 Meaning
  -------------------------- ------------------------------------
  `()V`                      No parameters; returns `void`
  `([Ljava/lang/String;)V`   Takes a `String[]`; returns `void`
  `Ljava/lang/String;`       Reference to `java.lang.String`
  `[Ljava/lang/String;`      Array of `String` references
  `(I)Ljava/lang/String;`    Takes an `int`; returns a `String`

### `stack`, `locals`, and `args_size`

For `main`, the observed output reported
`stack=2, locals=1, args_size=1`.

-   `stack=2`: maximum operand-stack depth, measured in JVM slots.
-   `locals=1`: local-variable slots needed by the method. Since `main`
    is static, there is no implicit `this`; the `String[] args`
    parameter occupies one slot.
-   `args_size=1`: one method argument, `String[] args`.

The instance constructor showed one local slot and one argument because
its implicit `this` reference occupies a slot.

`LineNumberTable` maps bytecode offsets to source line numbers, helping
debugging tools relate execution to source code.

## 8. Where Java Programs Fail

### Compile-time error

Example: missing semicolon.

``` java
System.out.println("Hello")
```

`javac` reports a compilation error because the source is invalid Java
syntax. Other examples include incompatible types, undeclared variables,
incompatible method arguments, and access violations.

### Class-loading or linking failure

A class can compile successfully but fail when the JVM tries to load or
link a required class.

-   `ClassNotFoundException`: commonly thrown when code explicitly
    attempts to load a class by name and cannot find it.
-   `NoClassDefFoundError`: a class definition needed during execution
    could not be made available as expected. Causes can include missing
    runtime dependencies or a previous class-initialization failure.
-   `VerifyError`: bytecode verification failed.
-   `UnsupportedClassVersionError`: the runtime does not support the
    class-file version.

Inspect the full error, runtime classpath, dependencies, class-file
version, and any earlier initialization failure.

### Runtime exception

``` java
public class RuntimeError {
    public static void main(String[] args) {
        int result = 10 / 0;
        System.out.println(result);
    }
}
```

This compiles, but integer division by zero throws `ArithmeticException`
during execution. Other common runtime exceptions include
`NullPointerException`, `ArrayIndexOutOfBoundsException`,
`IllegalArgumentException`, and `NumberFormatException`. Not every
runtime failure is an exception; `OutOfMemoryError` is an `Error`.

  ------------------------------------------------------------------------
  Stage                   Example                  First area to inspect
  ----------------------- ------------------------ -----------------------
  Compilation             Missing semicolon        Source and compiler
                                                   output

  Loading/linking         `NoClassDefFoundError`   Runtime classpath,
                                                   dependencies,
                                                   initialization

  Execution               `ArithmeticException`    Stack trace, values,
                                                   program logic
  ------------------------------------------------------------------------

These are useful troubleshooting categories, not perfectly isolated
phases. Some linkage or initialization failures occur only when a
particular code path is reached.

## 9. JVM Class Lifecycle

The main stages are **loading, linking, initialization, and execution**.
Linking consists of verification, preparation, and resolution.
Resolution may be deferred until a reference is needed.

### Loading

The JVM locates and loads a class definition and creates its runtime
representation, including a corresponding `java.lang.Class` object.

Common built-in class loaders: - **Bootstrap:** loads core runtime
classes. - **Platform:** loads platform classes and modules. -
**Application/System:** loads application classes from configured
locations such as the classpath.

The normal delegation model asks a parent loader first, although custom
class loaders can behave differently.

### Linking

**Verification:** checks the class-file structure and bytecode
constraints.

**Preparation:** creates storage for class-level variables and assigns
default values. A static `int` initially has value `0`; a reference
field initially has value `null`.

**Resolution:** connects symbolic class, field, and method references to
their runtime entities.

### Initialization

Executes static field initializers and static initialization blocks in
textual order. Required superclass initialization occurs before subclass
initialization. A class is not necessarily initialized merely because
its class file exists; initialization occurs when JVM-specified
triggering conditions are met.

### Execution and JIT

The JVM executes bytecode. It may interpret instructions and use a
Just-In-Time (JIT) compiler to compile frequently executed code into
native machine code for optimization. Behavior depends on JVM
implementation, configuration, and workload. A tiny `HelloWorld` program
is not a useful demonstration of substantial JIT optimization.

## 10. Hands-on Experiment: `InitializationDemo`

Source file:

``` java
public class InitializationDemo {
    static int count = initializeCount();

    static {
        System.out.println("Static block executed");
    }

    static int initializeCount() {
        System.out.println("Static field initializer executed");
        return 10;
    }

    public static void main(String[] args) {
        System.out.println("main executed");
        System.out.println("count = " + count);
    }
}
```

Commands:

``` bash
javac InitializationDemo.java
java InitializationDemo
javap -c InitializationDemo
```

Observed output:

``` text
Static field initializer executed
Static block executed
main executed
count = 10
```

### What `javap -c` revealed

The class contained a constructor, `initializeCount()`,
`main(String[])`, and `static {}`. In the class-file format, the special
class initialization method is named `<clinit>`; `javap` displays it as
`static {}`.

Relevant bytecode from the user's actual output:

``` text
static {};
  Code:
     0: invokestatic  #33 // Method initializeCount:()I
     3: putstatic     #23 // Field count:I
     6: getstatic     #7  // Field java/lang/System.out:Ljava/io/PrintStream;
     9: ldc           #37 // String Static block executed
    11: invokevirtual #15 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    14: return
```

Interpretation: 1. `invokestatic` calls `initializeCount()` and receives
`10`. 2. `putstatic` assigns the value to static field `count`. 3. The
next instructions print `Static block executed`. 4. `return` finishes
class initialization.

### `<init>` versus `<clinit>`

  -----------------------------------------------------------------------
  Special method          Purpose                 When it runs
  ----------------------- ----------------------- -----------------------
  `<init>`                Initializes an object   When a constructor is
                          instance; corresponds   invoked
                          to a constructor        

  `<clinit>`              Initializes static      During class
                          class state             initialization
  -----------------------------------------------------------------------

Static initialization is associated with class initialization, not with
constructing each object.

### String concatenation and `invokedynamic`

The observed `main` bytecode contained:

``` text
14: invokedynamic #29, 0
    // InvokeDynamic #0:makeConcatWithConstants:(I)Ljava/lang/String;
```

This came from:

``` java
System.out.println("count = " + count);
```

In this Java 25 compilation, the compiler used an `invokedynamic` call
site associated with `makeConcatWithConstants` for string concatenation.
This supports runtime linkage for the operation. One source expression
can translate into several bytecode instructions or use JVM linkage
features.

## 11. Follow-up Experiment: Textual Initialization Order

The user confirmed completing this experiment, but did not paste its
output. The output below is therefore described as expected behavior,
not independently verified observed output.

``` java
public class InitializationOrderDemo {
    static int first = printAndReturn("first", 1);
    static int second = printAndReturn("second", 2);

    static {
        System.out.println("static block");
    }

    static int third = printAndReturn("third", 3);

    static int printAndReturn(String name, int value) {
        System.out.println("initialize " + name);
        return value;
    }

    public static void main(String[] args) {
        System.out.println(first + ", " + second + ", " + third);
    }
}
```

Commands:

``` bash
javac InitializationOrderDemo.java
java InitializationOrderDemo
javap -c InitializationOrderDemo
```

Expected ordering: initialize `first`, initialize `second`, execute the
static block, initialize `third`, then print `1, 2, 3` from `main`.

## 12. Mental Models

-   Source code is not the execution format: `javac` produces class
    files containing bytecode and metadata.
-   Class files are inspectable: `javap -c` shows instructions;
    `javap -v` shows structural metadata and constant-pool
    relationships.
-   Constant-pool indices are symbolic references, not memory addresses.
-   Compilation is not execution: valid source can still fail during
    loading, linking, initialization, or runtime execution.
-   `<clinit>` initializes static class state; `<init>` initializes
    instances.
-   Static initializers and static blocks execute in textual order.
-   The JVM may interpret bytecode and JIT-compile hot code.

## 13. Engineering Principles

1.  Diagnose the failure stage before changing unrelated code.
2.  Inspect compiler output rather than guessing how source code was
    translated.
3.  Distinguish class initialization from object construction when
    reasoning about static state.
4.  Remember that compile-time and runtime dependency availability can
    differ.
5.  Treat the observed `invokedynamic` sequence as an observation from
    this Java 25 compilation, not a universal bytecode promise for every
    compiler and JVM.

## 14. Interview Questions

1.  What is the difference between the JDK and JVM?
2.  What does `javac` generate?
3.  How do `javap -c` and `javap -v` differ?
4.  What does a constant-pool `Methodref` represent?
5.  What does `([Ljava/lang/String;)V` mean?
6.  Why can a class with no declared constructor still have a
    constructor in `javap` output?
7.  What do `stack`, `locals`, and `args_size` mean?
8.  What are the main stages of the JVM class lifecycle?
9.  What are verification, preparation, and resolution?
10. What is the difference between `<init>` and `<clinit>`?
11. When do static field initializers execute?
12. Why can compilation succeed while runtime class loading fails?
13. What is JIT compilation used for?
14. Why might Java 25 bytecode fail on an older runtime?
15. Why might the compiler use `invokedynamic` for string concatenation?

## 15. Session Completion

### Covered and practiced

-   [x] Verified Java 25 JDK and tool paths.
-   [x] Compiled and ran `HelloWorld`.
-   [x] Inspected bytecode with `javap -c`.
-   [x] Inspected class-file metadata with `javap -v`.
-   [x] Examined the default constructor, constant pool, descriptors,
    and method metadata.
-   [x] Distinguished compilation, class-loading/linking, and runtime
    failures.
-   [x] Learned the main JVM lifecycle stages.
-   [x] Ran `InitializationDemo` and observed static initialization
    order.
-   [x] Inspected class initialization bytecode and the `invokedynamic`
    string-concatenation call site.
-   [x] Completed the follow-up initialization-order experiment (user
    confirmed completion; output was not supplied).

### Revisit or extend later

-   [ ] Review the final JVM lifecycle knowledge-check answers if
    needed.
-   [ ] Demonstrate a classpath failure with a controlled example.
-   [ ] Explore JVM memory areas and the relationship between stack
    frames, heap objects, and class metadata.
-   [ ] Continue into Java language fundamentals and object-oriented
    programming in later lessons.

## 16. KRB Academy Progress

**Module:** `01_Java/01_Fundamentals`\
**Lesson:** `01_java_fundamentals_and_execution.md`\
**Status:** Core introductory session covered; use this as the starting
point for the next Java session. The overall Java curriculum is not
complete.
