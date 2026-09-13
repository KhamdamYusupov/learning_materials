# Mastering Python for Java Developers

> A deep learning guide for an experienced Java / Spring Boot backend developer who wants to become **comfortable and productive in Python**, especially for **AI / GenAI / AWS Bedrock** work.
> Not a cheat sheet. Not a syntax reference. Not a beginner course.
> The recurring question is always the same: **"I know how Java does this. How does Python *think* about it, and why?"**

**Versions used in this guide (verified current as of September 2026):**

| Technology | Version used | Notes |
|---|---|---|
| Python | **3.14** (3.14.7) — examples run on 3.12+ unless noted | 3.15 lands October 2026 |
| uv | 0.12.x | the package/project manager used throughout |
| FastAPI | 0.141.x | install as `fastapi[standard]` |
| Pydantic | 2.13.x | Pydantic **v2** API only (`model_validate`, `model_dump`, ...) |
| pydantic-settings | 2.x | configuration from env / `.env` |
| SQLAlchemy | 2.0.x (2.1 is at release-candidate stage) | 2.0-style API only (`Mapped`, `mapped_column`, `select()`) |
| Alembic | 1.x | migrations |
| psycopg | 3.x (`psycopg[binary]`) | PostgreSQL driver |
| pytest | 9.1.x | |
| httpx | 0.28.x | HTTP client (sync + async) |
| NumPy / pandas | 2.5.x / 3.0.x | |
| scikit-learn | 1.7+ | |
| boto3 | 1.43.x | AWS SDK for Python |
| anthropic (Python SDK) | 1.x | released August 2026, built on `httpx2` |
| openai (Python SDK) | 2.x | Responses API |
| LangChain | 1.3.x (`langchain`, `langchain-core`, `langchain-aws`) | 1.x API (`create_agent`, `init_chat_model`) |
| LlamaIndex | 0.14.x (`llama-index-core`) | |
| Hugging Face Transformers | 5.x | PyTorch-only since v5 |
| PyTorch | 2.13.x | |

> ⚠️ **Version note.** Python tutorials on the internet age badly. If you find code with `from typing import List, Dict`, `Optional[str]` everywhere, `pip install` into the global interpreter, `requirements.txt` as the only manifest, Pydantic `.dict()` / `@validator`, SQLAlchemy `Column(...)` + `session.query(...)`, or LangChain `LLMChain` / `AgentExecutor` — it is describing an older world. This guide uses the current way and points out the legacy forms so you can recognise them when you meet them in real repositories.

---

## How to read this guide

You are not a beginner, so the guide does not start from "what is a variable". It starts from **how Python works** and **why it is designed the way it is**, because that is what actually makes Python feel foreign to a Java developer. Syntax is easy; the mental model is the hard part.

Every important concept follows the same rhythm:

```text
What is it?  →  Why does Python have it?  →  How does it work?  →  Java comparison  →  Example  →  Real-world usage
```

and every chapter ends with a short **Knowledge check** — conceptual questions, not trivia. If you can answer them in your own words you have the mental model; if not, re-read that chapter before moving on.

The guide is organised as a journey:

```text
Part I    — How Python thinks              (Chapters 1–3)
Part II   — The language, properly         (Chapters 4–12)
Part III  — Writing Python like a Python developer (Chapters 13–17)
Part IV   — Backend engineering in Python  (Chapters 18–24)
Part V    — Python for data, ML and GenAI  (Chapters 25–31)
Part VI   — Real projects                  (Chapter 32)
Part VII  — Becoming productive            (Chapters 33–38)
```

Recommended pace: read Parts I–III carefully with a terminal open; skim Part IV once and come back when you build Project 2; read Part V slowly; build every project in Part VI.

---

## Table of Contents

**Part I — How Python thinks**
1. [The Python Mental Model](#1-the-python-mental-model)
2. [Installation and the Professional Development Environment](#2-installation-and-the-professional-development-environment)
3. [Syntax and Basic Language Features](#3-syntax-and-basic-language-features)

**Part II — The language, properly**
4. [Collections](#4-collections)
5. [Functions](#5-functions)
6. [Python's Object Model](#6-pythons-object-model)
7. [Interfaces, Abstract Classes and Protocols](#7-interfaces-abstract-classes-and-protocols)
8. [Type Hints](#8-type-hints)
9. [Dataclasses and Pydantic](#9-dataclasses-and-pydantic)
10. [Modules and Packages](#10-modules-and-packages)
11. [Exceptions](#11-exceptions)
12. [Files, JSON and HTTP](#12-files-json-and-http)

**Part III — Writing Python like a Python developer**
13. [Pythonic Programming](#13-pythonic-programming)
14. [Iterators and Generators](#14-iterators-and-generators)
15. [Decorators](#15-decorators)
16. [Context Managers](#16-context-managers)
17. [Concurrency, Async Python and the GIL](#17-concurrency-async-python-and-the-gil)

**Part IV — Backend engineering in Python**
18. [Testing with pytest](#18-testing-with-pytest)
19. [Logging and Configuration](#19-logging-and-configuration)
20. [Python Web Frameworks (FastAPI focus)](#20-python-web-frameworks-fastapi-focus)
21. [FastAPI Project — Customer API](#21-fastapi-project--customer-api)
22. [Databases — SQLAlchemy and Alembic](#22-databases--sqlalchemy-and-alembic)
23. [Dependency Management](#23-dependency-management)
24. [Python Project Structure](#24-python-project-structure)

**Part V — Python for data, ML and GenAI**
25. [Python for Data and AI — NumPy, pandas, Jupyter](#25-python-for-data-and-ai--numpy-pandas-jupyter)
26. [Python and Machine Learning — scikit-learn](#26-python-and-machine-learning--scikit-learn)
27. [Python for Generative AI — the ecosystem](#27-python-for-generative-ai--the-ecosystem)
28. [Calling LLMs from Python](#28-calling-llms-from-python)
29. [Python + AWS for AI — boto3 and Amazon Bedrock](#29-python--aws-for-ai--boto3-and-amazon-bedrock)
30. [Python AI Frameworks — LangChain, LlamaIndex, Hugging Face, PyTorch](#30-python-ai-frameworks--langchain-llamaindex-hugging-face-pytorch)
31. [Python vs Java for AI Development](#31-python-vs-java-for-ai-development)

**Part VI — Real projects**
32. [Real Projects](#32-real-projects)
    - [Project 1 — Python CLI](#project-1--python-cli)
    - [Project 2 — REST API](#project-2--rest-api)
    - [Project 3 — AI Chat API](#project-3--ai-chat-api)
    - [Project 4 — RAG Document Q&A](#project-4--rag-document-qa)
    - [Project 5 — AWS GenAI Application](#project-5--aws-genai-application)
    - [Project 6 — Python AI Service + Java Backend](#project-6--python-ai-service--java-backend)

**Part VII — Becoming productive**
33. [Common Mistakes Java Developers Make in Python](#33-common-mistakes-java-developers-make-in-python)
34. [What a Java Developer Should NOT Learn Yet](#34-what-a-java-developer-should-not-learn-yet)
35. [Practical Learning Path](#35-practical-learning-path)
36. [Final Knowledge Check](#36-final-knowledge-check)
37. [Final Goal — What Do I Now Know About Python?](#37-final-goal--what-do-i-now-know-about-python)

---

# Part I — How Python thinks

---

# 1. The Python Mental Model

Before a single line of syntax, you need to know **what Python fundamentally is**, because almost everything that feels strange later (no type declarations, `self`, no interfaces, the GIL, "everything is an object") follows directly from a few design decisions made at the runtime level.

## 1.1 Python as a language vs Python as an implementation

**What is it?**
"Python" is two things:

1. **The language** — a specification (grammar, semantics, the standard library contract) maintained by the Python core team through PEPs (Python Enhancement Proposals, the equivalent of JEPs/JSRs).
2. **An implementation** — a program that executes Python code. The reference implementation is **CPython**, written in C. When you type `python`, you are almost certainly running CPython.

**Java comparison.** Java has the same split: the Java Language Specification vs. HotSpot / OpenJ9 / GraalVM. The difference is cultural: Java developers switch JVMs occasionally (GraalVM for native images); Python developers use CPython 99% of the time. Alternatives exist and you should know their names so you are not confused when you meet them:

| Implementation | What it is | When you'd meet it |
|---|---|---|
| **CPython** | Reference implementation in C. The default. | Always. |
| **PyPy** | JIT-compiled implementation, often 3–10× faster on pure-Python loops. | Rarely, in performance-sensitive pure-Python code. Not for NumPy/PyTorch-heavy work. |
| **Jython / GraalPy** | Python on the JVM. | Almost never in practice; GraalPy is interesting for Java interop experiments. |
| **MicroPython / CircuitPython** | Python for microcontrollers. | Embedded. |
| **Pyodide** | CPython compiled to WebAssembly. | Python in the browser (Jupyter-lite). |

**Practical consequence:** when this guide says "Python does X internally", it means "CPython does X". The GIL, reference counting, `.pyc` files — all CPython details. Other implementations may differ, and nobody will ask you about them in an AI project.

## 1.2 The execution model: how `.py` becomes running code

**What is it?**
Python is usually called an "interpreted language". That is only half-true. CPython *compiles* your source to **bytecode** and then *interprets* that bytecode on a **virtual machine**. It is closer to Java than most people think — the big difference is *when* the compilation happens and *what the bytecode knows about types*.

**Java:**

```text
Person.java
    ↓  javac (ahead of time, separate step, type-checks everything)
Person.class (bytecode with static types baked in)
    ↓  JVM loads the class
    ↓  interpreter runs bytecode
    ↓  JIT (C1/C2) compiles hot paths to native machine code
machine code
```

**Python (CPython):**

```text
person.py
    ↓  compiler inside the interpreter (at import/run time, no type checking)
bytecode (cached as __pycache__/person.cpython-314.pyc)
    ↓  Python Virtual Machine (the "eval loop", ceval.c)
execution — every operation dispatches on the *runtime* type of each object
```

**How does it work?**

1. **Parsing → AST.** CPython reads the source, tokenises it, builds an abstract syntax tree. Syntax errors are found here. *No type errors are found here* — `1 + "a"` parses fine.
2. **Compilation → bytecode.** The AST is compiled into a `code` object: a sequence of simple instructions (`LOAD_FAST`, `BINARY_OP`, `CALL`, `RETURN_VALUE`...). You can see it:

   ```python
   import dis

   def add(a, b):
       return a + b

   dis.dis(add)
   #   RESUME                   0
   #   LOAD_FAST_BORROW_LOAD_FAST_BORROW 0 (a, b)   # 3.14 combines instructions
   #   BINARY_OP                0 (+)
   #   RETURN_VALUE
   ```

   Compare with `javap -c`. The key difference: `BINARY_OP (+)` does not know whether `a` is an `int`, a `str` or your own class. It will find out **at runtime** by asking the objects.
3. **Caching → `.pyc`.** When a module is *imported*, CPython writes the bytecode to `__pycache__/module.cpython-314.pyc` so the next import skips compilation. This is purely a startup optimisation — it is not like a JAR, you never ship `.pyc` files deliberately, and you should add `__pycache__/` to `.gitignore`. The script you run directly (`python main.py`) is not cached.
4. **Execution → the Python VM.** The eval loop reads one bytecode instruction at a time and executes it. Since Python 3.11 there is a "specialising adaptive interpreter" that rewrites hot instructions into type-specialised versions (e.g. `BINARY_OP_ADD_INT`) after observing that the operands are always ints — a lightweight cousin of what the JIT does in HotSpot. Python 3.13/3.14 also ship an experimental copy-and-patch JIT, disabled by default. In practice: **CPython is still 10–100× slower than the JVM on pure-Python CPU-bound loops**, and the ecosystem is designed around that fact — the heavy lifting in NumPy, PyTorch, Pydantic (`pydantic-core`) and uv is written in C, C++ or Rust and Python is the *orchestration* layer.

**Java comparison — the honest version.**

| | Java | Python |
|---|---|---|
| Compilation step visible to you? | Yes (`javac`, `mvn compile`) | No — happens automatically at import |
| Type checking at compile time? | Yes, mandatory | No (optional external tools, see Chapter 8) |
| Bytecode knows static types? | Yes | No — instructions dispatch on runtime types |
| Startup | Slow (JVM warm-up) | Fast (ms) — why Python wins for CLIs and Lambda |
| Steady-state speed of pure code | Very fast (JIT to native) | Slow (interpreted, partial specialisation) |
| Speed of numeric / ML code | Depends on libraries | Fast — because the work happens in C/CUDA, not in Python |
| Threads run in parallel? | Yes | Not for Python bytecode in the default build (GIL — Chapter 17) |

> **Mental model to keep:** *Python is a fast-to-write, slow-to-run glue language sitting on top of very fast native libraries.* Once you accept that, the whole AI ecosystem makes sense: the "Python" in a PyTorch training loop is a few hundred lines of orchestration; the billions of floating-point operations happen in CUDA kernels.

## 1.3 The runtime: what exists while your program runs

In the JVM you think about the heap, the stack per thread, the class loader, the metaspace. CPython's runtime has fewer moving parts:

```text
┌──────────────────────────────────────────────────────────┐
│ CPython process                                          │
│                                                          │
│  ┌──────────────┐   ┌───────────────────────────────┐    │
│  │ Interpreter  │   │ Object heap (private allocator │    │
│  │ state        │   │ + malloc for big objects)      │    │
│  │ - sys.modules│   │  every int, str, list, function│    │
│  │   (loaded    │   │  class, module is an object    │    │
│  │    modules)  │   │  with a refcount + type ptr    │    │
│  │ - builtins   │   └───────────────────────────────┘    │
│  └──────────────┘                                        │
│  ┌──────────────┐  ┌──────────────┐                      │
│  │ Thread 1     │  │ Thread 2     │   ← OS threads, but  │
│  │ frame stack  │  │ frame stack  │     only one executes│
│  └──────────────┘  └──────────────┘     bytecode at a    │
│                 GIL (default build)      time             │
└──────────────────────────────────────────────────────────┘
```

- **`sys.modules`** — the dictionary of every module imported so far. Importing a module *executes* the file once and stores the resulting module object here. This is your "class loader", except modules are plain objects you can inspect and even monkey-patch (Chapter 10).
- **Frames** — each function call pushes a frame object with its local variables. `traceback` prints this stack. Since 3.11 frames are cheap C structs; conceptually they are still visible objects (`inspect.currentframe()`).
- **Objects** — *everything* is a heap-allocated object with a header: a reference count and a pointer to its type. Even the integer `5`. Even the function `add`. Even the class `User`. There are no primitives.

## 1.4 Dynamic typing (and why Python doesn't need declarations)

**What is it?**
In Python, **variables have no type. Values have types.** A variable is a *name bound to an object*. Re-binding it to an object of a different type is legal:

```python
x = 42        # the name x → an int object
x = "hello"   # the name x → a str object (the int is now unreferenced)
```

**Why does Python have it?**
Because the design goal was a language where you write the *what* and let the runtime handle bookkeeping. Guido van Rossum designed Python as a scripting language between shell and C; types on every variable were friction that did not pay for itself in that setting. Dynamic typing plus everything-is-an-object gives Python its flexibility: the same function can accept a list, a generator, or a database cursor as long as they *behave* the same (see duck typing below).

**How does it work?**
Java's `String name = "John";` allocates a *slot* of type `String` on the stack; the compiler guarantees only `String` references ever go there. Python's `name = "John"` creates (or reuses) a `str` object on the heap and puts an entry `"name" → <that object>` in the current namespace — literally a dictionary for globals, an array slot for function locals. There is no slot type. `type(name)` asks the *object*, not the variable.

```python
>>> name = "John"
>>> type(name)
<class 'str'>
>>> name = 30
>>> type(name)
<class 'int'>
```

**Java comparison.**

```java
String name = "John";   // slot typed String; compiler enforces
int age = 30;           // primitive slot on the stack
```

```python
name = "John"           # name → str object on the heap
age = 30                # age  → int object on the heap (no primitive)
```

Java's `var name = "John";` is *still static typing* — the compiler infers `String` and locks it. Python never locks anything. That is the difference between **type inference** (Java `var`, Kotlin, TypeScript) and **dynamic typing** (Python, JavaScript, Ruby).

**What dynamic typing actually means for you:**

- Mistakes like passing a `str` where an `int` was expected are found **when that line runs**, not when you build. Your tests and a type checker (Chapter 8) replace the compiler.
- Refactoring tools are weaker than IntelliJ's for Java, because the tool cannot always know what a name refers to. Type hints fix most of this.
- Functions are naturally "generic": `def first(items): return items[0]` works for any indexable thing without `<T>`.

## 1.5 Strong typing (Python is *not* loosely typed)

Dynamic ≠ weak. Python is **strongly typed**: it never silently coerces unrelated types.

```python
>>> 1 + "1"
TypeError: unsupported operand type(s) for +: 'int' and 'str'
>>> "3" * 2
'33'            # str * int is *defined* (repetition) — not a coercion
>>> 1 + 1.5
2.5             # int→float promotion is defined by the numeric tower
```

JavaScript would give you `"11"`. Java gives you `"11"` too (`1 + "1"` is string concatenation by language rule). Python refuses. The value's type is respected at all times; only explicit conversions (`int("1")`, `str(1)`) change type.

## 1.6 Duck typing — the core idea behind Python's object model

**What is it?**
> "If it walks like a duck and quacks like a duck, it's a duck."

A function that calls `.read()` on its argument works with *anything that has a `.read()` method*: a file, a socket, an `io.StringIO`, an S3 streaming body, your own class. Nobody declares `implements Readable`. The check happens at the moment `.read()` is called, on the actual object.

**Why does Python have it?**
It is the natural consequence of dynamic typing plus attribute lookup by name. Python objects are essentially dictionaries of attributes; calling `obj.read()` means "look up `read` on this object and call it". Whether the object's *class* promised to have `read` is irrelevant.

**Java comparison.** Java is **nominally typed**: a `FileInputStream` is an `InputStream` because it *says so* in its declaration. To make your class usable by `BufferedReader`, you must `extends Reader`. Python is **structurally typed at runtime**: compatibility comes from *having the right attributes*, not from the family tree. Python's `Protocol` (Chapter 7) brings structural typing to the *type checker* too — the closest thing to a Java interface, but still without `implements`.

**Real-world usage.** This is why Python libraries compose so easily. `json.dump(obj, fp)` accepts any `fp` with `.write()`. pandas `read_csv()` accepts a path, a URL, a file object, or a `StringIO`. Pydantic validates any object with the right attributes when `from_attributes=True`. You will write much less adapter code than in Java.

## 1.7 Memory management: reference counting + cycle GC

**What is it?**
CPython frees objects using **reference counting** as the primary mechanism, with a **generational cycle collector** as backup.

**How does it work?**

- Every object has a counter of how many references point to it. `x = obj` increments, `del x` or rebinding or a scope ending decrements. **When the count hits zero the object is freed immediately** — deterministically, on that exact line.
- Reference counting cannot free **cycles** (`a.other = b; b.other = a` with nothing else pointing at them — both counts stay at 1). The cyclic GC periodically scans container objects (lists, dicts, instances) to find unreachable cycles and frees them. It is generational (young objects scanned often, old ones rarely), triggered by allocation thresholds, tunable via `gc` module.

```python
import sys

a = []
print(sys.getrefcount(a))   # 2 (the variable a + the temporary argument)
b = a
print(sys.getrefcount(a))   # 3
del b
print(sys.getrefcount(a))   # 2
```

**Java comparison.** The JVM has *no* reference counting; everything is tracing GC (G1, ZGC...) that runs at its own pace. Consequences:

| | Java | CPython |
|---|---|---|
| When is an unreferenced object freed? | Eventually, at the next GC | Immediately (unless in a cycle) |
| Deterministic destructors? | No (`finalize` is unreliable; use try-with-resources) | Mostly yes (`__del__` runs at refcount 0) — but *still* use `with` (Chapter 16), because cycles and exceptions make it unreliable |
| GC pauses | Tunable, can be an ops concern | Small; the cycle collector is rarely a problem |
| Memory overhead per object | Header ~12–16 bytes; primitives free | Header 16 bytes + type-specific payload; a Python `int` is ~28 bytes, a `float` 24 bytes. `[1, 2, 3]` is a list of *pointers* to three int objects |
| Why the GIL exists | n/a | Reference counts are non-atomic integers updated everywhere; the GIL is the cheapest way to keep them consistent across threads (Chapter 17) |

That last row matters: the GIL is not an accident, it is the *price* of reference counting. The free-threaded build (Python 3.13+, officially supported in 3.14, binary `python3.14t`) replaces it with per-object locks and biased reference counting, at a single-thread cost — Chapter 17 covers when that matters.

**Memory allocator.** CPython uses `pymalloc`, an arena-based small-object allocator, for objects < 512 bytes, and the system `malloc` for bigger ones. Small ints (-5..256) and interned strings are shared singletons — which is why `a = 5; b = 5; a is b` is `True` but `a = 1000; b = 1000; a is b` may not be. Do not rely on it; use `==` for values and `is` only for `None`/`True`/`False`/identity checks.

## 1.8 Putting the mental model together

Here is what happens when you run this file:

```python
# app.py
import json

class User:
    def __init__(self, name):
        self.name = name

def greet(user):
    return f"Hello {user.name}"

if __name__ == "__main__":
    u = User("Ada")
    print(greet(u))
```

1. `python app.py` starts CPython, creates the interpreter state, populates `sys.modules` with built-ins.
2. `app.py` is parsed and compiled to bytecode (not cached — it is the main script).
3. The bytecode of the *module body* executes top to bottom, **as statements** — this is the part Java developers miss. `import json` runs the `json` package's `__init__.py` (or loads its `.pyc`) and binds the name `json`. `class User:` **executes** the class body and creates a `type` object named `User` and binds it. `def greet` creates a function object and binds it. `if __name__ == "__main__":` is an ordinary `if`, checking a module-level variable set by the interpreter.
4. `User("Ada")` calls the class object (classes are callable): `type.__call__` allocates an instance, calls `__init__(instance, "Ada")`, returns the instance.
5. `greet(u)` looks up the name `greet` in module globals, calls it, `user.name` looks up the attribute `name` in the instance's `__dict__`, formats the string.
6. When `main` finishes, refcounts drop to zero, objects are freed, the interpreter shuts down.

Nothing was type-checked. Nothing was declared. Every step was "look up a name at runtime and do something with the object you found". **That is Python.**

## Knowledge check — Chapter 1

1. Explain, in two sentences, why Python does not require `String name = "John";`. What is typed in Python, the variable or the value?
2. Both Java and Python compile to bytecode. What is the crucial difference in what that bytecode *knows*?
3. Is Python weakly typed? Give a one-line example that proves your answer.
4. Why is `.pyc` not analogous to a `.jar`?
5. Why does the existence of reference counting explain the existence of the GIL?
6. A colleague says "Python is slow, so it cannot be used for machine learning." What is wrong with that reasoning?

---

# 2. Installation and the Professional Development Environment

Java has one story: install a JDK, use Maven or Gradle, dependencies land in `~/.m2` and are resolved per project. Python's story used to be a mess of competing tools; since 2024 it has converged on one clear modern workflow — **uv + `pyproject.toml`** — with older tools still everywhere in the wild. This chapter teaches the modern workflow and tells you just enough about the alternatives to read any repository.

## 2.1 Python versions and the `python` / `python3` confusion

**What is it?** Python releases a new minor version every October (3.12 → Oct 2023, 3.13 → Oct 2024, 3.14 → Oct 2025, 3.15 → Oct 2026). Each minor gets 2 years of bug fixes and 5 years of security fixes. Unlike Java there is no LTS concept; the ecosystem generally supports the last 4–5 minors.

**The `python` vs `python3` mess:** on Linux/macOS the system often ships a `python3` binary (used by the OS itself — never install packages into it), and `python` may not exist or may point to it. Historically `python` meant Python 2. Rules:

- **Never use the OS's Python for projects.** It is for the OS. `sudo pip install` is the Python equivalent of editing the JDK's `lib` directory.
- **Let your project tool manage interpreters.** `uv python install 3.14` downloads a standalone build into `~/.local/share/uv/python/`; `uv` picks the right one per project from `.python-version` / `pyproject.toml`. This is exactly the role of SDKMAN or `asdf` for Java.
- Inside a virtual environment, `python` always means "this environment's interpreter". That is the only `python` you should type.

Which version to target: **3.12 as a floor, 3.13/3.14 for new projects.** Free-threaded builds (`3.14t`) are still a specialised choice (Chapter 17).

## 2.2 Why Python needs virtual environments

**What is the problem?**
In Java, dependencies are resolved *per project* from a shared cache: `~/.m2` holds every version of every JAR, and each project's `pom.xml` chooses. The JVM's classpath is constructed per run. Two projects can use different versions of Jackson with no conflict.

Python's import system has no per-project classpath by default. `import requests` searches `sys.path`, which points at *one* `site-packages` directory per interpreter. If you install `requests 2.31` globally, every project on that interpreter sees 2.31. Two projects needing different versions of the same library cannot coexist in one `site-packages`.

**The solution — virtual environments (`venv`):** a directory containing a lightweight copy/symlink of an interpreter plus its own empty `site-packages`. "Activating" it puts its `bin/` first on `PATH`, so `python` and `pip` refer to it. Effectively:

```text
Java:                                    Python:
project A → pom.xml → ~/.m2 (shared)     project A → .venv/ (private site-packages)
project B → pom.xml → ~/.m2 (shared)     project B → .venv/ (private site-packages)
                                         (uv also keeps a global *cache* of downloaded
                                          wheels, hard-linked into each .venv — so the
                                          disk cost is close to Maven's)
```

```text
my-app/
├── .venv/                 ← never commit; recreate from the lockfile
│   ├── bin/python         ← symlink to the real interpreter
│   ├── bin/pip, bin/uvicorn, ...
│   └── lib/python3.14/site-packages/   ← this project's dependencies
├── pyproject.toml
└── uv.lock
```

A venv is disposable: delete `.venv`, run `uv sync`, it is back. Think of it as `target/` plus a private classpath.

## 2.3 The tools, and what each one is for

| Tool | Role | Java analogue | Status in 2026 |
|---|---|---|---|
| `pip` | Installs packages from PyPI into the *current* interpreter/venv | manually downloading JARs | Ships with Python. Still used inside Dockerfiles and CI. No dependency locking. |
| `venv` | Creates a virtual environment (`python -m venv .venv`) | (no analogue) | Standard library. Fine, but uv does it faster and automatically. |
| `pipx` | Installs *CLI tools* (ruff, poetry, httpie) each in its own isolated venv, exposed on PATH | SDKMAN-installed tools, `npx -g` | Useful, but `uv tool install` does the same. |
| **`uv`** | Project manager + package installer + interpreter manager + tool runner, written in Rust, 10–100× faster than pip | **Maven/Gradle + SDKMAN in one binary** | **The recommended default for new projects.** |
| Poetry | Older project manager with its own lockfile; pioneered `pyproject.toml` workflows | Gradle-ish | Very common in existing repos (2019–2024). Read `poetry.lock`, run `poetry install`. Not recommended for new projects now. |
| `pyproject.toml` | The standard project manifest (PEP 621): name, version, dependencies, build system, tool config | `pom.xml` / `build.gradle` | The one file every modern project has. |
| `requirements.txt` | A flat list of pinned packages for `pip install -r` | a generated dependency list, not a manifest | Legacy manifest; still the deployment format for some Docker images and AWS Lambda layers. |
| `setup.py` / `setup.cfg` | Pre-`pyproject` packaging | old `build.xml` | Legacy; you will see it in older libraries. |
| conda | Package + environment manager for the scientific stack, handles non-Python binaries (CUDA, MKL) | n/a | Common in data science teams; not needed for backend/GenAI work. |

**The Java-to-Python mapping to keep in your head:**

```text
Java:    JDK          + Maven/Gradle      + pom.xml/build.gradle + ~/.m2 cache + target/
Python:  interpreter  + uv                + pyproject.toml       + uv cache    + .venv/
                                          + uv.lock (like Gradle's lockfile / Maven's effective POM, pinned)
```

## 2.4 The recommended workflow (used for every project in this guide)

Install uv once (it is a single static binary, no Python required):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh    # macOS / Linux
# Windows: powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
uv --version                                        # uv 0.12.x
```

Create a project:

```bash
uv init customer-api --python 3.14
cd customer-api
```

You get:

```text
customer-api/
├── .python-version      # "3.14" — which interpreter uv should use here
├── pyproject.toml       # manifest
├── README.md
└── main.py              # placeholder entry point
```

Add dependencies (this creates `.venv/` and `uv.lock` automatically on first run):

```bash
uv add "fastapi[standard]" "sqlalchemy>=2.0" "psycopg[binary]" pydantic-settings
uv add --dev pytest pytest-asyncio httpx ruff mypy
```

Run things **without activating anything** — `uv run` executes inside the project's venv, syncing it first if `pyproject.toml` changed:

```bash
uv run fastapi dev app/main.py      # like ./mvnw spring-boot:run
uv run pytest                        # like ./mvnw test
uv run ruff check . && uv run ruff format .
uv run python -c "import fastapi; print(fastapi.__version__)"
```

If you prefer the classic feel, activate the venv once per shell and drop the `uv run` prefix:

```bash
source .venv/bin/activate     # Windows: .venv\Scripts\activate
python -m pytest
deactivate
```

Reproduce the environment on another machine / in CI / in Docker:

```bash
uv sync --frozen         # installs exactly what uv.lock says; fails if lock is stale
```

Everyday commands:

| Task | uv | Maven/Gradle equivalent |
|---|---|---|
| Add dependency | `uv add httpx` | edit pom + `mvn install` |
| Add dev-only dependency | `uv add --dev pytest` | `<scope>test</scope>` |
| Remove | `uv remove httpx` | edit pom |
| Upgrade one | `uv lock --upgrade-package httpx` | `mvn versions:use-latest-versions` |
| Install a global CLI tool | `uv tool install ruff` | SDKMAN / brew |
| Run a tool once without installing | `uvx ruff check .` | `npx`-style |
| Install/choose interpreter | `uv python install 3.14` / `uv python pin 3.14` | SDKMAN `sdk use java 21` |
| Build a wheel | `uv build` | `mvn package` |
| Publish | `uv publish` | `mvn deploy` |
| Show dependency tree | `uv tree` | `mvn dependency:tree` |

## 2.5 Anatomy of `pyproject.toml`

```toml
[project]
name = "customer-api"
version = "0.1.0"
description = "Customer management API"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.141",
    "sqlalchemy>=2.0,<3",
    "psycopg[binary]>=3.2",
    "pydantic-settings>=2.6",
]

[project.scripts]                      # console entry points (like a Spring Boot fat-jar main, or a Maven exec)
customer-api = "customer_api.main:run"

[dependency-groups]                    # PEP 735 — dev/test groups, not installed in production
dev = ["pytest>=9", "pytest-asyncio", "httpx", "ruff", "mypy"]

[build-system]                         # only needed if the project is itself a package (library or src/ layout app)
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]                            # tool configuration lives here too (no separate checkstyle.xml)
line-length = 100
[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]   # pycodestyle, pyflakes, isort, pyupgrade, bugbear

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.mypy]
strict = true
```

Compare with `pom.xml`: `[project]` ≈ `<groupId>/<artifactId>/<version>` + `<dependencies>`; `[dependency-groups]` ≈ test/provided scopes; `[build-system]` ≈ the packaging plugin; `[tool.*]` ≈ plugin `<configuration>`. Notice how little there is: no XML, no plugin versions for compile/test, no lifecycle phases. Python projects are *not compiled*, so the "build" is only relevant if you want to produce an installable package.

**Version specifiers** (PEP 440) — the bits you will read constantly:

| Specifier | Meaning |
|---|---|
| `fastapi>=0.141` | at least 0.141 |
| `sqlalchemy>=2.0,<3` | the common "compatible major" range |
| `pydantic~=2.13` | "compatible release": `>=2.13, ==2.*` |
| `httpx==0.28.1` | exact pin (belongs in lockfiles, rarely in `pyproject`) |
| `fastapi[standard]` | package + optional **extra** dependency set (like a Maven profile that pulls extra deps) |

## 2.6 The essential toolbelt beyond the package manager

- **ruff** — linter *and* formatter, Rust, replaces flake8/isort/black/pyupgrade. The Python `checkstyle + spotless` in one 1-second command. Use it from day one; it will teach you idiomatic Python by complaining.
- **mypy** or **pyright** — static type checkers (Chapter 8). Pyright is what VS Code's Pylance uses; mypy is the classic. Pick one; the guide shows both.
- **pytest** — the test runner (Chapter 18).
- **IDE** — PyCharm (JetBrains; will feel like IntelliJ) or VS Code + Pylance + Ruff extension. PyCharm is the shortest path for an IntelliJ user.
- **pre-commit** — runs ruff/mypy before each commit; optional.
- **Jupyter** — interactive notebooks (Chapter 25); essential for exploring AI libraries, not for backend code.

## 2.7 Docker for a uv project

The Java developer's instinct — multi-stage build, non-root user, cached dependency layer — works exactly the same:

```dockerfile
FROM python:3.14-slim AS base
COPY --from=ghcr.io/astral-sh/uv:0.12 /uv /uvx /bin/
WORKDIR /app
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_NO_DEV=1

# Dependency layer — cached until pyproject/uv.lock change (like copying pom.xml first)
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-install-project

# Application layer
COPY src/ ./src/
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen

ENV PATH="/app/.venv/bin:$PATH"
USER nobody
EXPOSE 8000
CMD ["fastapi", "run", "src/customer_api/main.py", "--port", "8000"]
```

There is no JAR: the image contains the source and a venv. The "artifact" of a Python service is the container image (or, for libraries, a wheel).

## Knowledge check — Chapter 2

1. In Java two projects can depend on different Jackson versions without any special setup. Why can Python not do this without virtual environments?
2. What is the difference between `pyproject.toml` and `uv.lock`? Which one do you edit, and which one do you commit? (Trick: both are committed.)
3. What does `uv run pytest` do that plain `pytest` might not?
4. You see a repo with `poetry.lock` and another with `requirements.txt`. What do you run in each to get a working environment?
5. Why should you never `sudo pip install` anything?

---

# 3. Syntax and Basic Language Features

This chapter covers the Python syntax a Java developer *actually needs*, with the difference explained each time. Read it with a REPL open (`uv run python`, or `uv run ipython` for a nicer one).

## 3.1 Variables and assignment

```python
name = "John"
age = 30
price = 19.99
active = True
nothing = None
```

Already covered in Chapter 1: these are *name bindings*, not typed slots. Extra things to know:

- **No `final`/`const`.** Convention: `MAX_RETRIES = 3` (UPPER_CASE) means "treat as constant". Type checkers honour `Final`: `MAX_RETRIES: Final = 3`. Nothing stops reassignment at runtime.
- **Naming convention is `snake_case`** for variables, functions, methods and modules; `PascalCase` for classes; `_leading_underscore` for "internal" (there is no `private` — Chapter 6). Java's `camelCase` in Python code is the fastest way to be identified as a Java developer.
- **Multiple assignment / unpacking:**

  ```python
  a, b = 1, 2
  a, b = b, a          # swap without a temp
  first, *rest = [1, 2, 3, 4]    # first=1, rest=[2, 3, 4]
  ```
- **Chained comparison:** `if 0 <= x < 10:` is legal and means what it says.
- **Walrus `:=`** assigns inside an expression: `if (m := pattern.match(line)) is not None: use(m)`. Useful, easy to overuse.

## 3.2 Indentation is syntax

Python has no braces. A block is defined by a colon and consistent indentation (4 spaces, never tabs — ruff will format this for you).

```python
def process(order):
    if order.total > 100:
        apply_discount(order)
        notify(order)          # inside the if
    log(order)                 # outside the if — the dedent ends the block
```

**Why?** Python takes the position that since you indent anyway for humans, the compiler should read the same structure — removing an entire class of "brace mismatch" bugs and enforcing readable code. Consequences:

- An empty block needs `pass` (or `...`): `class Marker: pass`.
- Long expressions continue inside `()`, `[]`, `{}` without backslashes; that is why you see `(` around long conditions.
- A stray tab or a mis-indented line is a `IndentationError`. Your editor + ruff make this a non-issue in practice.

## 3.3 Comments and docstrings

```python
# Single-line comment. There is no /* block comment */.

def area(radius: float) -> float:
    """Return the area of a circle.

    Docstrings are the Javadoc equivalent: a string literal as the first
    statement of a module, class or function. Tools (IDEs, pytest, Sphinx,
    LangChain's @tool!) read them at runtime via ``area.__doc__``.
    """
    return 3.14159 * radius ** 2
```

The docstring is a real runtime object — `help(area)` prints it. In GenAI code this matters: LangChain and the Anthropic tool runner **turn the docstring into the tool description the LLM sees**.

## 3.4 Strings

Strings are immutable Unicode sequences, like Java `String`. Differences worth knowing:

```python
s = 'single' + " or double quotes — identical"
multi = """triple quotes
span lines"""

# f-strings (Python 3.6+) — the default way to format. Any expression inside {}.
user, n = "Ada", 3
print(f"{user} has {n} item{'s' if n != 1 else ''}")      # Ada has 3 items
print(f"{n:>5}|{3.14159:.2f}|{1234567:,}")                 # '    3|3.14|1,234,567'
print(f"{user=}")                                          # user='Ada'   (debug form)

# Common operations
"a,b,c".split(",")                 # ['a', 'b', 'c']
", ".join(["a", "b", "c"])         # 'a, b, c'   ← join is a *str* method, not a list method
"  x ".strip()                     # 'x'
"hello".upper().startswith("HE")   # True
"abc"[1:]                          # 'bc'  — slicing (Chapter 4)
len("héllo")                       # 5 — code points, not bytes
"hello" in "say hello"             # True — substring test via `in`
```

- No `StringBuilder`: `"".join(parts)` is the idiomatic way to build a string from many pieces; `+=` in a loop is fine for small cases (CPython optimises it).
- **Bytes vs str** is explicit: `"text".encode("utf-8")` → `b'text'`; `data.decode()` → `str`. HTTP bodies, S3 objects and files opened in `"rb"` mode give you `bytes`. Mixing them is a `TypeError`, which is a feature.
- `str.format()` and `%` formatting exist in old code; write f-strings.
- Python 3.14 adds **t-strings** (`t"..."`, PEP 750), which produce a `Template` object instead of a string so libraries can safely process interpolations (SQL, HTML). You will start seeing them in libraries; you do not need them yet.

## 3.5 Numbers

```python
i = 10 ** 100          # int: arbitrary precision. No overflow, no long, no BigInteger.
f = 0.1 + 0.2          # float: IEEE-754 double, same as Java double (0.30000000000000004)
7 / 2                  # 3.5   — `/` is ALWAYS float division (unlike Java)
7 // 2                 # 3     — floor division (rounds toward -inf: -7 // 2 == -4)
7 % 2                  # 1
2 ** 10                # 1024  — power operator
divmod(7, 2)           # (3, 1)
round(2.675, 2)        # 2.67 (binary float surprise, same as Java)
abs(-3), max(1, 2), min([3, 1, 2]), sum([1, 2, 3])
int("42"), float("3.5"), str(42)      # explicit conversions

from decimal import Decimal            # like BigDecimal — use for money
Decimal("19.99") * 3                   # Decimal('59.97')
```

There is no `int`/`Integer` distinction, no `long`, no `short`, no `byte`; `int` is one type of unlimited size, `float` is a double, `complex` exists. `bool` is a subclass of `int` (`True + True == 2`) — occasionally handy (`sum(x > 0 for x in xs)` counts positives), occasionally surprising.

## 3.6 Booleans, `None`, and truthiness

```python
True, False            # capitalised
None                   # the single "no value" object — its own type NoneType
```

**`None` vs Java `null`.** Both mean "no value", but:

- `None` is a real object; `type(None)` is `NoneType`. Attribute access on it raises `AttributeError: 'NoneType' object has no attribute 'x'` — that message is Python's `NullPointerException`.
- Compare with `is`: `if x is None:` / `if x is not None:`. Never `== None`.
- Functions with no `return` return `None` implicitly (like `void`, except the value exists).
- There is no `Optional<T>` wrapper class. The **type hint** `str | None` (Chapter 8) plays the role of `Optional<String>` for the type checker, and the *value* is simply `None`. There is no `.map()/.orElse()` chain; you write `x if x is not None else default`, or `x or default` when a falsy `x` should also be replaced.

**Truthiness.** Any object can be used in a boolean context. Falsy: `False`, `None`, `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `()` and any object whose `__bool__`/`__len__` says so. Everything else is truthy. Idiomatic Python leans on this:

```python
if not items:            # instead of  if len(items) == 0:
    return
if name:                 # instead of  if name is not None and name != "":
    greet(name)
```

Java has no equivalent (`if (list)` does not compile). Be careful with the one trap: `if count:` treats `0` as "no value", so when `0` is a valid value, write `if count is not None:`.

**Logical operators** are words: `and`, `or`, `not`. They short-circuit and *return one of the operands*, not a bool: `name or "anonymous"` returns `name` if truthy else `"anonymous"` — the Python idiom for a default.

## 3.7 Operators the Java developer should notice

| Java | Python | Note |
|---|---|---|
| `==` (reference), `.equals()` (value) | `is` (identity), `==` (value, calls `__eq__`) | Python's `==` is `.equals()`. Use `is` only for `None`/singletons. |
| `&&` `\|\|` `!` | `and` `or` `not` | |
| `x++`, `x += 1` | `x += 1` only | no `++`/`--` |
| `?:` ternary | `a if cond else b` | condition in the middle |
| `instanceof` | `isinstance(x, Cls)` | accepts a tuple: `isinstance(x, (int, float))` |
| `x != null && x.y` | `x is not None and x.y` | or `x and x.y` when falsy-check is OK |
| `String.valueOf(x)` | `str(x)` | |
| `list.contains(x)`, `map.containsKey(k)` | `x in list`, `k in map` | `in` works on every container and strings |
| `Math.pow(a, b)` | `a ** b` | |
| `a / b` (int division) | `a // b` | `/` is float division |
| `obj.hashCode()` | `hash(obj)` | |
| bitwise `& \| ^ ~ << >>` | same | `\|` is also the type-union operator in hints (`int \| None`) and dict merge (`{**a, **b}` ≡ `a \| b`) |

## 3.8 Conditionals

```python
def classify(n: int) -> str:
    if n < 0:
        return "negative"
    elif n == 0:              # `elif`, not `else if`
        return "zero"
    else:
        return "positive"
```

No parentheses required around the condition; `elif` keyword; no `switch` on old versions — see `match` below. Since the condition is any expression with truthiness, you rarely write `== True` or `!= None`.

## 3.9 Loops

Python has two loops and both are simpler than Java's.

**`for` is always a for-each.** There is no C-style `for (int i = 0; i < n; i++)`. You iterate over *any iterable* (Chapter 14):

```python
for item in items:                      # Java: for (var item : items)
    print(item)

for i in range(5):                      # 0,1,2,3,4    — Java: for (int i = 0; i < 5; i++)
    ...
for i in range(2, 10, 3):               # 2,5,8        — start, stop (exclusive), step
    ...

for i, item in enumerate(items):        # index + value. THE way to get an index. Not range(len(items)).
    print(i, item)
for i, item in enumerate(items, start=1):
    ...

for name, score in zip(names, scores):  # iterate two sequences in lockstep (stops at the shortest)
    ...
for name, score in zip(names, scores, strict=True):   # 3.10+: raise if lengths differ

for key, value in config.items():       # dict entries — Java: for (var e : map.entrySet())
    ...

for line in open("data.txt"):           # files are iterable line by line (use `with`, Chapter 16)
    ...
```

`range` is not a list; it is a lazy, memory-free sequence object (`range(10**9)` is instant). `enumerate` and `zip` are also lazy. This "iterate over a lazy thing" pattern is the heart of Python and Chapter 14 goes deep on it.

**`while`** is the same as Java's, without parentheses. There is no `do…while`; write `while True: ... if done: break`.

**`break`, `continue`** work as in Java. **`else` on loops** exists: the `else` block runs if the loop finished *without* `break` — a niche idiom for "search and not found":

```python
for user in users:
    if user.id == target:
        found = user
        break
else:
    raise LookupError(target)     # only if the loop never hit break
```

**No labelled break.** Extract the nested loop into a function and `return` instead.

## 3.10 `match` — structural pattern matching (3.10+)

**What is it?** Python's `match` looks like Java's `switch` but is much closer to Java 21's *pattern matching for switch* with record deconstruction — and more powerful than both, because it can destructure lists, dicts and objects.

```python
def handle(event: dict) -> str:
    match event:
        case {"type": "order_created", "order_id": oid}:
            return f"new order {oid}"
        case {"type": "order_cancelled", "order_id": oid, "reason": reason}:
            return f"cancelled {oid}: {reason}"
        case {"type": "ping"}:
            return "pong"
        case _:                                  # default; `_` is a wildcard
            raise ValueError(f"unknown event: {event}")
```

Matching objects (a "class pattern") uses the constructor-like syntax and matches on attributes:

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

def describe(p: Point) -> str:
    match p:
        case Point(x=0, y=0):
            return "origin"
        case Point(x=0):
            return "on the y axis"
        case Point(x=x, y=y) if x == y:         # guard
            return "on the diagonal"
        case Point():
            return "somewhere"
```

Sequences: `case [x, y]`, `case [first, *rest]`. Literals and alternatives: `case 200 | 201 | 204:`. Capture a value: `case Point() as p:`.

**Important gotcha:** a bare name in a pattern *captures*, it does not compare. `case MAX:` binds the name `MAX` to anything. To compare against a constant use a dotted name: `case Limits.MAX:`.

**When to use it:** parsing JSON-ish events, dispatching on message types, interpreting ASTs, handling LLM tool-call payloads. **When not to:** a plain `if/elif` on one value is still clearer than `match` with three literal cases. Do not use `match` as a "switch" just because it exists.

## 3.11 A note on the REPL and scripts

Python's interactive shell is a core part of the workflow in a way that `jshell` never became for Java. Try things in `python` / `ipython`, in a Jupyter notebook, or with `python -i script.py` (run then drop into a REPL with the script's variables loaded). Every Python developer debugs by "import it and poke at it". Adopt the habit — it is the fastest way to learn a library like boto3 or LangChain.

## Knowledge check — Chapter 3

1. What does Python's `==` correspond to in Java, and when would you use `is`?
2. Why is `for i in range(len(items))` considered un-Pythonic, and what replaces it?
3. `7 / 2` in Java is `3`; what is it in Python, and how do you get Java's behaviour?
4. Explain truthiness. Give one case where `if x:` is a bug and `if x is not None:` is required.
5. Compare Python's `match` with Java's `switch` — what can `match` do that `switch` cannot? What is the "bare name captures" trap?
6. What is the Python equivalent of a Javadoc comment, and why does it matter for GenAI code specifically?

---

# Part II — The language, properly

---

# 4. Collections

Java has an *interface hierarchy* (`Collection` → `List`/`Set`/`Queue`, `Map`) with many implementations. Python has **four built-in concrete types you use directly** — `list`, `tuple`, `set`, `dict` — plus a few standard-library specialisations. There is no `interface List`; the *abstract behaviours* (Sequence, Mapping, Iterable...) exist in `collections.abc` and are used for type hints and duck typing, not for choosing implementations.

```text
Java                          Python           ...but
------------------------------------------------------------------
ArrayList<T>                  list             heterogeneous by default, slicing, negative indexes
LinkedList / ArrayDeque       collections.deque
HashMap<K,V>                  dict             insertion-ordered (guaranteed since 3.7)
LinkedHashMap                 dict             (same thing — dict already keeps order)
TreeMap                       (none built-in)  sort keys when needed; `sortedcontainers` on PyPI
HashSet<T>                    set              elements must be hashable
TreeSet                       (none built-in)
List.of(...) / unmodifiable   tuple            immutable *sequence*; also the "multiple return values" type
record (as a small value)     tuple / NamedTuple / dataclass (Chapter 9)
Optional<T>                   T | None
Arrays / int[]                list, or array.array / NumPy for numeric
```

## 4.1 The underlying model: references everywhere

Before the types: **every Python collection stores references to objects, never the objects themselves.** `[1, 2, 3]` is an array of three pointers to three `int` objects. A `dict` maps hash → (key ref, value ref). This is like Java collections of `Integer`, never like `int[]`, and it explains both the memory cost and the aliasing behaviour below.

```python
a = [1, 2, 3]
b = a              # b is the SAME list object — like Java `List<Integer> b = a;`
b.append(4)
print(a)           # [1, 2, 3, 4]

c = a.copy()       # shallow copy: new list, same element objects (a[:] and list(a) also work)
c.append(5)
print(a)           # [1, 2, 3, 4]

import copy
nested = [[1, 2], [3]]
shallow = nested.copy()
shallow[0].append(99)
print(nested)      # [[1, 2, 99], [3]]  ← inner lists shared, exactly as Java's `new ArrayList<>(x)`
deep = copy.deepcopy(nested)   # independent
```

Java developers *know* this from `List<List<Integer>>` but often forget it in Python because there is no `new` to remind them. Chapter 33 lists the classic bugs (mutable default arguments, shared class-level lists).

## 4.2 `list` — the workhorse sequence

**What is it?** A mutable, ordered, dynamically-sized array of references. It *is* `ArrayList`: O(1) index and append (amortised), O(n) insert/delete at the front, contiguous memory.

```python
nums = [3, 1, 2]
nums.append(4)              # add at end             ArrayList.add
nums.insert(0, 0)           # insert at index         add(i, x)
nums.extend([5, 6])         # add all                 addAll
nums += [7]                 # same as extend
nums.remove(3)              # remove first occurrence by VALUE (ValueError if absent)
last = nums.pop()           # remove & return last (O(1)); pop(0) is O(n)
nums[0]                     # index                   get(0)
nums[-1]                    # last element            get(size()-1)
len(nums)                   # size()
2 in nums                   # contains — O(n)
nums.index(2)               # indexOf (ValueError if absent)
nums.sort()                 # in place, stable, returns None!
sorted_copy = sorted(nums)  # new list; works on ANY iterable
nums.sort(key=lambda n: -n) # sort by key function      Comparator.comparing(...)
nums.reverse(); reversed(nums)
nums.count(2)
```

**Where the ArrayList comparison breaks down:**

- **Heterogeneous by nature.** `[1, "a", None, [2]]` is fine. With type hints you *say* `list[int]`, but nothing enforces it at runtime.
- **Slicing** creates a *new* list from a range: `nums[1:3]` (index 1 and 2), `nums[:2]`, `nums[2:]`, `nums[::2]` (every second), `nums[::-1]` (reversed copy). Slices never raise on out-of-range bounds. Assignment to a slice mutates in place: `nums[1:3] = [9, 9, 9]`.
- **Negative indexes** count from the end. `nums[-2]` is the second to last.
- `sort()` returns `None` — `x = nums.sort()` is a classic bug. The rule: methods that mutate in place return `None` (`append`, `sort`, `reverse`, `extend`); functions that build a new object return it (`sorted`, `reversed`, `list(...)`).
- **No `remove(int index)` overload.** `del nums[i]` or `nums.pop(i)`.
- **Multiplying** repeats: `[0] * 5` → `[0, 0, 0, 0, 0]`. But `[[]] * 3` gives three references to the *same* inner list — use a comprehension `[[] for _ in range(3)]`.

**Real-world usage.** Lists hold rows from a DB query, messages in an LLM conversation (`messages: list[dict]`), chunks of a document, batches of embeddings. You will pass lists to NumPy (`np.array(list)`) and get lists back from JSON.

## 4.3 `tuple` — immutable sequence, and Python's "multiple values"

**What is it?** An immutable, ordered sequence. Created with commas (the parentheses are optional, the comma is what makes a tuple): `point = (3, 4)`, `single = (3,)`, `empty = ()`.

**Why does Python have it?** Two reasons that are more important than "immutable list":

1. **Multiple return values.** `def min_max(xs): return min(xs), max(xs)` returns a tuple; `lo, hi = min_max(xs)` unpacks it. There is no need for a `Pair` class or an out-parameter. This is used *everywhere*.
2. **Hashability.** A tuple of hashable items is hashable, so it can be a `dict` key or `set` member: `cache[(user_id, date)] = value`. Lists cannot.

```python
point = (3, 4)
x, y = point                     # unpacking
point[0]                         # 3 — indexing and slicing work like list
point[0] = 5                     # TypeError: 'tuple' object does not support item assignment
len(point), 3 in point
coords = {(0, 0): "origin", (1, 0): "east"}   # tuple as dict key
```

**Java comparison.** Tuples are *not* `List.of()`. `List.of()` is an unmodifiable list used for constants; tuples are a lightweight positional record used for "a few values that belong together for a moment". For a *named* structure that outlives one function call, Python developers reach for a `dataclass` or `NamedTuple` (Chapter 9) — the way you would reach for a `record`. The rule of thumb: **tuple for heterogeneous fixed structure (`(name, age)`), list for homogeneous variable-length data (`[user1, user2, ...]`).**

Immutability is shallow: `t = ([1], 2); t[0].append(3)` works because the list inside is still mutable.

## 4.4 `set` — hashed membership

**What is it?** An unordered collection of unique, hashable objects. Backed by a hash table, O(1) membership.

```python
tags = {"python", "java", "python"}     # {'python', 'java'} — duplicates collapse
empty = set()                           # NOT {} — that is an empty dict
tags.add("go"); tags.discard("cobol")   # discard: no error if absent; remove: KeyError if absent
"java" in tags                          # O(1)
a = {1, 2, 3}; b = {3, 4}
a | b, a & b, a - b, a ^ b              # union, intersection, difference, symmetric difference
a <= b                                  # subset test
frozenset(a)                            # immutable, hashable set (can be a dict key / set member)
```

Java's `HashSet` is the right mental model. The differences: set algebra is built in with operators (Java needs `retainAll`/`addAll` in a copy), and the most common use is **deduplicate while keeping order** — which a set alone does *not* do:

```python
unique_in_order = list(dict.fromkeys(items))   # dict preserves insertion order, keys are unique
```

**Hashability** is the rule that decides what can go in a set or be a dict key: an object needs `__hash__` and `__eq__` consistent with each other (same contract as Java `hashCode`/`equals`). Built-ins: `int`, `str`, `float`, `bool`, `tuple` (of hashables), `frozenset`, `bytes`, `None` are hashable; `list`, `dict`, `set` are not (mutable). Your own classes are hashable by identity by default, like Java `Object`; `@dataclass(frozen=True)` gives value-based hashing (Chapter 9).

## 4.5 `dict` — the most important data structure in Python

**What is it?** A hash map from hashable keys to values. Since Python 3.7, **insertion order is guaranteed** (the implementation switched to a compact ordered layout in 3.6, and the language spec locked it in 3.7). So a Python `dict` is Java's `LinkedHashMap`, and there is no separate unordered `HashMap`.

**Why is it so central?** Because Python *itself* runs on dicts: module namespaces are dicts, object attributes live in `__dict__`, keyword arguments arrive as a dict, JSON objects become dicts. A Python developer's first model for "some structured data" is a dict; the class comes later, if at all. This is the opposite of the Java reflex (DTO first).

```python
user = {"name": "Ada", "age": 36}         # literal
user = dict(name="Ada", age=36)           # keyword form (string keys only)

user["name"]                              # 'Ada' — KeyError if missing (NOT null!)
user.get("email")                         # None if missing
user.get("email", "n/a")                  # default
user["email"] = "ada@x.io"                # insert/overwrite
del user["age"]
"name" in user                            # key membership (like containsKey)
len(user)

user.keys(), user.values(), user.items()  # live *views*, iterable, set-like for keys
for key, value in user.items():
    ...

user.setdefault("tags", []).append("vip")     # get-or-insert-default, then use it
user.pop("age", None)                          # remove with default
merged = {**defaults, **overrides}             # merge (right wins) — or defaults | overrides (3.9+)
user.update(other_dict)

from collections import defaultdict, Counter
by_country = defaultdict(list)                 # missing key → list() automatically
for u in users:
    by_country[u.country].append(u)            # Java: computeIfAbsent(k, k -> new ArrayList<>()).add(u)
Counter(words).most_common(3)                  # word frequencies in one line
```

**Where the HashMap comparison breaks down:**

- `d[key]` on a missing key **raises `KeyError`** instead of returning `null`. Use `.get()` when absence is normal; use `[]` when absence is a bug (fail fast).
- Keys can be any hashable, including tuples: `distances[("A", "B")]`.
- Ordered by insertion, always.
- `dict` is often used *instead of a class* for ad-hoc structured data — especially for JSON payloads and LLM messages. When the shape matters, upgrade to a `TypedDict` (typing only) or a Pydantic model (validation) — Chapters 8–9.
- **Dictionary views** (`keys()`, `items()`) are live: they reflect later changes, and mutating a dict while iterating it raises `RuntimeError`. Iterate over `list(d.keys())` if you must mutate.

**Real-world usage.** Every JSON body from an API is a `dict`; every boto3 response is a (nested) `dict`; an LLM chat message is `{"role": "user", "content": "..."}`; FastAPI turns a Pydantic model into a dict for the response; config is a dict; pandas is built on dict-of-columns. Learning to *read nested dicts fluently* — `response["output"]["message"]["content"][0]["text"]` — is a core GenAI-in-Python skill.

## 4.6 Slicing, unpacking, and membership — the sequence toolkit

**Slicing** works on `list`, `tuple`, `str`, `bytes`, `range`, NumPy arrays and pandas objects with the same `[start:stop:step]` syntax. Stop is exclusive; omitted means "from the beginning" / "to the end"; negatives count from the end.

```python
s = "abcdefgh"
s[2:5]      # 'cde'
s[:3]       # 'abc'
s[-3:]      # 'fgh'
s[::2]      # 'aceg'
s[::-1]     # 'hgfedcba'
chunks = [items[i:i + 100] for i in range(0, len(items), 100)]   # batching — used constantly for embeddings
```

**Unpacking** applies to any iterable and nests:

```python
(a, b), c = (1, 2), 3
head, *body, tail = range(10)          # head=0, body=[1..8], tail=9
for (key, (x, y)) in points.items(): ...
first, second = "ab"                   # strings unpack too

def connect(host, port): ...
args = ("localhost", 5432)
connect(*args)                          # * unpacks a sequence into positional args
opts = {"host": "localhost", "port": 5432}
connect(**opts)                         # ** unpacks a mapping into keyword args
```

**Membership `in`** is one operator for every container: O(n) on list/tuple/str (substring), O(1) on set/dict (keys). If you find yourself writing `x in some_list` inside a loop over thousands of items, convert the list to a set first — the single most common Python performance fix.

## 4.7 Comprehensions

**What is it?** A concise expression that builds a list, set, dict, or generator from an iterable with optional filtering. It is Python's replacement for the *simple* uses of Java Streams.

```python
squares = [n * n for n in range(10)]                        # list
evens   = [n for n in nums if n % 2 == 0]                   # with filter
pairs   = [(x, y) for x in xs for y in ys if x != y]        # nested loops (left to right)
lengths = {word: len(word) for word in words}               # dict
seen    = {user.email.lower() for user in users}            # set
lazy    = (n * n for n in range(10**9))                     # generator — no memory (Chapter 14)
```

**Java comparison:**

```java
List<Integer> evens = nums.stream().filter(n -> n % 2 == 0).map(n -> n * n).toList();
```

```python
evens = [n * n for n in nums if n % 2 == 0]
```

Read a comprehension as: `[<what to produce> for <item> in <source> if <condition>]` — the "for" is the source, the front is the map, the "if" is the filter. When it gets longer than one comfortable line or has more than two `for`s, **use a for-loop**. Pythonic means readable, not compressed (Chapter 13).

**Real-world usage:**

```python
# boto3 → list of bucket names
names = [b["Name"] for b in s3.list_buckets()["Buckets"]]
# LLM messages → only user turns
user_turns = [m["content"] for m in messages if m["role"] == "user"]
# text chunks → embeddings request
payload = [{"text": chunk} for chunk in chunks]
```

## 4.8 Nested collections and the "JSON shape" mindset

Real data is nested: lists of dicts of lists. Practise reading structures by their shape:

```python
response = {
    "output": {
        "message": {
            "role": "assistant",
            "content": [{"text": "Hello!"}],
        }
    },
    "usage": {"inputTokens": 12, "outputTokens": 3},
}
text = response["output"]["message"]["content"][0]["text"]        # Bedrock Converse response
tokens = response["usage"]["inputTokens"] + response["usage"]["outputTokens"]
```

For robust code, stop indexing blindly at the boundary and validate into a Pydantic model (Chapter 9). Inside your own code, keep passing typed objects; use raw dicts at the edges where JSON enters and leaves.

## 4.9 Other standard-library collections worth knowing

| Type | Use | Java analogue |
|---|---|---|
| `collections.deque` | O(1) append/pop at both ends; `maxlen` ring buffer | `ArrayDeque` |
| `collections.Counter` | multiset / frequency counting | `Map<T,Integer>` + `merge` |
| `collections.defaultdict` | dict with factory for missing keys | `computeIfAbsent` everywhere |
| `collections.OrderedDict` | mostly obsolete (dict is ordered); has `move_to_end` for LRU caches | `LinkedHashMap(accessOrder=true)` |
| `collections.namedtuple` / `typing.NamedTuple` | tuple with named fields | tiny record |
| `heapq` | functions over a list as a min-heap | `PriorityQueue` |
| `bisect` | binary search / insert into sorted list | `Collections.binarySearch` |
| `array.array` | compact typed numeric array | `int[]` (but you'll use NumPy) |
| `queue.Queue` | thread-safe FIFO | `BlockingQueue` |

## Knowledge check — Chapter 4

1. `dict` ≈ `HashMap` is the usual claim. Name two ways that comparison is *wrong* or misleading.
2. Why can a tuple be a dictionary key and a list cannot? What Java contract is the same idea?
3. `b = a; b.append(1)` changes `a`. Explain in terms of references. How do you make an independent copy of a nested list?
4. What does `nums.sort()` return, and what is the general rule that predicts it?
5. Rewrite `nums.stream().filter(n -> n > 0).map(Math::sqrt).toList()` as a comprehension. When would you *not* use a comprehension?
6. You need "all unique values, in first-seen order". Why is `list(set(items))` wrong, and what is right?

---

# 5. Functions

In Java, a method is a *member* of a class; there are no free-standing functions, and a method is not a value (you need a `Function<>` or method reference to pass one around). In Python, **a function is an ordinary object** created by the `def` statement, living in a namespace like any variable, passable, storable, decoratable. Most of the module-level code you will read in Python is functions, not classes. This chapter is about getting comfortable with that.

## 5.1 Defining and calling

```python
def greet(name: str, greeting: str = "Hello") -> str:
    """Return a greeting for name."""
    return f"{greeting}, {name}!"

greet("Ada")                       # 'Hello, Ada!'
greet("Ada", "Hi")                 # positional
greet(name="Ada", greeting="Hi")   # keyword arguments — any order
greet("Ada", greeting="Hi")        # mix: positionals first, then keywords
```

Notes for the Java developer:

- `def` is an executable statement: when it runs, a function object is created and bound to the name. Functions can therefore be defined inside `if`s, inside other functions, at runtime.
- **No overloading.** Two `def greet` in the same scope: the second replaces the first. Python replaces overloading with default arguments, `*args`, and `isinstance` checks — or `functools.singledispatch` for genuine type-based dispatch.
- **Return values** are optional; missing `return` yields `None`. Multiple values → a tuple (Section 4.3).
- Type hints (`name: str`, `-> str`) are documentation for tools; nothing enforces them (Chapter 8).

## 5.2 Parameters: positional, keyword, defaults, `*args`, `**kwargs`

Python's parameter system is richer than Java's and you *must* read it fluently because every library API uses it.

```python
def request(method, url, *, timeout=10.0, headers=None, **kwargs):
    ...
```

| Piece | Meaning |
|---|---|
| `method, url` | required positional (or keyword) parameters |
| `*` | everything after it is **keyword-only** — `request("GET", url, 5)` is an error; you must write `timeout=5`. Used to make call sites readable and APIs evolvable. |
| `timeout=10.0` | default value |
| `headers=None` | default for a mutable — see the trap below |
| `**kwargs` | collects any extra keyword arguments into a dict (`{"verify": False}`) — used to pass options through to another layer |
| `*args` (not shown) | collects extra positional arguments into a tuple |
| `/` (not shown) | parameters before it are **positional-only**: `def f(a, b, /)` |

Reading a signature in the docs: `httpx.get(url, *, params=None, headers=None, timeout=...)` means "url positional, everything else must be named". That `*` is why Python call sites look like `client.converse(modelId=..., messages=...)` — long keyword calls are the norm and are considered *good* style.

**Default arguments are evaluated once, at `def` time.** This is the most famous Python trap:

```python
def add_tag(tag, tags=[]):        # BUG: the same list object is reused across calls
    tags.append(tag)
    return tags

add_tag("a")   # ['a']
add_tag("b")   # ['a', 'b']  ← surprise

def add_tag(tag, tags=None):      # idiom
    if tags is None:
        tags = []
    tags.append(tag)
    return tags
```

Immutable defaults (`0`, `""`, `None`, tuples) are safe. Mutable defaults (`[]`, `{}`, `set()`, objects) are not.

**`*args` / `**kwargs`** appear in three roles:

```python
def log_call(*args, **kwargs):                  # 1. accept anything (decorators, wrappers)
    print(args, kwargs)

def create_engine(url, **engine_options):       # 2. pass-through options to a lower layer
    return Engine(url, **engine_options)

print(*[1, 2, 3], sep=", ")                     # 3. at the call site: unpack a sequence/mapping (Section 4.6)
```

## 5.3 Functions are objects — first-class functions

```python
def square(x): return x * x

f = square              # bind another name — no method reference syntax needed
f(4)                    # 16
ops = {"sq": square, "neg": lambda x: -x}
ops["sq"](3)            # 9
list(map(square, [1, 2, 3]))
square.__name__         # 'square'
square.__doc__          # docstring or None
square.calls = 0        # you can even set attributes on a function (rarely wise)
```

**Java comparison.** In Java 8+ you *can* pass behaviour, but only through a functional interface (`Function<Integer,Integer> f = Main::square`), and the function still lives on a class. In Python the function *is* the value, with no wrapper type, no interface, no class. Higher-order functions (functions taking/returning functions) are therefore ordinary: `sorted(users, key=lambda u: u.age)`, `map`, `filter`, `functools.partial`, callbacks in boto3, `@app.get(...)` in FastAPI — all just pass function objects around.

## 5.4 `lambda`

An anonymous single-expression function: `lambda x, y=0: x + y`. Same idea as `(x, y) -> x + y`, with a hard rule: **one expression, no statements, no annotations.** Use it for tiny throwaway callbacks (`key=lambda u: u.created_at`); the moment it grows, write a `def` with a name. Python style leans away from lambdas more than Java style leans away from lambdas — a named function with a docstring is preferred as soon as the logic means something. The `operator` module has ready-made ones: `key=operator.attrgetter("age")`, `key=operator.itemgetter("name")`.

## 5.5 Scope and closures

Python resolves names with the **LEGB** rule: Local → Enclosing function → Global (module) → Builtins. There is no block scope: a variable assigned inside an `if` or `for` is visible after it, within the same function.

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count          # needed to *assign* to an enclosing variable
        count += 1
        return count
    return increment

c = make_counter()
c(); c()                        # 2  — `count` lives on in the closure
```

**Closures** capture *variables*, not values (late binding) — the same caveat as Java's "effectively final", but Python lets you mutate through `nonlocal`. The classic surprise:

```python
fns = [lambda: i for i in range(3)]
[f() for f in fns]              # [2, 2, 2] — all see the final i
fns = [lambda i=i: i for i in range(3)]   # capture by default-arg → [0, 1, 2]
```

`global x` inside a function lets you assign to a module-level variable. Doing so is a smell in application code (Chapter 33 — global state).

**Real-world usage.** Closures are how decorators work (Chapter 15), how FastAPI dependencies capture configuration, how you make a "configured function" without a class:

```python
def make_bedrock_caller(client, model_id):
    def call(prompt: str) -> str:
        r = client.converse(modelId=model_id, messages=[{"role": "user", "content": [{"text": prompt}]}])
        return r["output"]["message"]["content"][0]["text"]
    return call

ask = make_bedrock_caller(boto3.client("bedrock-runtime"), "amazon.nova-pro-v1:0")
ask("Summarise this")
```

In Java that would be a class with a constructor and one method. In Python it *can* be a class, but a closure is often the more natural choice when there is one behaviour and some captured state. `functools.partial(call, client, model_id)` is the even shorter version when no logic is added.

## 5.6 Decorators (preview)

`@something` above a `def` means "replace this function with `something(function)`". It is *just* a higher-order function applied at definition time — so all the machinery you just learned (functions as values, closures, `*args/**kwargs`) is what decorators are made of. Chapter 15 builds them from scratch; for now, recognise them:

```python
@app.get("/users/{user_id}")           # FastAPI: register route  (≈ @GetMapping)
@functools.lru_cache(maxsize=128)      # memoise                  (≈ @Cacheable)
@dataclass                             # generate __init__, __eq__, __repr__ (≈ record/Lombok)
@pytest.fixture                        # pytest DI                (≈ @BeforeEach + parameter injection)
@tool                                  # LangChain: expose function to an LLM
```

## 5.7 Useful built-in and `functools` helpers

```python
from functools import partial, lru_cache, cache, wraps, reduce

double = partial(operator.mul, 2)           # fix some arguments
@cache                                       # unbounded memo (3.9+); lru_cache(maxsize=...) for bounded
def fib(n): return n if n < 2 else fib(n-1) + fib(n-2)

callable(obj)                                # does it have __call__?
any(u.active for u in users); all(...)       # short-circuit over a generator
sorted(users, key=lambda u: (u.country, -u.age))   # multi-key sort via tuple
min(users, key=lambda u: u.age)
```

## 5.8 Java methods vs Python functions — the mental shift

| Java | Python |
|---|---|
| Behaviour must live in a class | Module-level functions are the default unit of behaviour |
| `static` utility methods in a `Utils` class | plain functions in a module (`from myapp.utils import slugify`) |
| Overloading | defaults, `*args`, keyword-only, `singledispatch` |
| Method references / lambdas via functional interfaces | functions are values; `lambda` for one-liners |
| Strategy pattern = interface + N classes | often just pass a function |
| `void` | implicit `None` |
| Multiple returns → a small class | a tuple, unpacked at the call site |
| `@Override`, `final`, `private` | conventions and docs; no enforcement |

The single most useful adjustment: **when you are about to write a class with one method, write a function.** When you are about to write a `*Utils` class, write a module.

## Knowledge check — Chapter 5

1. Why does `def f(items=[])` misbehave, and what does the fix look like? What does this reveal about *when* default values are evaluated?
2. What does a bare `*` in a parameter list do, and why do Python libraries use it so much?
3. In what sense is a Python function "an object"? Give two things you can do with a function that you cannot do with a Java method.
4. Explain the `[lambda: i for i in range(3)]` result. Which Java rule prevents the same bug?
5. You are about to create `class PriceCalculator` with a constructor and one `calculate()` method. What would a Python developer write instead, and when would the class still be the right call?

---

# 6. Python's Object Model

This is the chapter where Java knowledge helps the most *and* misleads the most. Python has classes, inheritance, polymorphism, `super()`, and `@staticmethod` — so it looks like Java. But the *mechanism* is completely different: a Python class is a **runtime object built by executing a block of code**, an instance is essentially **a dictionary plus a pointer to its class**, and every "feature" (properties, private fields, operator overloading, `len()`) is implemented by **looking up specially-named attributes at runtime**. Once you see that, `self`, "everything is an object", and dunder methods stop being strange.

## 6.1 Everything is an object

```python
>>> type(42), type("x"), type([]), type(None)
(<class 'int'>, <class 'str'>, <class 'list'>, <class 'NoneType'>)
>>> def f(): pass
>>> type(f)
<class 'function'>
>>> type(int)               # a class is an object too — of type `type`
<class 'type'>
>>> type(type)
<class 'type'>
>>> import math; type(math)
<class 'module'>
>>> (42).bit_length(), "x".upper(), [].append
```

There are no primitives, no static-only holders, no "class vs object" barrier. `int` is an object you can pass around (`converters = {"age": int}`), a module is an object with attributes, a class is an instance of `type`. In Java, `Integer.class` is a *description* of a class; in Python, `int` *is* the class, callable to create instances.

**Why?** Uniformity. One rule — "attributes are looked up on objects by name" — covers functions, classes, modules, instances. That is what makes Python introspectable and metaprogrammable with little machinery (Pydantic, dataclasses, pytest and FastAPI all rely on it).

## 6.2 A class, line by line

```python
class User:

    def __init__(self, name):
        self.name = name

    def say_hello(self):
        return f"Hello {self.name}"
```

**What happens when this runs:**

1. `class User:` starts executing the *class body* as a block of code, in a fresh namespace (a dict).
2. `def __init__` and `def say_hello` create two **function objects** and put them into that namespace under the keys `"__init__"` and `"say_hello"`.
3. When the body finishes, Python calls `type("User", (object,), namespace)` — the metaclass builds a **class object** with those attributes and binds the name `User` in the module.

So a class is *created at runtime by running code*. You can put `if` statements, loops or `print()` in a class body. The methods are plain functions stored in a dict: `User.__dict__["say_hello"]` is the function.

**What happens on `u = User("Ada")`:**

1. `User(...)` calls the class object. `type.__call__` runs `User.__new__(User)` — allocates an empty instance with an empty `__dict__` and a `__class__` pointer to `User`.
2. Then it calls `User.__init__(u, "Ada")` — **the instance is passed as the first argument**. `__init__` is an *initialiser*, not a constructor: the object already exists when it runs. It returns `None`.
3. `self.name = name` puts `"name": "Ada"` into `u.__dict__`.

```python
>>> u = User("Ada")
>>> u.__dict__
{'name': 'Ada'}
>>> u.__class__
<class '__main__.User'>
```

**What happens on `u.say_hello()`:**

1. Attribute lookup: is `say_hello` in `u.__dict__`? No. In `type(u).__dict__`? Yes — it's a function.
2. Functions found on a class via an instance are turned into a **bound method**: an object that remembers `u` and, when called, calls the function with `u` prepended. `u.say_hello()` ≡ `User.say_hello(u)`.

```python
>>> User.say_hello(u)      # explicit form — same thing
'Hello Ada'
>>> u.say_hello            # a bound method object
<bound method User.say_hello of <__main__.User object at 0x...>>
```

## 6.3 Why `self` and not `this`

Because there is no magic. In Java, `this` is an implicit variable the compiler injects into every instance method. In Python, methods are ordinary functions; the instance has to arrive *somehow*, so it arrives as the first parameter, explicitly. Naming it `self` is convention only (`def say_hello(me)` works). The explicitness is deliberate ("explicit is better than implicit", `import this`):

- `self.name` vs `name` — always clear whether you mean an attribute or a local. Java's `this.name = name` in constructors is the same disambiguation, made mandatory everywhere.
- A method is *just a function whose first parameter is the instance* — which is why you can store functions in dicts, pass `User.say_hello` around, and why decorators can wrap methods exactly like functions.

Forgetting `self` produces `TypeError: say_hello() takes 0 positional arguments but 1 was given` — the "1" is the instance being passed in. You will see this error once; then you will never forget it.

## 6.4 Attributes: instance vs class variables

```python
class Counter:
    instances = 0                   # class variable — shared, lives in Counter.__dict__

    def __init__(self):
        self.count = 0              # instance variable — lives in self.__dict__
        Counter.instances += 1      # explicit class access
```

Lookup on `obj.x` is: instance `__dict__` → class `__dict__` → parent classes (MRO) → `AttributeError`. Assignment `obj.x = ...` **always writes to the instance dict** — it never modifies the class. That asymmetry creates the classic trap:

```python
class Team:
    members = []                    # ONE list shared by every Team instance
    def add(self, m):
        self.members.append(m)      # reads the class list, mutates it — shared across all teams!
```

Rule: declare **mutable per-object state in `__init__`**, never at class level. Class-level assignments are for real constants and for dataclass field declarations (which the decorator moves into `__init__` for you).

There is no `static` keyword for fields; a class variable *is* the static field. Accessing it as `self.instances` works (falls through to the class) but `Counter.instances` is clearer.

**Attributes can be added dynamically**: `u.email = "x"` works on any normal instance, even if `__init__` never mentioned `email`. This is why Python has no "field declaration" — the instance dict grows on demand. (Advanced: `__slots__ = ("name", "email")` on a class replaces the dict with fixed slots — smaller, faster, and no dynamic attributes. Libraries use it; you rarely need it.)

## 6.5 Methods: instance, class, static

```python
class Temperature:
    def __init__(self, celsius: float):
        self.celsius = celsius

    def to_fahrenheit(self) -> float:            # instance method — gets the instance
        return self.celsius * 9 / 5 + 32

    @classmethod
    def from_fahrenheit(cls, f: float) -> "Temperature":   # gets the CLASS — alternative constructor
        return cls((f - 32) * 5 / 9)

    @staticmethod
    def is_valid(celsius: float) -> bool:        # gets nothing — a function namespaced in the class
        return celsius >= -273.15
```

- `@classmethod` is the idiom for **named/alternative constructors** (`Temperature.from_fahrenheit(212)`, `dict.fromkeys`, `Model.model_validate(...)`) — where Java uses static factory methods. `cls` is the actual class, so subclasses get subclass instances.
- `@staticmethod` is rarer than Java `static`: if the function does not need the class, Python developers usually just put it at module level.

## 6.6 Encapsulation: there is no `private`

Python has **no access modifiers**. Everything is public. Conventions carry the meaning:

| Spelling | Meaning | Enforcement |
|---|---|---|
| `name` | public API | — |
| `_name` | "internal; don't touch from outside" (like package-private/protected in spirit) | none; linters and `from x import *` skip it |
| `__name` (two leading underscores, no trailing) | name-mangled to `_ClassName__name` to avoid clashes in subclasses | mild obfuscation, not privacy |
| `__name__` (dunder) | reserved for Python's protocols (`__init__`, `__len__`) | don't invent your own |

**Why?** "We're all consenting adults." Python trusts the reader and prioritises testability and debuggability over sealing. In practice `_single_underscore` is respected by teams and tools, and that is enough.

**No getters/setters.** Public attributes are the default. If you later need logic on read/write, you add a **property** *without changing callers*:

```python
class Account:
    def __init__(self, balance: float):
        self._balance = balance

    @property
    def balance(self) -> float:               # acc.balance   (read)
        return self._balance

    @balance.setter
    def balance(self, value: float) -> None:  # acc.balance = 10   (write, validated)
        if value < 0:
            raise ValueError("balance cannot be negative")
        self._balance = value
```

This is the key reason `getX()/setX()` is un-Pythonic: Java needs getters *up front* because switching from a field to a method breaks callers; Python's `@property` makes that switch invisible, so you start with plain attributes and add properties only when there is real logic. (Chapter 33 has the full argument.)

## 6.7 Dunder methods — how Python "protocols" work

Python has no operator interfaces like `Comparable`; instead, built-in operations look up **specially-named methods** on the object's class. Implement them and your object plugs into the language:

| You write | Python calls | Java analogue |
|---|---|---|
| `str(obj)`, `print(obj)`, f-string | `__str__` | `toString()` |
| REPL display, `repr(obj)`, in collections | `__repr__` (aim for "unambiguous, ideally re-creatable") | `toString()` for debugging |
| `a == b` | `__eq__` (then `__hash__` must be consistent) | `equals` / `hashCode` |
| `a < b`, `sorted(objs)` | `__lt__` (or `functools.total_ordering`) | `Comparable` |
| `len(obj)` | `__len__` | `size()` |
| `obj[key]`, `obj[key] = v` | `__getitem__`, `__setitem__` | `get`/`put` |
| `for x in obj` | `__iter__` | `Iterable` |
| `x in obj` | `__contains__` | `contains` |
| `obj()` | `__call__` | functional interface |
| `with obj:` | `__enter__`/`__exit__` | `AutoCloseable` |
| `a + b`, `a * b` | `__add__`, `__mul__` | — (no operator overloading in Java) |
| `bool(obj)`, `if obj:` | `__bool__` / `__len__` | — |
| `obj.missing` | `__getattr__` (fallback) | — (proxies via reflection) |

```python
class Money:
    def __init__(self, amount: int, currency: str):
        self.amount, self.currency = amount, currency

    def __repr__(self) -> str:
        return f"Money({self.amount!r}, {self.currency!r})"

    def __eq__(self, other) -> bool:
        if not isinstance(other, Money):
            return NotImplemented          # let Python try the reflected operation
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __hash__(self) -> int:
        return hash((self.amount, self.currency))

    def __add__(self, other: "Money") -> "Money":
        if other.currency != self.currency:
            raise ValueError("currency mismatch")
        return Money(self.amount + other.amount, self.currency)

Money(5, "EUR") + Money(7, "EUR")     # Money(12, 'EUR')
```

`@dataclass` (Chapter 9) writes `__init__`, `__repr__`, `__eq__` (and optionally `__hash__`, ordering) for you. In real code you mostly *consume* dunder methods (NumPy's `+`, pandas' `[]`, SQLAlchemy's `==` producing SQL, Pydantic's `__eq__`) and write only `__repr__`/`__eq__` occasionally.

## 6.8 Inheritance

```python
class Animal:
    def __init__(self, name: str):
        self.name = name
    def speak(self) -> str:
        raise NotImplementedError

class Dog(Animal):
    def __init__(self, name: str, breed: str):
        super().__init__(name)             # call parent initialiser — NOT automatic
        self.breed = breed
    def speak(self) -> str:                 # override: just define it; no @Override
        return "Woof"

isinstance(Dog("Rex", "lab"), Animal)      # True
issubclass(Dog, Animal)                    # True
Dog.__mro__                                # (Dog, Animal, object) — method resolution order
```

Differences from Java:

- `super().__init__()` is **not** called implicitly. Forgetting it is a real bug.
- No `@Override` annotation — misspelling a method name silently creates a new one. `typing.override` (3.12+) exists for the type checker: `@override def speak(self)`.
- **Multiple inheritance** is allowed: `class A(B, C)`. The MRO (C3 linearisation) decides the lookup order left-to-right. The idiomatic use is **mixins** — small classes adding one capability (`class JSONMixin: def to_json(self)...`). Diamond problems are resolved by the MRO; `super()` follows the MRO, not "the parent". You will meet mixins in Django, SQLAlchemy and pytest plugins; you rarely need to write them.
- No `final` classes/methods (`typing.final` for the checker), no `abstract` keyword (Chapter 7's `ABC`), no interfaces (Chapter 7's `Protocol`).
- Overriding is by name only; there is no signature check at runtime.

**Python leans on inheritance far less than Java.** Framework base classes exist (`BaseModel`, `DeclarativeBase`, `unittest.TestCase`), but application code mostly uses composition, functions, and duck typing. A 5-level class hierarchy in a Python service is a Java-accent smell.

## 6.9 Composition and polymorphism, the Python way

**Polymorphism** in Python is *duck typing*: any objects with a `.speak()` method are interchangeable, related or not.

```python
class Dog:
    def speak(self): return "Woof"
class Robot:
    def speak(self): return "Beep"           # no shared base class

for thing in [Dog(), Robot()]:
    print(thing.speak())                     # works — polymorphism without inheritance
```

When you want the *type checker* to verify "has `.speak()`", you add a `Protocol` (Chapter 7). When you want a runtime guarantee that subclasses implement it, you use an `ABC`. Neither is required for the code to run.

**Composition** looks like Java's, minus the interface declarations and the constructor injection ceremony:

```python
class OrderService:
    def __init__(self, repo, payment, notifier):     # dependencies passed in; any objects with the right methods
        self.repo = repo
        self.payment = payment
        self.notifier = notifier

    def place(self, order):
        self.payment.charge(order.total)
        self.repo.save(order)
        self.notifier.send(order.customer_email, "Thanks!")
```

There is no Spring container needed to make this testable: pass fakes in tests. FastAPI's `Depends` (Chapter 20) does wiring at the request level when you want it.

## 6.10 Introspection — why Python frameworks feel magical

Because classes and instances are dicts, frameworks read them:

```python
vars(u)                     # instance attributes as a dict
dir(u)                      # all attribute names incl. inherited
getattr(u, "name"), setattr(u, "name", "B"), hasattr(u, "email")
User.__annotations__        # {'name': 'str'} — how dataclasses and Pydantic discover fields
inspect.signature(func)     # how FastAPI reads a route's parameters to build DI + OpenAPI
```

FastAPI reads your function signature and type hints at import time to build validation and docs. Pydantic reads class annotations to build validators. pytest reads fixture names from test function parameters. This is reflection, but cheap and idiomatic, and it is why Python frameworks need no XML, no annotation processors, and little boilerplate.

## 6.11 Where the Java class model differs, in one table

| Concept | Java | Python |
|---|---|---|
| Class | compile-time type | runtime object created by executing a block |
| Instance | fixed layout from field declarations | `__dict__` grown at runtime (or `__slots__`) |
| Constructor | `new` + constructor | `__new__` (allocate) + `__init__` (initialise) |
| `this` | implicit | explicit `self` first parameter |
| Fields | declared, typed, access-controlled | any attribute, convention-based privacy |
| Getters/setters | needed for evolvability | plain attributes; `@property` later |
| Static | `static` fields/methods | class variables, `@classmethod`, `@staticmethod`, or module-level functions |
| Interfaces | `interface` + `implements` | duck typing, `Protocol`, `ABC` |
| Overloading | by signature | not available; defaults/`*args` |
| `equals/hashCode/toString/compareTo` | override | `__eq__/__hash__/__repr__/__lt__` |
| Operator overloading | no | yes (dunders) |
| Reflection | `java.lang.reflect`, heavyweight | `getattr`, `__dict__`, `inspect`, cheap and common |

## Knowledge check — Chapter 6

1. Describe what happens, step by step, when `class User:` and then `User("Ada")` and then `u.say_hello()` execute.
2. Why must `self` be written explicitly? What does that tell you about what a method really is?
3. `class Team: members = []` — why is this a bug for per-team members and what is the fix?
4. A Java colleague insists on `get_name()` / `set_name()`. Explain why Python starts with public attributes and what `@property` changes about the "evolvability" argument.
5. What is a dunder method? Give three Java concepts that Python implements through dunder methods.
6. Explain how FastAPI can build request validation from a function signature without annotation processors.

---

# 7. Interfaces, Abstract Classes and Protocols

In Java, `interface` is the central abstraction tool: you define a contract, classes `implements` it, the compiler enforces it, and DI wires implementations. Python solves "contract between components" in **four different ways**, and choosing between them is a skill. Here they are from lightest to heaviest.

## 7.1 Level 0 — Duck typing (the default)

```java
interface PaymentService {
    void pay(long amountCents);
}
```

In Python, the *default* answer to "how do I define the contract" is: **you don't; you document it and rely on the shape.**

```python
class StripePayment:
    def pay(self, amount_cents: int) -> None: ...

class FakePayment:               # test double — no interface needed
    def __init__(self): self.calls = []
    def pay(self, amount_cents: int) -> None: self.calls.append(amount_cents)

def checkout(payment, order):     # accepts anything with .pay()
    payment.pay(order.total_cents)
```

**Why is this acceptable?** Because the contract *is* enforced — at call time, by the `AttributeError` you get if `.pay` is missing — and tests catch that immediately. For small codebases and for test doubles, the absence of a declared interface is a productivity win. **When it is not enough:** large codebases where readers need the contract spelled out, and any place you want the type checker to help.

## 7.2 Level 1 — `Protocol`: structural typing for the type checker

**What is it?** A class that declares *only method/attribute signatures*. Any object with matching members **satisfies** it — no `implements`, no inheritance. This is duck typing made checkable: Java's interface semantics, but structural (like Go interfaces or TypeScript types).

```python
from typing import Protocol

class PaymentService(Protocol):
    def pay(self, amount_cents: int) -> None: ...

class StripePayment:                          # does NOT mention PaymentService
    def pay(self, amount_cents: int) -> None:
        print("charging via Stripe")

def checkout(payment: PaymentService, total: int) -> None:
    payment.pay(total)

checkout(StripePayment(), 1200)              # mypy/pyright: OK — structurally matches
checkout("not a service", 1200)              # mypy/pyright: error — str has no .pay
```

**How it works.** At runtime `Protocol` does nothing (unless you add `@runtime_checkable`, which makes `isinstance(x, PaymentService)` check for the method *names*). At type-check time, mypy/pyright compare shapes. You can define a Protocol *after the fact* for third-party classes you cannot modify — impossible with Java interfaces.

**When to use it:** the go-to choice for **dependency contracts in application code** (repositories, gateways, notifiers) and for **accepting "anything with method X"** in library code (`SupportsRead`, `Iterable`, etc.). It documents the contract, enables IDE completion and type checking, and adds zero coupling.

## 7.3 Level 2 — `ABC`: abstract base classes with runtime enforcement

**What is it?** A base class using the `abc` module; methods marked `@abstractmethod` must be overridden, or instantiation fails **at runtime**.

```python
from abc import ABC, abstractmethod

class PaymentService(ABC):
    @abstractmethod
    def pay(self, amount_cents: int) -> None: ...

    def pay_and_log(self, amount_cents: int) -> None:      # concrete shared behaviour is allowed
        self.pay(amount_cents)
        print(f"paid {amount_cents}")

class StripePayment(PaymentService):                       # explicit `implements`-like relationship
    def pay(self, amount_cents: int) -> None:
        print("stripe")

PaymentService()          # TypeError: Can't instantiate abstract class ... with abstract method pay
class Broken(PaymentService): pass
Broken()                  # TypeError as well — caught when you *create*, not when you *call*
```

**Java comparison.** An `ABC` is exactly a Java abstract class (or an interface with default methods). It is **nominal**: `StripePayment` must inherit. It gives you a runtime guarantee, shared implementation, and a place for template-method patterns.

**When to use it:** when you are designing a **framework or plugin system** where subclasses are the extension point (`class MyCommand(BaseCommand)`), when the base class provides substantial shared logic, or when a wrong implementation must fail *early* (at import/instantiation) rather than on first call. Python's own library uses ABCs for `collections.abc` (`Mapping`, `Sequence`) — inherit from `collections.abc.Mapping`, implement `__getitem__`, `__iter__`, `__len__`, and you get `.get()`, `.keys()`, `in` for free.

## 7.4 Level 3 — Concrete inheritance

Plain subclassing with overridable methods, as in Chapter 6. Use it when you truly have an "is-a" with shared state and behaviour (a `BaseModel` subclass, a custom `Exception` hierarchy, a `DeclarativeBase` ORM model). Do *not* use it to share utilities — use composition or a function.

## 7.5 Choosing

```text
Is the "contract" only needed for readability / tooling?         → Protocol (or nothing)
Do I need to accept third-party objects I cannot modify?          → Protocol
Do I need runtime enforcement that subclasses implement X?         → ABC
Do I need shared implementation + enforced overrides (template)?   → ABC
Is it a framework extension point where users subclass?            → ABC
Is it a small script / test double / one-off?                      → duck typing
Would a function do?                                               → a function
```

A useful mapping for your Java instincts:

| Java habit | Python equivalent |
|---|---|
| `interface Repository<T>` + `JpaRepository` | `Protocol` for the contract, a concrete class per store |
| `abstract class BaseHandler` with template method | `ABC` |
| `Runnable`, `Supplier`, `Function` | `Callable[[...], R]` type hint — just pass a function |
| `Comparable<T>` | `__lt__` + `functools.total_ordering`, or a `key=` function |
| `Iterable<T>` | `collections.abc.Iterable[T]` (a Protocol-like ABC) |
| Marker interface (`Serializable`) | nothing; or a Protocol if there is a method |

**Composition beats all four when the "interface" has one method** — pass a function. `def checkout(charge: Callable[[int], None], total: int)` with `checkout(stripe.pay, 1200)` is the most Pythonic implementation of a single-method `PaymentService`.

## 7.6 Real-world usage

- **FastAPI**: dependency overrides in tests work because dependencies are just callables; no interface required.
- **LangChain**: `BaseChatModel`, `BaseRetriever`, `BaseTool` are ABCs — the framework-extension-point case.
- **SQLAlchemy**: `DeclarativeBase` is concrete inheritance carrying real machinery.
- **Standard library**: `collections.abc` and `typing` ship dozens of Protocols/ABCs (`Iterable`, `Mapping`, `SupportsFloat`); use them in your hints instead of concrete types: accept `Iterable[str]`, not `list[str]`.
- **Your services**: a `Protocol` per external dependency (`class LLMClient(Protocol): def complete(self, prompt: str) -> str`) makes swapping Bedrock for a fake in tests trivial and type-checked.

## Knowledge check — Chapter 7

1. What is the difference between *nominal* and *structural* typing? Which does Java use? Which do `ABC` and `Protocol` use?
2. Why can a `Protocol` be satisfied by a third-party class you cannot edit, while a Java interface cannot?
3. When does an `ABC` violation surface — at import time, at instantiation, or at first method call? Why does that matter for a plugin system?
4. A single-method interface in Java (`interface Notifier { void send(String msg); }`) — what are the two most Pythonic translations?
5. In a FastAPI service with `CustomerRepository` backed by PostgreSQL and an in-memory fake for tests, which of the four levels would you pick for the contract, and why?

---

# 8. Type Hints

This is the chapter that decides whether Python feels like "JavaScript with worse tooling" or like "a language with optional, gradual static typing that a Java developer can enjoy". Modern professional Python is **typed**: FastAPI, Pydantic, SQLAlchemy 2.0, LangChain and the Anthropic SDK are all built around type hints. You must be able to read them fluently and write them well.

## 8.1 What type hints are — and are not

**What is it?** Annotations on parameters, return values, variables and class attributes:

```python
def total(prices: list[float], discount: float = 0.0) -> float:
    return sum(prices) * (1 - discount)

count: int = 0
```

**What they are NOT:**

> Python type hints are primarily for developers and tools. **Python does not become statically typed like Java because hints are present.** The interpreter stores them (in `__annotations__`) and otherwise ignores them. `total("oops", "x")` still runs (and fails at `sum`).

**How it works:**

- At runtime: hints are stored as metadata. Since Python 3.14 (PEP 649/749) annotations are **evaluated lazily** — you can reference classes defined later in the file without quoting them, and importing a module no longer evaluates every annotation. Frameworks read them through `typing.get_type_hints()` or `annotationlib`.
- At development time: a **type checker** (mypy, pyright) reads the whole program, infers the types of un-annotated code, and reports inconsistencies — like `javac`, but as a separate, optional step.
- At runtime, *optionally*: libraries like **Pydantic**, **FastAPI**, **typer**, and the Anthropic **`@beta_tool`** read the hints and *act* on them (validate JSON, parse CLI args, generate a JSON schema for the LLM). This is where hints become more than documentation.

**Why does Python have this design?** Gradual typing (PEP 484, 2015) was added to a 25-year-old dynamically typed language with millions of lines of code. It had to be optional, backwards compatible and enforced by external tools so that the runtime's flexibility (duck typing, monkey patching, `**kwargs`) stayed intact. The pay-off is that you type *where it matters* — public functions, data models, service boundaries — and stay loose in scripts and tests.

**Java comparison:**

| | Java | Python |
|---|---|---|
| Types | mandatory, checked by `javac`, enforced at runtime by the JVM | optional, checked by mypy/pyright, ignored by the interpreter |
| Missing type | compile error | inferred by the checker, or `Any` |
| Wrong type at runtime | impossible (ClassCastException aside) | possible; fails when the wrong operation is attempted |
| Generics | erased at runtime | erased at runtime too (`list[int]` is `list`) |
| Where validation happens | compile time + explicit Bean Validation | type checker (dev) + Pydantic (runtime, at boundaries) |

The right mental model: **type hints + a checker in CI ≈ `javac`'s safety for the code you annotate; Pydantic at the boundaries ≈ Bean Validation + Jackson.**

## 8.2 The vocabulary you must read fluently

Modern syntax (3.10+; 3.12+ for generics syntax). Legacy forms in parentheses — you will see them in older code and they still work.

```python
# Built-in generics — lowercase, no import (legacy: from typing import List, Dict, Set, Tuple)
names: list[str]
ages: dict[str, int]
tags: set[str]
point: tuple[float, float]            # fixed-length tuple
row: tuple[int, ...]                  # variable-length homogeneous tuple

# Optional / union — the | operator (legacy: Optional[str], Union[str, int])
email: str | None = None              # "may be None" — Java Optional<String>, conceptually
value: int | str

# Any — opt out of checking for this value
payload: Any                          # from typing import Any; equivalent to "raw Object + no checks"

# Literal — a fixed set of values (Java enum-like for strings)
from typing import Literal
role: Literal["system", "user", "assistant"]

# Callable — function types (Java Function<A, R>)
from collections.abc import Callable
handler: Callable[[str, int], bool]   # takes (str, int) → returns bool
on_done: Callable[[], None]

# Iterable / Iterator / Sequence / Mapping — accept the abstract shape, not the concrete container
from collections.abc import Iterable, Iterator, Sequence, Mapping
def total(prices: Iterable[float]) -> float: ...       # accepts list, tuple, set, generator...
def lookup(config: Mapping[str, str]) -> str: ...      # accepts dict or any read-only mapping

# Type aliases
type UserId = int                                      # 3.12+ (legacy: UserId = int  or  UserId: TypeAlias = int)
type Json = dict[str, "Json"] | list["Json"] | str | int | float | bool | None

# Final, ClassVar
from typing import Final, ClassVar
MAX_RETRIES: Final = 3
class Config:
    instances: ClassVar[int] = 0       # class variable, not an instance field (matters for dataclasses)

# Annotated — attach metadata (used by FastAPI, Pydantic, typer)
from typing import Annotated
def get_user(user_id: Annotated[int, Path(gt=0)]): ...
```

Two rules of good Python typing that differ from Java habits:

1. **Accept abstract, return concrete.** Parameters: `Iterable[str]`, `Mapping[str, int]`, `Sequence[float]`. Returns: `list[str]`, `dict[str, int]`. (Java's "program to the interface" but with *protocols*, and applied only to inputs.)
2. **Don't over-annotate locals.** `users = repo.find_all()` — the checker infers `list[User]`. Annotate function signatures and class attributes; let inference do the rest.

## 8.3 Generics and `TypeVar`

Python 3.12 introduced Java-like syntax for type parameters:

```python
def first[T](items: Sequence[T]) -> T:            # 3.12+  (legacy: T = TypeVar("T"); def first(items: Sequence[T]) -> T)
    return items[0]

class Repository[T]:                              # generic class
    def __init__(self) -> None:
        self._items: dict[int, T] = {}
    def get(self, id: int) -> T | None:
        return self._items.get(id)
    def add(self, id: int, item: T) -> None:
        self._items[id] = item

repo = Repository[User]()
u = repo.get(1)            # checker knows: User | None

def largest[T: (int, float)](xs: Iterable[T]) -> T: ...          # constrained
def clamp[N: float](x: N) -> N: ...                              # bounded ( <N extends Number> )
```

Generics are **erased at runtime**, exactly like Java: `Repository[User]()` creates a plain `Repository`. They exist for the checker and for frameworks that inspect annotations (Pydantic supports generic models).

## 8.4 `Protocol` and `TypedDict` — typing without classes

Covered in Chapter 7: `Protocol` types "anything with these members". The second tool for typing *shapes* is `TypedDict` — a dict with known keys:

```python
from typing import TypedDict, NotRequired

class ChatMessage(TypedDict):
    role: Literal["system", "user", "assistant"]
    content: str
    name: NotRequired[str]

msg: ChatMessage = {"role": "user", "content": "hi"}     # checker verifies keys and value types
msg["contnt"]                                             # checker: error (typo)
```

`TypedDict` is *just a dict at runtime* — no validation, no methods. It is the right tool for typing JSON-shaped data you pass to/from libraries (boto3's `boto3-stubs` types every response as TypedDicts; LangChain and the LLM SDKs type messages this way). When you need **runtime validation**, use Pydantic (Chapter 9); when you need behaviour, use a dataclass or a class.

## 8.5 Narrowing, `isinstance`, and `None`

The checker tracks control flow. This is how `str | None` becomes safe without `Optional.map`:

```python
def display(user: User | None) -> str:
    if user is None:
        return "anonymous"
    return user.name                    # narrowed to User here

def parse(value: int | str) -> int:
    if isinstance(value, str):
        return int(value)               # str branch
    return value                        # int branch

def find(users: list[User], name: str) -> User | None:
    return next((u for u in users if u.name == name), None)

user = find(users, "Ada")
user.name                               # checker: error — "User | None" has no attribute "name"
assert user is not None                 # narrowing via assert (common in tests)
user.name                               # OK
```

`match` statements narrow too. `typing.TypeGuard` / `TypeIs` let you write custom narrowing functions. Compared to Java's `Optional`, Python's approach is less ceremony and the same safety *if you run the checker*.

## 8.6 Running a type checker

Add one to the project (Chapter 2) and to CI, exactly as you would treat `javac` errors.

```bash
uv add --dev mypy          # or: uv add --dev pyright
uv run mypy src/           # uses [tool.mypy] from pyproject.toml
uv run pyright src/
```

```toml
[tool.mypy]
python_version = "3.14"
strict = true                       # start strict on new projects; relax per-module if needed
plugins = ["pydantic.mypy"]         # better Pydantic inference

[tool.pyright]
typeCheckingMode = "strict"
```

Example session:

```python
# src/app/pricing.py
def apply_discount(price: float, pct: int | None) -> float:
    return price * (1 - pct / 100)
```

```text
$ uv run mypy src/
src/app/pricing.py:2: error: Unsupported operand types for / ("None" and "int")  [operator]
Found 1 error in 1 file (checked 1 source file)
```

Fix: `if pct is None: return price`. That is the same class of error `javac` catches for you — except the fix used a flow check rather than `Optional`.

**mypy vs pyright:** both are good. Pyright is faster and stricter by default, and what VS Code uses live; mypy has the broadest plugin ecosystem (SQLAlchemy, Pydantic, Django). Pick one per project; the `ty` (Astral) and `pyrefly` (Meta) checkers are newer Rust-based options gaining adoption in 2026 but not yet the default.

**Third-party stubs:** libraries without inline types provide `*-stubs` packages: `uv add --dev boto3-stubs[bedrock-runtime,s3]` gives you typed boto3 clients (`response["output"]["message"]` auto-completes). This is the Python equivalent of getting the library's JAR *with* its API — hugely worth it for AWS code.

## 8.7 Where hints do real work: FastAPI, Pydantic, tools

```python
from fastapi import FastAPI
from pydantic import BaseModel

class CreateCustomer(BaseModel):
    name: str
    email: str
    age: int | None = None

app = FastAPI()

@app.post("/customers", status_code=201)
def create_customer(payload: CreateCustomer) -> CreateCustomer:
    return payload
```

Here the hints *are* the contract: FastAPI reads `payload: CreateCustomer`, Pydantic validates the JSON body against the annotations, an OpenAPI schema is generated, and the return annotation defines the response schema. The same annotations drive `@beta_tool` (Anthropic) and `@tool` (LangChain): the parameter types become the JSON schema the model must follow. **In the Python AI world, type hints are executable specifications.** That is the single strongest reason to write them well.

## 8.8 How much to type

| Code | Typing level |
|---|---|
| Public functions, service/repository classes, models | Fully annotated |
| FastAPI routes, Pydantic models, tool functions | Fully annotated (they *need* it) |
| Internal helpers | Annotate signatures; let bodies infer |
| Tests | Light; annotate fixtures' return types if the IDE benefits |
| Notebooks, scripts, experiments | Little or none |

Avoid `Any` as a lazy escape (it disables checking downstream) — use `object` if you mean "anything, and I won't touch it", or a `Protocol`/`TypedDict` if you know the shape.

## Knowledge check — Chapter 8

1. What happens at runtime if you call `total("abc")` where `total(prices: list[float])`? What does that tell you about type hints?
2. Why is `Iterable[str]` a better parameter type than `list[str]`? Which Java principle is this?
3. Explain how the checker makes `user: User | None` safe without an `Optional` wrapper class.
4. What is the difference between `TypedDict`, `dataclass`, and a Pydantic model for representing `{"role": ..., "content": ...}`?
5. Why is "type hints are executable specifications" true for FastAPI and LLM tool definitions but false for the interpreter?
6. Where in the pipeline (editor / CI / runtime) does a Python project catch the error `javac` would catch, and what tool catches it?

---

# 9. Dataclasses and Pydantic

Java has `record` (immutable value carrier), Lombok (`@Data`), Jackson (JSON ↔ objects), and Bean Validation (`@NotNull`, `@Email`). Python covers the same ground with two tools you will use daily: **`dataclasses`** from the standard library for in-process data, and **Pydantic** for data crossing a boundary (JSON, env vars, LLM output) that must be **validated and converted**.

```text
Java                                Python
------------------------------------------------------------------------
record Point(int x, int y)          @dataclass(frozen=True) class Point
Lombok @Data / @Builder              @dataclass
DTO + Jackson + Bean Validation      Pydantic BaseModel
@ConfigurationProperties             pydantic-settings BaseSettings
```

## 9.1 Dataclasses

**What is it?** A decorator that reads the class's annotated attributes and generates `__init__`, `__repr__`, `__eq__` (and optionally ordering and hashing) for you.

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int
    tags: list[str] = field(default_factory=list)     # NOT tags: list[str] = []  (shared mutable default — same trap as functions)
    active: bool = True

u = User("Ada", 36)
u                                # User(name='Ada', age=36, tags=[], active=True)
u == User("Ada", 36)             # True — value equality
u.age = 37                       # mutable by default
```

**Why does Python have it?** Before 3.7, every small data holder needed a hand-written `__init__` that copied N arguments to `self.x = x` plus `__repr__` and `__eq__`. Dataclasses remove that boilerplate exactly as `record`/Lombok did in Java — while leaving the class an ordinary class you can add methods to.

**How does it work?** The decorator inspects `User.__annotations__` in definition order, builds source code for the methods, `exec`s it, and attaches them to the class. Nothing about the class is special afterwards; it is a normal class with generated methods. The annotations are *not* enforced — `User(123, "x")` constructs happily. Dataclasses are about **structure**, not validation.

**Options you will use:**

```python
@dataclass(frozen=True)              # immutable: assignment raises; makes it hashable → usable as dict key/set member
class Point:
    x: float
    y: float

@dataclass(order=True)               # generates <, <=, ... comparing fields in order
class Version: major: int; minor: int

@dataclass(slots=True)               # 3.10+: use __slots__ → less memory, faster attribute access
class Row: id: int; value: float

@dataclass(kw_only=True)             # all fields keyword-only → clear call sites, safe reordering
class Config: host: str; port: int = 5432

# Post-init hook for derived fields / checks
@dataclass
class Order:
    items: list[float]
    total: float = field(init=False)
    def __post_init__(self) -> None:
        self.total = sum(self.items)

from dataclasses import asdict, astuple, replace
asdict(u)                            # {'name': 'Ada', ...} (recursive) — handy before json.dumps
replace(p, x=10)                     # copy-with (Kotlin `copy`, record "wither")
```

**Java comparison.** `@dataclass(frozen=True)` ≈ `record`: immutable, value-equal, hashable, with a canonical constructor. Plain `@dataclass` ≈ Lombok `@Data`: mutable with generated boilerplate. Differences: fields are declared by annotation (no `private final`), defaults are in the declaration (no builder needed for optional fields — keyword arguments do that job), and you can subclass dataclasses (with care about field order).

**Real-world usage.** Domain objects inside a service, results of a computation (`@dataclass class SearchResult: doc_id: str; score: float`), configuration objects built in code, return types richer than a tuple. Also: `match` patterns work on them (Chapter 3), and they type-check well. When the data comes from *outside* (JSON, env, LLM) — reach for Pydantic instead.

## 9.2 Pydantic — validation, parsing and serialisation

**What is it?** A library (v2, core written in Rust) whose `BaseModel` classes **validate and coerce input data into typed Python objects**, and **serialise them back** to dicts/JSON. It is the de-facto standard for request/response models in FastAPI, for settings, for structured LLM output, and for the config objects of LangChain, the Anthropic SDK, and many others.

```python
from pydantic import BaseModel, Field, EmailStr, field_validator, ConfigDict

class Customer(BaseModel):
    id: int
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr                                  # needs `uv add "pydantic[email]"`
    age: int | None = Field(default=None, ge=0, le=150)
    tags: list[str] = []                             # SAFE here — Pydantic copies defaults per instance

    @field_validator("name")
    @classmethod
    def strip_name(cls, v: str) -> str:
        return v.strip()

# Deserialisation (JSON → object) with validation
c = Customer.model_validate({"id": "42", "name": " Ada ", "email": "ada@x.io", "age": 36})
c.id, c.name                    # (42, 'Ada')  — "42" was coerced to int; name stripped
c = Customer.model_validate_json('{"id": 1, "name": "Bo", "email": "bo@x.io"}')

# Validation errors — structured, with locations, like a Bean Validation ConstraintViolation list
Customer(id=1, name="", email="nope")
# pydantic_core._pydantic_core.ValidationError: 2 validation errors for Customer
# name    String should have at least 1 character [type=string_too_short, input_value='', ...]
# email   value is not a valid email address ...

# Serialisation (object → dict / JSON)
c.model_dump()                          # {'id': 1, 'name': 'Bo', 'email': 'bo@x.io', 'age': None, 'tags': []}
c.model_dump(exclude_none=True, mode="json")
c.model_dump_json(indent=2)             # JSON string; handles datetime, UUID, Decimal, nested models
Customer.model_json_schema()            # JSON Schema — this is what FastAPI puts in OpenAPI and what LLM tools use
```

**Why is Pydantic everywhere?** Because dynamic typing has one weak spot: data coming from *outside* the process carries no types. A Java `@RequestBody CustomerDto` gets Jackson + Bean Validation to build a trusted object; Pydantic is that layer for Python, and it also generates the schema (OpenAPI / JSON Schema) from the same declaration. One class = validation + parsing + docs + serialisation. In AI code the same trick constrains LLM output: "answer with JSON matching this schema" → `Model.model_validate_json(llm_text)`.

**How does it work?** At class creation, Pydantic reads the annotations, builds a validation "core schema", and compiles it into a fast validator in `pydantic-core` (Rust). Calling `Model(**data)` or `model_validate` runs that validator: each field is converted (in *lax* mode `"42"` → `42`; `strict=True` disables coercion), constraints checked, validators run, and either a model instance is created or a `ValidationError` listing *all* problems is raised.

**Key features to know:**

```python
# Nested models and collections — validated recursively
class Order(BaseModel):
    id: int
    customer: Customer
    lines: list["OrderLine"]

# Model-level validation (cross-field)
from pydantic import model_validator
class DateRange(BaseModel):
    start: date
    end: date
    @model_validator(mode="after")
    def check(self) -> "DateRange":
        if self.end < self.start:
            raise ValueError("end before start")
        return self

# Config — e.g. build from ORM objects / arbitrary objects with attributes (FastAPI response models from SQLAlchemy rows)
class CustomerOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
CustomerOut.model_validate(sqlalchemy_row)

# Aliases — JSON uses camelCase, Python uses snake_case
from pydantic.alias_generators import to_camel
class ApiModel(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)
    customer_id: int          # accepts/produces "customerId"

# Immutability & strictness
class Frozen(BaseModel):
    model_config = ConfigDict(frozen=True, strict=True)

# Discriminated unions — polymorphic JSON (Jackson @JsonTypeInfo)
from typing import Literal
class Card(BaseModel):  kind: Literal["card"]; number: str
class Bank(BaseModel):  kind: Literal["bank"]; iban: str
class Payment(BaseModel):
    method: Card | Bank = Field(discriminator="kind")

# Validating non-model data
from pydantic import TypeAdapter
TypeAdapter(list[Customer]).validate_python(rows)
```

**Legacy Pydantic v1 forms you will see in old code** (do not write them): `.dict()` → `model_dump()`, `.json()` → `model_dump_json()`, `parse_obj` → `model_validate`, `@validator` → `@field_validator`, `class Config:` → `model_config = ConfigDict(...)`, `Optional[x]` without default now *requires* the field.

## 9.3 Dataclass vs Pydantic — which one?

| | `@dataclass` | Pydantic `BaseModel` |
|---|---|---|
| Validates/coerces input? | No | Yes |
| Cost | Zero deps, tiny | Extra dependency, some overhead per instantiation |
| JSON in/out | `asdict` + `json` manually | built in, with schema |
| Typical use | internal domain objects, results, config built in code | anything crossing a boundary: HTTP, files, env, DB rows to API, LLM output |
| Java analogue | `record` / Lombok | DTO + Jackson + Bean Validation |

A common professional layout: **Pydantic at the edges** (request/response models, settings, LLM tool schemas), **dataclasses or plain classes inside** (domain/service layer), **SQLAlchemy models** for persistence. Small services often skip the middle and pass Pydantic models through — acceptable when the app is small; Chapter 21 shows both.

## 9.4 Settings with pydantic-settings

The configuration counterpart of Spring's `@ConfigurationProperties` + `application.yml` + env overrides:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_", extra="ignore")
    database_url: str
    bedrock_model_id: str = "amazon.nova-pro-v1:0"
    aws_region: str = "us-east-1"
    debug: bool = False

settings = Settings()      # reads APP_DATABASE_URL etc. from env / .env; validation error if missing
```

Chapter 19 covers configuration in full.

## 9.5 Real-world usage

```python
# FastAPI request/response
@app.post("/customers", response_model=CustomerOut, status_code=201)
def create(payload: CreateCustomer): ...

# Structured LLM output (Anthropic SDK helper — Chapter 28)
class Extracted(BaseModel):
    company: str
    sentiment: Literal["positive", "neutral", "negative"]
resp = client.messages.parse(model="claude-opus-5", max_tokens=1024, messages=[...], output_format=Extracted)
resp.parsed_output.sentiment

# LangChain tools / structured output
llm.with_structured_output(Extracted)

# Validating boto3 responses you care about
class ConverseUsage(BaseModel):
    inputTokens: int
    outputTokens: int
usage = ConverseUsage.model_validate(response["usage"])
```

## Knowledge check — Chapter 9

1. Why does `tags: list[str] = []` need `field(default_factory=list)` in a dataclass but is fine in a Pydantic model?
2. `User(123, "x")` succeeds for a dataclass. What does that tell you about the purpose of dataclasses vs. Pydantic?
3. Map these Java pieces to Python: `record`, `@Data`, `@RequestBody` + `@Valid`, `@ConfigurationProperties`, Jackson `@JsonProperty("customerId")`.
4. Where does Pydantic run — at development time like mypy, or at runtime? Why does that make it essential at boundaries?
5. Explain how one Pydantic class can serve as request validation, OpenAPI documentation, and an LLM tool schema at the same time.

---

# 10. Modules and Packages

Java's unit of code is the class, grouped into packages that mirror directories, compiled into JARs on a classpath. Python's unit of code is the **module** — a `.py` file — grouped into **packages** — directories. Understanding how `import` actually works removes a whole category of confusion ("why does my import fail when I run the file directly?", "what is `__init__.py` for?").

## 10.1 Modules

**What is it?** A module is a file `name.py`. Importing it **executes the file top to bottom once**, and the resulting namespace (every name defined at top level) becomes a module object stored in `sys.modules["name"]`.

```python
# pricing.py
TAX_RATE = 0.2                       # module-level "constant"

def with_tax(price: float) -> float:
    return price * (1 + TAX_RATE)

class PriceList: ...

print("pricing loaded")              # runs on first import — side effects at import time are visible!
```

```python
# main.py
import pricing                       # executes pricing.py once, binds the name `pricing`
pricing.with_tax(100)

from pricing import with_tax, TAX_RATE     # bind specific names into *this* namespace
from pricing import with_tax as taxed      # alias
import numpy as np                         # the conventional alias style
```

**Java comparison.** A Java class file is *loaded* by the class loader and its static initialiser runs; nothing else "executes". A Python module *is* a script that runs, and its top-level statements can do anything — define functions and classes (the normal case), but also create a global client, read config, or start a server. This is why:

- **The second import is free.** `sys.modules` caches the module; `import pricing` in ten files runs it once. A module is therefore a natural **singleton** — `settings = Settings()` at module level in `config.py` gives every importer the same object. This replaces a lot of Spring's singleton bean plumbing.
- **Import-time side effects are a smell.** Do not connect to databases or call AWS at import time; create clients lazily or in an app lifespan hook (Chapter 20). Otherwise `import myapp` in a test needs network.
- **Circular imports** are possible (`a` imports `b` imports `a`) and fail with `ImportError: cannot import name ...` when a module is only half-initialised. Fix by restructuring, or importing inside the function that needs it.

## 10.2 Packages and `__init__.py`

**What is it?** A package is a directory containing modules and an `__init__.py` file. The directory name becomes the package name, and `__init__.py` is the module that runs when the package itself is imported.

```text
customer_api/                 ← package
├── __init__.py               ← runs on `import customer_api`; can be empty
├── main.py
├── config.py
├── api/                      ← sub-package
│   ├── __init__.py
│   └── customers.py          ← module: customer_api.api.customers
├── services/
│   ├── __init__.py
│   └── customer_service.py
└── repositories/
    ├── __init__.py
    └── customer_repository.py
```

- `__init__.py` marks a directory as a regular package. It can be empty (most common) or **curate the public API** by re-exporting names:

  ```python
  # customer_api/services/__init__.py
  from .customer_service import CustomerService
  __all__ = ["CustomerService"]          # what `from customer_api.services import *` exports; also documents the API
  ```

  Then callers write `from customer_api.services import CustomerService` instead of the deeper path. Libraries do this heavily (`from fastapi import FastAPI` — `FastAPI` actually lives in `fastapi.applications`).
- Directories without `__init__.py` are *namespace packages* (PEP 420) — importable, but with subtle behaviour. **Always add `__init__.py`** in application code.

**Java comparison.** A Java package is only a namespace; there is no "package object" and nothing to run. A Python package is a real object with attributes (its submodules and whatever `__init__.py` defines). `com.acme.service.CustomerService` ↔ `customer_api.services.customer_service.CustomerService` — with the extra level because the *file* is a namespace too. A JAR ↔ a **distribution** (a wheel on PyPI, e.g. `fastapi`) which may contain one or more importable packages.

## 10.3 How `import` finds things: `sys.path`

```python
import sys; print(sys.path)
# ['', '/app/src', '/usr/lib/python3.14', '.../site-packages', ...]
```

Python searches these directories in order for a top-level name. The entries come from: the directory of the script you ran (or `''` = cwd for `-m`/REPL), `PYTHONPATH`, the standard library, and the active venv's `site-packages`. **That is the classpath**, built at startup.

The two things that bite Java developers:

1. **Running a file inside a package directly** (`python customer_api/main.py`) puts `customer_api/` on `sys.path`, *not* its parent — so `import customer_api.config` fails with `ModuleNotFoundError`. Run modules **with `-m` from the project root**: `python -m customer_api.main`. Or install the project into the venv (uv does this for `src/` layouts) so the package is found through `site-packages` regardless of cwd.
2. **Name shadowing**: a file called `json.py` or `test.py` or `logging.py` in your project shadows the standard library module with the same name. Bizarre errors follow. Don't name modules after stdlib modules.

## 10.4 Absolute vs relative imports

```python
# absolute — preferred; unambiguous
from customer_api.services.customer_service import CustomerService
from customer_api import config

# relative — inside a package, relative to the current module
from .customer_service import CustomerService       # sibling module
from ..config import settings                       # parent package
```

Style guidance: **absolute imports in application code**, relative imports only for tight intra-package references (like inside `__init__.py`). Relative imports break when a module is run as a script, which is another reason to use `-m`.

Import ordering convention (ruff enforces it): standard library, blank line, third-party, blank line, your own package.

## 10.5 `__name__` and `if __name__ == "__main__":`

Every module has a `__name__` attribute. When imported, it is the dotted module name (`"customer_api.main"`). When executed directly (`python main.py` or `python -m customer_api.main`), it is the string `"__main__"`. Hence the idiom:

```python
# customer_api/main.py
from fastapi import FastAPI
app = FastAPI()

def run() -> None:
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)

if __name__ == "__main__":     # only when run as a script — not when imported by tests or by the server
    run()
```

It is the Python replacement for `public static void main`, with one important difference: **the file remains importable without triggering the main behaviour.** Tests can `from customer_api.main import app` without starting a server. A file whose top level does real work (instead of defining things) cannot be imported safely — that is the design reason for the idiom.

For an installed project, `[project.scripts]` in `pyproject.toml` (`customer-api = "customer_api.main:run"`) creates a console command that calls `run()` — the equivalent of a Spring Boot executable JAR's entry point.

## 10.6 The `src/` layout and why professionals use it

```text
customer-api/
├── pyproject.toml
├── src/
│   └── customer_api/          ← the importable package
│       ├── __init__.py
│       └── ...
└── tests/
    └── test_customers.py      ← imports customer_api (installed into the venv by uv)
```

With `src/`, the package is **not** importable from the project root by accident; it must be installed into the venv (`uv sync` does this in editable mode, so edits are live). That guarantees tests exercise the package the way users will install it, and prevents the `sys.path` confusion in 10.3. The alternative "flat" layout (`customer_api/` next to `tests/`) is fine for small apps; Chapter 24 discusses both.

## 10.7 What lives where — a professional module map

| Java | Python |
|---|---|
| `com.acme.customer.CustomerController` | `customer_api/api/customers.py` (router + handlers) |
| `com.acme.customer.CustomerService` | `customer_api/services/customer_service.py` or just `services.py` |
| `CustomerRepository` interface + JPA impl | `customer_api/repositories/customer_repository.py` |
| `Customer` entity | `customer_api/models.py` (SQLAlchemy) |
| `CustomerDto` / `CreateCustomerRequest` | `customer_api/schemas.py` (Pydantic) |
| `application.yml` + `@ConfigurationProperties` | `customer_api/config.py` (`Settings`) |
| `Application.java` | `customer_api/main.py` |
| `StringUtils` | `customer_api/utils.py` (functions) |

Python files are commonly **several related classes/functions per module** ("schemas.py" holds all schemas); one-class-per-file is a Java habit, not a Python requirement. Split when a file gets long or has separate concerns, not by class count.

## Knowledge check — Chapter 10

1. What does `import pricing` actually *do* the first time, and the second time? Why does that make module-level objects natural singletons?
2. Why does `python customer_api/main.py` fail with `ModuleNotFoundError` while `python -m customer_api.main` works?
3. What is `__init__.py` for, and what are its two common contents?
4. Explain `if __name__ == "__main__":` to a Java developer, including *why* the idiom exists (hint: importability).
5. Which Java concept corresponds to a wheel on PyPI, and which to a Python package directory? Why are they not the same thing?

---

# 11. Exceptions

Python's exceptions look like Java's — `try`, `except`, `finally`, `raise`, class hierarchies — but the *philosophy* differs, and the difference explains most of the code you will read.

## 11.1 The syntax

```python
def load_config(path: str) -> dict:
    try:
        with open(path) as f:                    # may raise FileNotFoundError (an OSError)
            data = json.load(f)                  # may raise json.JSONDecodeError (a ValueError)
    except FileNotFoundError:
        return {}                                # missing file is normal — default config
    except json.JSONDecodeError as e:
        raise ConfigError(f"invalid config {path}: {e}") from e   # wrap, keep the cause
    else:
        log.info("config loaded from %s", path)  # runs only if NO exception occurred
        return data
    finally:
        log.debug("config load attempted")       # always runs
```

- `except X as e` binds the exception; `e` is deleted at the end of the block (Python 3 quirk).
- `except (TypeError, ValueError):` catches several (tuple, not `|`).
- `else:` runs when the `try` body completed without exception — useful to keep the `try` body minimal so you don't accidentally catch exceptions from code that was not supposed to be guarded.
- `raise` with no argument inside `except` re-raises the current exception (Java: `throw e;` but preserving the original traceback).
- `raise NewError(...) from e` sets `__cause__` — printed as "The above exception was the direct cause of the following exception". Java's `new X(msg, cause)`. `from None` suppresses the chain.

## 11.2 No checked exceptions — and what replaces them

**The important difference:**

> Python has **no checked exceptions**. No `throws` clause, no compiler forcing you to catch or declare. Any function may raise anything.

**Why?** Python's designers (like C#'s and Kotlin's) judged that checked exceptions produce more `catch (Exception e) {}` and wrapper-exception boilerplate than safety. Combined with dynamic typing there is nothing at compile time to enforce it anyway.

**What replaces it:**

1. **Documentation and naming.** Docstrings say "Raises: `ValueError` if …". Library docs list exceptions (`botocore.exceptions.ClientError`, `httpx.HTTPStatusError`).
2. **A well-designed hierarchy** so callers can catch at the level they care about — `except OSError` catches every file/network OS error; `except httpx.HTTPError` catches all httpx failures.
3. **Let it propagate.** The default Python stance is: if you can't *handle* it meaningfully here, don't catch it. The web framework's error handler or the CLI's top level will turn it into a 500 / an error message with a traceback. Catch-log-rethrow wrappers at every layer are a Java-accent.
4. **EAFP** — "Easier to Ask Forgiveness than Permission." Python prefers *trying* an operation and catching the exception over checking beforehand:

   ```python
   # LBYL (Java style)                       # EAFP (Python style)
   if key in config:                          try:
       value = config[key]                        value = config[key]
   else:                                      except KeyError:
       value = default                            value = default
   # (…or simply config.get(key, default))
   ```

   EAFP avoids race conditions (file exists → deleted → open fails) and double lookups, and it is idiomatic because exceptions in CPython are cheap to *set up* (the `try` costs nothing) and moderately expensive only when *raised*. Using exceptions for expected control flow — `StopIteration` ends every `for` loop! — is normal in Python.

## 11.3 The hierarchy you need

```text
BaseException
├── SystemExit, KeyboardInterrupt, GeneratorExit     ← don't catch these (bare `except:` does — never write bare except)
└── Exception                                        ← the root of "normal" errors; catch this at top-level boundaries only
    ├── ValueError          bad value ("42x" → int)           ≈ IllegalArgumentException
    ├── TypeError           wrong type (1 + "a", missing arg)  ≈ ClassCastException / IllegalArgumentException
    ├── KeyError            missing dict key                   ≈ (a null from Map.get, made loud)
    ├── IndexError          list index out of range            ≈ IndexOutOfBoundsException
    ├── AttributeError      obj has no attribute               ≈ NullPointerException (on None) / NoSuchMethodError
    ├── LookupError         parent of KeyError, IndexError
    ├── OSError             file/network/OS failures           ≈ IOException
    │   ├── FileNotFoundError, PermissionError, TimeoutError, ConnectionError…
    ├── RuntimeError        generic; NotImplementedError, RecursionError
    ├── StopIteration       iterator exhausted (protocol, not an error)
    ├── ImportError / ModuleNotFoundError
    ├── AssertionError      `assert` failed
    └── (library roots)     httpx.HTTPError, botocore.exceptions.ClientError, pydantic.ValidationError, sqlalchemy.exc.SQLAlchemyError, anthropic.APIError…
```

Notable mappings: Python has **no `NullPointerException`** — calling a method on `None` is `AttributeError: 'NoneType' object has no attribute 'x'`. There is **no RuntimeException/Exception split**: everything is unchecked. `assert` is stripped with `python -O`; never use it for validation of input, only for invariants and tests.

## 11.4 Custom exceptions

```python
class AppError(Exception):
    """Base for all application errors — lets callers `except AppError`."""

class NotFoundError(AppError):
    def __init__(self, entity: str, id: object):
        super().__init__(f"{entity} {id} not found")
        self.entity, self.id = entity, id

class ConflictError(AppError): ...
```

Keep them small; store structured fields for handlers to use; give the project *one* base class. In FastAPI you then register a single handler that maps `NotFoundError → 404`, `ConflictError → 409` (Chapter 21) — the equivalent of `@ControllerAdvice`.

## 11.5 Practical patterns

```python
# Retry a transient failure (real code should use `tenacity` or the SDK's built-in retries)
for attempt in range(3):
    try:
        return client.converse(**kwargs)
    except client.exceptions.ThrottlingException:
        time.sleep(2 ** attempt)
raise RuntimeError("gave up after 3 attempts")

# Handle a boto3 error by code (boto3 raises one ClientError type with a code inside)
from botocore.exceptions import ClientError
try:
    s3.get_object(Bucket=b, Key=k)
except ClientError as e:
    if e.response["Error"]["Code"] == "NoSuchKey":
        return None
    raise                                      # anything else: propagate

# Exception groups (3.11+) — several failures at once, e.g. from asyncio.TaskGroup
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch(a)); tg.create_task(fetch(b))
except* httpx.HTTPError as eg:                 # note the star
    for e in eg.exceptions: log.error(e)

# contextlib.suppress — the clean "ignore this specific error"
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove(tmp_path)

# Add context notes (3.11+)
try:
    process(row)
except ValueError as e:
    e.add_note(f"while processing row {i}")
    raise
```

**Anti-patterns to unlearn:** `except Exception: pass` (silences everything), catching `Exception` in business logic just to log and rethrow, wrapping every library exception in your own at every layer, using return codes / `None` to signal errors that should raise.

## Knowledge check — Chapter 11

1. Python has no checked exceptions. What three practices replace the safety `throws` was supposed to give?
2. Explain EAFP with an example and say why it is not "using exceptions for control flow, which is bad" in Python.
3. What does `raise ... from e` do, and what is its Java equivalent? When would you use `from None`?
4. Why should you never write a bare `except:`? What does it catch that `except Exception:` does not?
5. What is Python's equivalent of a `NullPointerException`, and what is the equivalent of `@ControllerAdvice` for turning domain errors into HTTP codes?

---

# 12. Files, JSON and HTTP

This chapter is practical backend Python: the operations you will do in every service, script and Lambda. It also builds the first small "real" client.

## 12.1 Files and paths

Use `pathlib.Path` — an object-oriented path API (like `java.nio.file.Path` but friendlier) — and the `with` statement to guarantee closing (Chapter 16).

```python
from pathlib import Path

root = Path(__file__).parent            # directory of the current module (Java: no easy equivalent)
data = root / "data" / "customers.json" # `/` joins paths
data.exists(), data.is_file(), data.suffix, data.stem, data.name
data.parent.mkdir(parents=True, exist_ok=True)

text = data.read_text(encoding="utf-8")           # whole file as str — fine for small files
data.write_text(json.dumps(payload), encoding="utf-8")
raw: bytes = data.read_bytes()

with open(data, encoding="utf-8") as f:           # explicit; default mode "r" = text
    for line in f:                                # streams line by line — memory-safe for huge files
        process(line.rstrip("\n"))

with open("out.csv", "w", newline="") as f:       # modes: r, w (truncate), a (append), x (create), rb/wb (bytes)
    writer = csv.writer(f)
    writer.writerow(["id", "name"])

for p in root.glob("**/*.md"):                    # recursive glob
    ...
```

Text mode decodes to `str` (always pass `encoding="utf-8"` — the default is platform-dependent until 3.15); binary mode gives `bytes`. Java's `Files.readAllLines` / `BufferedReader` ↔ `read_text` / iterating the file object.

## 12.2 JSON

The standard library `json` maps JSON to plain Python objects — no bean mapping unless you add Pydantic:

```text
JSON object  → dict        JSON array → list
string → str, number → int / float, true/false → True/False, null → None
```

```python
import json

payload = {"name": "Ada", "tags": ["a", "b"], "active": True, "score": None}
s = json.dumps(payload)                         # '{"name": "Ada", "tags": ["a", "b"], "active": true, "score": null}'
json.dumps(payload, indent=2, ensure_ascii=False, sort_keys=True)
obj = json.loads(s)                             # back to dict
obj["tags"][0]

with open("cfg.json") as f: cfg = json.load(f)  # load/dump = file variants; loads/dumps = string variants
with open("cfg.json", "w") as f: json.dump(cfg, f, indent=2)

json.dumps(datetime.now())                      # TypeError: Object of type datetime is not JSON serializable
json.dumps(obj, default=str)                    # crude fallback; or use Pydantic's model_dump_json()
```

**Java comparison.** Jackson maps JSON ↔ typed POJOs, with dozens of annotations. Python's `json` maps JSON ↔ dicts/lists, full stop. To get typed objects you add Pydantic: `Customer.model_validate_json(s)` / `customer.model_dump_json()`. For most scripts, dicts are enough and much faster to write.

## 12.3 Environment variables

```python
import os

db_url = os.environ["DATABASE_URL"]             # KeyError if missing — fail fast
region = os.environ.get("AWS_REGION", "us-east-1")
os.getenv("DEBUG", "false").lower() == "true"   # everything is a string; parse it
```

For real applications use `pydantic-settings` (Chapters 9, 19) so parsing, defaults and `.env` are handled once. Java's `System.getenv` / Spring's relaxed binding ↔ `os.environ` / `BaseSettings`.

## 12.4 HTTP clients: `requests` and `httpx`

| Library | Sync | Async | HTTP/2 | Notes |
|---|---|---|---|---|
| `requests` | ✅ | ❌ | ❌ | The classic (2011). Everywhere in tutorials and older code. Fine for scripts. |
| **`httpx`** | ✅ | ✅ | ✅ | Same API shape as requests, plus async. **Use this.** FastAPI's `TestClient` is built on it. |
| `urllib.request` | ✅ | ❌ | ❌ | Standard library, clunky; use only when you cannot add dependencies (some Lambda cases). |
| `aiohttp` | ❌ | ✅ | ❌ | Older async client/server; common in async-first codebases. |

```python
import httpx

# One-off
r = httpx.get("https://api.example.com/customers", params={"page": 1}, timeout=10.0)
r.status_code                     # 200
r.raise_for_status()              # raise httpx.HTTPStatusError for 4xx/5xx
data = r.json()                   # parsed body → dict/list
r.headers["content-type"]; r.text; r.content (bytes)

# Client with connection pooling, base URL and default headers — like a configured RestClient/WebClient bean
with httpx.Client(base_url="https://api.example.com", headers={"Authorization": f"Bearer {token}"}, timeout=10.0) as client:
    r = client.post("/customers", json={"name": "Ada", "email": "ada@x.io"})   # json= serialises + sets content-type
    r = client.get(f"/customers/{cid}")
    r = client.put(f"/customers/{cid}", json=payload)
    r = client.delete(f"/customers/{cid}")
```

Streaming downloads/uploads: `with client.stream("GET", url) as r: for chunk in r.iter_bytes(): ...`. SSE from an LLM endpoint: `for line in r.iter_lines(): ...`.

## 12.5 Synchronous vs asynchronous HTTP — the practical view

Java has `RestClient` (blocking) and `WebClient` (reactive). Python has the same split, and the same rule: **pick one style per application**.

```python
# Sync: one request blocks the calling thread until the response arrives.
def fetch_all(ids):
    with httpx.Client() as c:
        return [c.get(f"/items/{i}").json() for i in ids]      # N sequential round-trips

# Async: the coroutine yields while waiting; the event loop runs other coroutines meanwhile.
import asyncio
async def fetch_all(ids):
    async with httpx.AsyncClient() as c:
        responses = await asyncio.gather(*(c.get(f"/items/{i}") for i in ids))   # N concurrent round-trips
        return [r.json() for r in responses]
```

When does async matter? **When you have many concurrent I/O waits** — fan-out to 50 services, thousands of concurrent WebSocket/SSE clients, an API gateway that mostly waits on LLM providers. Async gives you that concurrency with one thread, cheaply. For a script that makes three calls, or a CRUD API where each request does one DB query, sync is simpler and just as fast. Chapter 17 gives the complete picture (event loop, GIL, when threads are enough). The practical decision rule:

- FastAPI service calling LLMs/other services with fan-out or streaming → **async** (`httpx.AsyncClient`, async SDK clients).
- CLI, batch script, Lambda doing sequential work, simple CRUD → **sync**.
- **Never mix** a blocking call (`requests.get`, `time.sleep`, sync boto3) inside an `async def` — it freezes the whole event loop. Use the async client, or push the blocking call to a thread with `asyncio.to_thread(...)`.

## 12.6 Build a small REST client

A typed, tested client for a hypothetical Customers API — the shape you will reuse for any third-party API (and for calling a Python AI service from anywhere).

```python
# customers_client.py
from __future__ import annotations
import httpx
from pydantic import BaseModel

class Customer(BaseModel):
    id: int
    name: str
    email: str

class CreateCustomer(BaseModel):
    name: str
    email: str

class CustomersClient:
    """Thin typed wrapper over the Customers REST API."""

    def __init__(self, base_url: str, api_key: str, timeout: float = 10.0):
        self._http = httpx.Client(
            base_url=base_url,
            headers={"Authorization": f"Bearer {api_key}", "Accept": "application/json"},
            timeout=timeout,
        )

    def get(self, customer_id: int) -> Customer | None:
        r = self._http.get(f"/customers/{customer_id}")
        if r.status_code == 404:
            return None
        r.raise_for_status()
        return Customer.model_validate(r.json())

    def list(self, page: int = 1, size: int = 50) -> list[Customer]:
        r = self._http.get("/customers", params={"page": page, "size": size})
        r.raise_for_status()
        return [Customer.model_validate(item) for item in r.json()["items"]]

    def create(self, payload: CreateCustomer) -> Customer:
        r = self._http.post("/customers", json=payload.model_dump())
        r.raise_for_status()
        return Customer.model_validate(r.json())

    def close(self) -> None:
        self._http.close()

    # Make it usable with `with` (Chapter 16)
    def __enter__(self) -> "CustomersClient":
        return self
    def __exit__(self, *exc) -> None:
        self.close()


if __name__ == "__main__":
    import os
    with CustomersClient("https://api.example.com", os.environ["API_KEY"]) as client:
        created = client.create(CreateCustomer(name="Ada", email="ada@x.io"))
        print(created)
        print(client.get(created.id))
```

And a test without a network, using `httpx.MockTransport` (Chapter 18 covers pytest in depth):

```python
# test_customers_client.py
import httpx
from customers_client import CustomersClient, CreateCustomer

def test_create_customer():
    def handler(request: httpx.Request) -> httpx.Response:
        assert request.url.path == "/customers"
        assert request.headers["authorization"] == "Bearer k"
        return httpx.Response(201, json={"id": 1, "name": "Ada", "email": "ada@x.io"})

    client = CustomersClient("https://api.example.com", "k")
    client._http = httpx.Client(base_url="https://api.example.com", transport=httpx.MockTransport(handler),
                                headers={"Authorization": "Bearer k"})
    created = client.create(CreateCustomer(name="Ada", email="ada@x.io"))
    assert created.id == 1
```

**Compare with Java.** The Spring version would be a `RestClient` bean + a `record CustomerDto` + `@Service` + Jackson config. The Python version is ~40 lines with no framework: `httpx` for transport, Pydantic for the DTOs, a plain class for the client, `with` for lifecycle. That "no framework needed for simple things" is the normal Python feeling.

## Knowledge check — Chapter 12

1. What Python types does `json.loads` produce, and how do you get typed objects from it?
2. Why should you use `with open(...)` (or `Path.read_text`) rather than `f = open(...)`? What Java construct is this?
3. Explain the difference between `httpx.Client` and `httpx.AsyncClient`, and give one situation where each is the right choice.
4. What goes wrong if you call `requests.get()` inside an `async def` FastAPI endpoint?
5. What does `raise_for_status()` do, and how does a Python client conventionally represent "404 → not found" to its caller?

---

# Part III — Writing Python like a Python developer

---

# 13. Pythonic Programming

"Pythonic" is not about cleverness. It means: **the code uses the language's native idioms so that another Python developer reads it without friction.** The goal of this chapter is to make your Python look like Python — not like Java transliterated — *without* tipping into the opposite failure, the show-off one-liner.

Python's own statement of taste is built in: `python -c "import this"` prints *The Zen of Python*. The lines that matter most for a Java developer:

> Beautiful is better than ugly. Explicit is better than implicit. Simple is better than complex. Flat is better than nested. Readability counts. There should be one — and preferably only one — obvious way to do it. If the implementation is hard to explain, it's a bad idea.

## 13.1 The core idioms, side by side

Each pair below shows the Java-accented version and the Pythonic version. The Pythonic one is not shorter for its own sake — it removes a concept (an index, a temporary, a flag) that the reader would otherwise have to track.

**Iterate, don't index.**

```python
# Java accent                                    # Pythonic
for i in range(len(users)):                      for user in users:
    print(users[i].name)                             print(user.name)

for i in range(len(users)):                      for i, user in enumerate(users):
    print(i, users[i].name)                          print(i, user.name)

for i in range(len(names)):                      for name, score in zip(names, scores):
    print(names[i], scores[i])                       print(name, score)
```

**Build collections with comprehensions when the transformation is simple.**

```python
result = []                                      result = [u.email for u in users if u.active]
for u in users:
    if u.active:
        result.append(u.email)

index = {}                                       index = {u.id: u for u in users}
for u in users:
    index[u.id] = u
```

**Use `any` / `all` with a generator instead of loops with flags.**

```python
found = False                                    if any(u.email == email for u in users):
for u in users:                                      ...
    if u.email == email:
        found = True                             if all(line.strip() for line in lines):
        break                                        ...
```

**Unpack instead of indexing tuples.**

```python
pair = get_pair()                                lo, hi = get_pair()
lo = pair[0]; hi = pair[1]
for item in pairs:                               for key, value in pairs:
    key = item[0]; value = item[1]
```

**Use truthiness and `in`.**

```python
if len(items) > 0:                               if items:
if name != None and name != "":                  if name:
if key in d.keys():                              if key in d:
if x == 1 or x == 2 or x == 3:                   if x in (1, 2, 3):
```

**Use `dict.get`, `setdefault`, `defaultdict`, `Counter`.**

```python
if k in counts:                                  counts[k] = counts.get(k, 0) + 1
    counts[k] += 1                               # or: counts = Counter(items)
else:
    counts[k] = 1
```

**Use `with` for anything that must be closed.** (Chapter 16.)

**Return early; avoid deep nesting** ("flat is better than nested"). Same as Java best practice, but Python's lack of braces makes deep nesting hurt more.

**Prefer functions and modules over classes** for stateless behaviour. (Chapter 5.)

**Use f-strings**, never `%` or `.format()` in new code.

**Let exceptions propagate** unless you can handle them. (Chapter 11.)

**Use `pathlib`**, not `os.path` string surgery.

## 13.2 `map`, `filter`, and when comprehensions win

`map`, `filter`, `functools.reduce` exist and return lazy iterators. Modern Python prefers comprehensions for most cases because the expression is visible inline:

```python
list(map(lambda u: u.email, users))          # works, but reads worse than
[u.email for u in users]

list(filter(None, values))                   # OK idiom: drop falsy values
map(str, numbers)                            # OK: existing function, no lambda — reads fine
```

Rule: **`map`/`filter` with an existing named function is fine; with a `lambda`, use a comprehension.** `reduce` is almost always clearer as a loop, `sum`, `max`, or `"".join`.

## 13.3 Generators and laziness as a habit

`(x for x in ...)` — parentheses instead of brackets — produces a lazy generator (Chapter 14). Idiomatic Python passes generators straight into consumers instead of materialising lists:

```python
total = sum(order.total for order in orders)                 # no intermediate list
first_admin = next((u for u in users if u.is_admin), None)   # first match or None
", ".join(str(n) for n in numbers)
```

## 13.4 Context managers as a habit

Any resource — file, DB session, HTTP client, lock, temporary directory, timer — is used inside `with`. If you find yourself writing `try: ... finally: x.close()`, there is almost certainly a context manager for it (Chapter 16).

## 13.5 When "Pythonic" becomes "clever" — the line

The same features that make Python readable make it easy to write unreadable code. Signs you have crossed the line:

| Clever (avoid) | Why | Do instead |
|---|---|---|
| Comprehension longer than one line or with 3+ `for`/`if` clauses | reader must mentally unroll it | a `for` loop |
| Nested comprehensions producing nested lists | same | loop, or a helper function |
| Walrus `:=` in a comprehension condition and body | two things at once | a loop |
| `lambda` with a conditional inside a `key=` inside `sorted` inside a comprehension | four levels of indirection | `def sort_key(u): ...` |
| Chains of `and`/`or` used as if/else (`x and y or z`) | fails when `y` is falsy | `y if x else z` |
| Dunder tricks / metaclasses in application code | framework machinery in business logic | plain classes |
| One-line `try/except` swallowing errors | hides bugs | explicit handling |
| Reflection (`getattr(self, "handle_" + kind)()`) for dispatch | not discoverable, not typed | a dict of functions or `match` |
| Playing code golf with `*`/`**`, slicing and `zip(*matrix)` | readers stop | write it out |

The tie-breaker: **would a competent colleague understand this in five seconds without a comment?** If a comment is required to explain *how* the line works (rather than *why* it exists), rewrite it.

## 13.6 A worked example

Java-accented Python (a real-world shape):

```python
class UserProcessor:
    def __init__(self):
        self.results = []

    def process(self, users):
        for i in range(0, len(users)):
            user = users[i]
            if user.get("active") == True:
                if user.get("email") != None:
                    if len(user.get("email")) > 0:
                        self.results.append(user.get("email").lower())
        return self.results

processor = UserProcessor()
emails = processor.process(users)
```

Pythonic:

```python
def active_emails(users: Iterable[dict]) -> list[str]:
    """Lower-cased e-mails of active users that have one."""
    return [u["email"].lower() for u in users if u.get("active") and u.get("email")]
```

What changed: no class holding accumulated state (a function returning a value is safer and testable), no index loop, truthiness instead of `== True` / `!= None` / `len() > 0`, a comprehension because the transformation is one filter + one map, a docstring instead of a class name. Same behaviour; the reader sees the intent at once.

## Knowledge check — Chapter 13

1. Why is `for i in range(len(xs))` considered un-Pythonic *even though it works*? What concept does the Pythonic form remove for the reader?
2. Give the rule of thumb for `map`/`filter` vs comprehensions.
3. Name three signs that a comprehension should be turned back into a loop.
4. Rewrite in Pythonic style: `if len(name) > 0 and name != None:`. What is one input where the rewrite behaves differently, and does it matter?
5. Explain "flat is better than nested" and how early returns and `any()`/`all()` implement it.

---

# 14. Iterators and Generators

This is the chapter where Python's data-processing style diverges most from typical Java code. Java developers reach for collections and, since Java 8, Streams. Python developers reach for **iterators** — and the tool for making them, **generators** — and the entire language (`for`, comprehensions, `sum`, `zip`, `json`, files, DB cursors, LLM streaming responses) is built on that protocol.

## 14.1 The iteration protocol

**What is it?**

- An **iterable** is any object that can produce an iterator: it has `__iter__()`. Lists, tuples, dicts, sets, strings, files, ranges, generators, DB result sets, httpx streaming responses.
- An **iterator** is an object with `__next__()` that returns the next value or raises `StopIteration` when done. It is *single-pass* and *stateful*. Every iterator is also an iterable (its `__iter__` returns itself).

The `for` loop is sugar over this protocol:

```python
for item in items:                  # is exactly:
    body(item)                      it = iter(items)          # calls items.__iter__()
                                    while True:
                                        try:
                                            item = next(it)   # calls it.__next__()
                                        except StopIteration:
                                            break
                                        body(item)
```

```python
>>> it = iter([10, 20])
>>> next(it)
10
>>> next(it)
20
>>> next(it)
StopIteration
>>> next(iter([]), "default")     # next() with a default — the "first or default" idiom
'default'
```

**Java comparison.** `Iterable<T>` / `Iterator<T>` with `hasNext()` / `next()` is the same idea. Differences: Python has no `hasNext()` — you *try* `next()` and catch `StopIteration` (EAFP, Chapter 11); and *far* more things are iterables in Python, so `for` covers almost every loop you write.

Iterators are consumed once:

```python
it = iter([1, 2, 3])
list(it)        # [1, 2, 3]
list(it)        # []   ← exhausted. A list is re-iterable; an iterator is not.
```

This bites when you pass a generator to two consumers. If you need to iterate twice, materialise with `list(...)` or use `itertools.tee`.

## 14.2 Generators: functions that produce iterators

**What is it?** A function containing `yield` is a **generator function**. Calling it does not run the body; it returns a **generator object** (an iterator). Each `next()` runs the body until the next `yield`, hands out that value, and *suspends* — locals and position preserved — until the next `next()`.

```python
def count_up(limit: int):
    n = 0
    while n < limit:
        yield n                 # produce a value, pause here
        n += 1                  # resume here on the next next()
    # falling off the end raises StopIteration for the consumer

gen = count_up(3)
type(gen)                      # <class 'generator'>
next(gen), next(gen), next(gen)   # 0, 1, 2
next(gen)                      # StopIteration
for n in count_up(3): print(n) # the normal way to use it
```

**Why does Python have it?** Writing an iterator class (`__iter__`, `__next__`, state fields) is tedious. A generator lets you write the *producing logic as ordinary sequential code* and get an iterator for free. It is also the natural expression of **lazy evaluation**: values are computed on demand, one at a time, with O(1) memory.

**How does it work?** The generator object holds a suspended frame (the function's locals and instruction pointer). `next()` resumes the frame on the current thread, runs until `yield`, saves the frame, returns. Nothing runs concurrently — a generator is a **coroutine-shaped control-flow trick, not a thread**. (This same machinery, extended, is what `async def` / `await` are built on — Chapter 17.)

## 14.3 Why generators matter: processing large data

```python
def read_records(path: Path) -> Iterator[dict]:
    """Stream JSON-lines records without loading the whole file."""
    with path.open(encoding="utf-8") as f:
        for line in f:                       # the file is itself a lazy iterator of lines
            if line.strip():
                yield json.loads(line)

def only_active(records: Iterable[dict]) -> Iterator[dict]:
    for r in records:
        if r.get("active"):
            yield r

def batched(items: Iterable[T], size: int) -> Iterator[list[T]]:
    batch: list[T] = []
    for item in items:
        batch.append(item)
        if len(batch) == size:
            yield batch
            batch = []
    if batch:
        yield batch
# (Python 3.12+: itertools.batched(items, size) does this)

# A pipeline: 10 GB file, constant memory, each stage pulls from the previous
for batch in batched(only_active(read_records(Path("events.jsonl"))), 100):
    embeddings = embed(batch)                # e.g. 100 texts per Bedrock call
    store(embeddings)
```

Each stage is a generator; the `for` at the bottom *pulls* one item at a time through the chain. No stage builds a list. Swap `read_records` for a generator over an S3 stream or a DB cursor and nothing else changes. This is the pattern behind ETL scripts, document chunking for RAG, log processing, and paginated API iteration (boto3 *paginators* are generators: `for page in s3.get_paginator("list_objects_v2").paginate(Bucket=b)`).

**Generator expressions** are the inline form — comprehensions with parentheses:

```python
total_bytes = sum(len(line) for line in open(path))     # never holds the file in memory
```

## 14.4 Generators vs Java Streams

They are *related* but **not the same thing**, and knowing the difference prevents wrong expectations.

| | Java Stream | Python generator / iterator |
|---|---|---|
| Laziness | Yes — intermediate ops are lazy until a terminal op | Yes — each `next()` computes one value |
| Single use | Yes | Yes |
| Composition style | Fluent method chain: `.filter().map().collect()` | Nested calls / pipelines of generator functions / comprehensions; `itertools` helpers |
| Built-in parallelism | `.parallel()` | None (a generator runs on the caller's thread) |
| Terminal operations | `collect`, `reduce`, `count`… | any consumer: `for`, `list()`, `sum()`, `max()`, `"".join()` |
| Pushing values *in* | No | Yes — `gen.send(value)` (rarely used directly; the basis of coroutines) |
| Infinite sequences | possible (`Stream.iterate`) | natural (`while True: yield`) |
| Where it shines | declarative collection transformations | streaming I/O, pipelines, custom iteration, memory-bounded processing |

The Stream API is a *library for transforming collections*. A generator is a *language mechanism for producing values lazily*. Comprehensions cover the everyday `filter/map/collect`; generators cover the lazy/streaming part; `itertools` covers the rest (`chain`, `islice`, `groupby`, `takewhile`, `accumulate`, `product`, `batched`, `pairwise`).

## 14.5 Writing an iterable class (when you need to)

Sometimes an object should be iterable more than once, e.g. a paginated resource. Implement `__iter__` — usually *as a generator*:

```python
class Paginated:
    def __init__(self, client, path: str, page_size: int = 100):
        self.client, self.path, self.page_size = client, path, page_size

    def __iter__(self) -> Iterator[dict]:
        page = 1
        while True:
            items = self.client.get(self.path, params={"page": page, "size": self.page_size}).json()["items"]
            if not items:
                return                     # ends iteration
            yield from items               # delegate to another iterable
            page += 1

for customer in Paginated(client, "/customers"):     # re-iterable; each `for` starts fresh
    ...
```

`yield from` forwards every item of a sub-iterable (and, in coroutine land, delegates control). It replaces `for x in sub: yield x`.

## 14.6 Generators in the AI world

- **LLM streaming**: `for text in stream.text_stream:` (Anthropic), `for event in client.converse_stream(...)["stream"]:` (Bedrock), `for chunk in llm.stream(prompt):` (LangChain) — all iterators yielding tokens/chunks as they arrive.
- **FastAPI `StreamingResponse(generator())`** streams a generator's output to the HTTP client — the easy way to forward LLM tokens (Project 3).
- **Datasets**: Hugging Face `datasets` supports streaming mode — an iterator over terabyte-scale data.
- **Document loaders**: LangChain loaders expose `lazy_load()` generators to avoid loading every document in memory.

## Knowledge check — Chapter 14

1. Define iterable and iterator precisely. Which one is a `list`, and which is what `iter(list)` returns? Which is a generator?
2. What actually happens when `next()` is called on a generator? Why is a generator *not* a thread?
3. Why does `list(gen)` return `[]` the second time? How do you fix it when you genuinely need two passes?
4. Explain in what sense Java Streams and Python generators are related, and give two concrete ways they differ.
5. Sketch a three-stage generator pipeline that reads a huge file, filters lines, and yields batches of 100 — and say why memory stays constant.

---

# 15. Decorators

Decorators look like Java annotations and are used where Java uses annotations, AOP and proxies — but they are a *completely different mechanism*: **plain functions, executed at definition time, that receive a function (or class) and return a replacement.** Understanding them from first principles is what lets you read FastAPI, pytest, dataclasses, LangChain and the Anthropic SDK with confidence.

## 15.1 First principles: `@` is function application

```python
@my_decorator
def hello():
    ...
```

means **exactly**:

```python
def hello():
    ...
hello = my_decorator(hello)
```

That is all the syntax does. `my_decorator` is called *once*, at the moment the `def` statement executes (at import time), with the function object as its argument. Whatever it returns is bound to the name `hello`. Usually it returns a new function that *wraps* the original; sometimes it returns the original after registering it somewhere; sometimes a completely different object.

Everything follows from Chapter 5: functions are objects; functions can take and return functions; closures let the returned function remember the original.

## 15.2 Building one: logging

```python
import functools
import logging

log = logging.getLogger(__name__)

def logged(func):                                   # takes a function
    @functools.wraps(func)                          # copy __name__, __doc__, signature → keeps introspection intact
    def wrapper(*args, **kwargs):                   # accepts anything, so it can wrap any function
        log.info("→ %s args=%s kwargs=%s", func.__name__, args, kwargs)
        result = func(*args, **kwargs)              # call the original
        log.info("← %s = %r", func.__name__, result)
        return result
    return wrapper                                  # this replaces the original

@logged
def add(a: int, b: int) -> int:
    return a + b

add(2, 3)          # logs "→ add args=(2, 3) kwargs={}" then "← add = 5"; returns 5
add.__name__       # 'add' — thanks to functools.wraps (without it: 'wrapper')
```

The three moving parts: the decorator (`logged`), the wrapper (`wrapper`), and `functools.wraps`. Always use `wraps`; FastAPI, pytest and type checkers depend on the wrapped function looking like the original.

## 15.3 Timing — the same shape

```python
import time

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            log.info("%s took %.1f ms", func.__name__, (time.perf_counter() - start) * 1000)
    return wrapper

@timed
@logged                    # decorators stack bottom-up: timed(logged(fetch))
def fetch(url): ...
```

## 15.4 Decorators with arguments — a factory

`@retry(times=3)` is *called first* to produce the actual decorator. Three levels:

```python
def retry(times: int = 3, exceptions: tuple[type[BaseException], ...] = (Exception,), delay: float = 0.5):
    def decorator(func):                                       # level 2: the real decorator
        @functools.wraps(func)
        def wrapper(*args, **kwargs):                          # level 3: the wrapper
            last: BaseException | None = None
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last = e
                    log.warning("%s failed (attempt %d/%d): %s", func.__name__, attempt, times, e)
                    time.sleep(delay * attempt)
            raise RuntimeError(f"{func.__name__} failed after {times} attempts") from last
        return wrapper
    return decorator

@retry(times=5, exceptions=(httpx.TransportError,))
def call_model(prompt: str) -> str: ...
```

`@retry(times=5)` → `retry(times=5)` returns `decorator` → `decorator(call_model)` returns `wrapper` → `call_model = wrapper`. (In production, use the `tenacity` library, which is this idea with backoff/jitter done right.)

## 15.5 Authentication / authorisation — decorators that check context

```python
from contextvars import ContextVar

current_user: ContextVar[dict | None] = ContextVar("current_user", default=None)

def requires_role(role: str):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            user = current_user.get()
            if user is None:
                raise PermissionError("not authenticated")
            if role not in user["roles"]:
                raise PermissionError(f"requires role {role}")
            return func(*args, **kwargs)
        return wrapper
    return decorator

@requires_role("admin")
def delete_customer(customer_id: int) -> None: ...
```

This is `@PreAuthorize("hasRole('ADMIN')")` — minus the proxy, the aspect, and the container. In FastAPI you would express the same thing with a **dependency** (`Depends(require_admin)`) rather than a decorator, because dependencies integrate with request handling; the decorator form is what you use in plain services and CLIs.

## 15.6 Class decorators and decorators as *registration*

A decorator can also just *record* the function and return it unchanged. This is how routers, CLI frameworks, plugin registries and LLM tool registries work:

```python
routes: dict[str, Callable] = {}

def route(path: str):
    def register(func):
        routes[path] = func          # side effect: registration
        return func                  # unchanged
    return register

@route("/health")
def health(): return {"ok": True}

routes["/health"]()                  # {'ok': True}
```

`@app.get("/users")` in FastAPI, `@pytest.fixture`, `@tool` in LangChain, `@beta_tool` in the Anthropic SDK and `@app.command()` in Typer are all this pattern, sometimes combined with wrapping.

Decorators on **classes** receive the class and return a (possibly modified) class: `@dataclass` inspects annotations and *adds methods*; `@functools.total_ordering` fills in comparison methods; `@runtime_checkable` marks a Protocol.

## 15.7 Java comparison — annotations, AOP, proxies

| Java | Mechanism | Python |
|---|---|---|
| `@Override`, `@Deprecated` | compiler hints; metadata | type checker via `@override`, `warnings.deprecated` (3.13) — still functions |
| `@GetMapping("/x")` | metadata read by Spring at startup via reflection | `@app.get("/x")` — a function that registers the route *now* |
| `@Transactional`, `@Cacheable`, `@Retryable` | **proxy** generated by Spring AOP around the bean; needs a container; fails on self-invocation | a wrapper function; no container; works on any callable anywhere (`@lru_cache`, `@retry`) |
| `@Aspect` + pointcut | separate class + weaving | just apply the decorator where you want it (explicit) |
| Lombok `@Data` | annotation processor generating bytecode at compile time | `@dataclass` generating methods at runtime |
| `@Valid` | Bean Validation via reflection | Pydantic model type hints (no decorator needed) |

The essential difference: **a Java annotation is inert data; something else must read it. A Python decorator is executable code that transforms its target immediately.** That is why Python needs no container to make `@retry` work, why decorators compose by stacking, and why you can write your own in five lines.

## 15.8 Reading decorators in real code

```python
@app.post("/chat", response_model=ChatResponse)         # registration + metadata for OpenAPI
async def chat(req: ChatRequest, svc: Annotated[ChatService, Depends(get_chat_service)]): ...

@pytest.fixture
def client(): ...                                      # registration into pytest's DI registry

@functools.lru_cache(maxsize=1)
def get_settings() -> Settings: return Settings()       # memoised singleton — the Python "@Bean @Singleton"

@dataclass(frozen=True)
class Chunk: text: str; source: str                    # class decorator generating methods

@tool                                                   # LangChain: function → Tool with schema from hints + docstring
def search_docs(query: str) -> str:
    """Search the documentation for a query."""

@beta_tool                                              # Anthropic SDK: same idea
def get_weather(city: str) -> str:
    """Get the current weather for a city."""

@retry(stop=stop_after_attempt(3), wait=wait_exponential())   # tenacity
def call_bedrock(...): ...
```

When you see `@something`, ask: *does it wrap (change behaviour), register (record the function), or generate (add members)?* — and remember it ran at import time.

## Knowledge check — Chapter 15

1. Write out, without the `@` syntax, what `@timed def f(): ...` means. When does `timed` run?
2. What does `functools.wraps` do and what breaks if you omit it?
3. Explain why `@retry(times=3)` needs three nested functions while `@logged` needs two.
4. Contrast a Spring `@Cacheable` proxy with `@functools.lru_cache`: what does each need in order to work, and what is the well-known limitation of the Spring version that Python's does not have?
5. Give one example each of a decorator that wraps, one that registers, and one that generates.

---

# 16. Context Managers

Java has `try-with-resources` for `AutoCloseable`. Python has the `with` statement and the **context manager protocol** — and uses it for far more than closing: transactions, locks, temporary state, timers, mocks, and any "do something before, guarantee something after."

## 16.1 The statement

```python
with open("data.txt", encoding="utf-8") as f:
    data = f.read()
# f is closed here — whether the block finished, returned, or raised
```

is equivalent to:

```python
manager = open("data.txt", encoding="utf-8")
f = manager.__enter__()             # setup; the return value is bound to `as f`
try:
    data = f.read()
except BaseException as e:
    if not manager.__exit__(type(e), e, e.__traceback__):   # returning True would suppress the exception
        raise
else:
    manager.__exit__(None, None, None)
```

**Why does Python have it?** Because reference counting *usually* closes files promptly, but not reliably (cycles, exceptions holding frames, other implementations), and because "guarantee cleanup" is such a common need that `try/finally` everywhere was noise. `with` names the pattern.

**Java comparison.** `try (var f = new FileReader(p)) { ... }` is the same for the closing case. Differences: Python's protocol receives the *exception* in `__exit__` (so it can commit vs rollback), can *suppress* it, and any object can implement two methods to participate — you will use it for many non-resource things.

## 16.2 Where you will use it

```python
with open(path) as f: ...                                   # files
with httpx.Client() as client: ...                          # HTTP connection pools
with Session(engine) as session, session.begin(): ...       # DB session + transaction (commit on success, rollback on error)
with lock: ...                                              # threading.Lock — acquire/release
with tempfile.TemporaryDirectory() as tmp: ...              # created, then deleted
with pytest.raises(ValueError): ...                         # assert an exception is raised
with patch("app.services.boto3.client") as m: ...           # temporary mock
with client.messages.stream(...) as stream: ...             # Anthropic streaming — closes the HTTP stream
with torch.no_grad(): ...                                   # PyTorch: temporarily disable gradient tracking
with contextlib.suppress(FileNotFoundError): ...            # ignore a specific error
async with httpx.AsyncClient() as c: ...                    # async variant (__aenter__/__aexit__)
```

Multiple managers in one statement: `with open(a) as fa, open(b) as fb:` (3.10+ allows parenthesised multi-line form).

## 16.3 Writing your own — the class way

```python
class Timer:
    def __init__(self, label: str):
        self.label = label

    def __enter__(self):
        self.start = time.perf_counter()
        return self                                   # bound to `as t`

    def __exit__(self, exc_type, exc, tb):
        elapsed = (time.perf_counter() - self.start) * 1000
        status = "failed" if exc_type else "ok"
        log.info("%s: %.1f ms (%s)", self.label, elapsed, status)
        return False                                  # do not suppress exceptions

with Timer("embed batch") as t:
    embed(batch)
```

## 16.4 Writing your own — the generator way (preferred)

`contextlib.contextmanager` turns a generator into a context manager: code before `yield` is `__enter__`, the yielded value is `as x`, code after `yield` is `__exit__`. A `try/finally` around the `yield` guarantees cleanup; a `try/except` lets you react to the exception.

```python
from contextlib import contextmanager

@contextmanager
def timer(label: str):
    start = time.perf_counter()
    try:
        yield                                        # the with-body runs here
    finally:
        log.info("%s: %.1f ms", label, (time.perf_counter() - start) * 1000)

@contextmanager
def transaction(session):
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise

@contextmanager
def temp_env(**overrides):
    old = {k: os.environ.get(k) for k in overrides}
    os.environ.update(overrides)
    try:
        yield
    finally:
        for k, v in old.items():
            if v is None: os.environ.pop(k, None)
            else: os.environ[k] = v

with timer("load"), transaction(session) as s, temp_env(AWS_REGION="eu-west-1"):
    ...
```

This is the form you will write 90% of the time. It is also how pytest fixtures with teardown work (`yield` inside a fixture — Chapter 18) and how FastAPI's `lifespan` and dependency cleanup work (Chapter 20).

## 16.5 Async context managers

`async with` uses `__aenter__`/`__aexit__` (or `@contextlib.asynccontextmanager`) so setup/teardown can `await`. Async HTTP clients, async DB sessions and FastAPI's lifespan use it:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient()       # startup
    yield
    await app.state.http.aclose()              # shutdown

app = FastAPI(lifespan=lifespan)
```

## 16.6 `ExitStack` — dynamic numbers of managers

```python
from contextlib import ExitStack
with ExitStack() as stack:
    files = [stack.enter_context(open(p)) for p in paths]    # all closed on exit, in reverse order
```

## Knowledge check — Chapter 16

1. Write the `try/finally` expansion of `with x as y:`. What information does `__exit__` receive that Java's `close()` does not, and what can it do with it?
2. Why is `with` recommended even though CPython's reference counting usually closes files immediately?
3. Convert the `Timer` class into a `@contextmanager` generator. Where do `__enter__` and `__exit__` end up?
4. How is a pytest fixture with cleanup related to context managers?
5. Give three non-resource uses of `with` in real Python code.

---

# 17. Concurrency, Async Python and the GIL

For a Java backend developer this is the chapter most likely to overturn assumptions. In Java, "make it concurrent" means threads: a thread pool, `CompletableFuture`, now virtual threads — and they run in *parallel* on all cores. In CPython, threads exist and are useful, but **only one thread executes Python bytecode at a time** in the default build. The ecosystem therefore developed three distinct tools, and choosing correctly requires understanding *why* each exists.

## 17.1 Vocabulary first

- **Concurrency** — dealing with many things *in progress* at once (interleaved). A single core can be concurrent.
- **Parallelism** — doing many things *at the same instant* on multiple cores.
- **I/O-bound** — the task spends its time *waiting* (network, disk, database, an LLM API). The CPU is idle during the wait.
- **CPU-bound** — the task spends its time *computing* (parsing, hashing, matrix math in pure Python).

The questions that decide everything in Python: **is the work I/O-bound or CPU-bound? and if CPU-bound, does the heavy part run in Python or in C?**

## 17.2 The GIL

**What is it?** The **Global Interpreter Lock** is a mutex inside CPython that a thread must hold to execute Python bytecode. Multiple threads can exist and be scheduled by the OS, but only the one holding the GIL runs Python code; the others wait. The interpreter forces a switch every 5 ms (`sys.getswitchinterval()`) and — crucially — **a thread releases the GIL whenever it blocks on I/O** (socket read, file read, `time.sleep`, a DB driver waiting) and while inside many C extensions (NumPy math, hashing, compression, `psycopg` waiting on PostgreSQL).

**Why does it exist?** Chapter 1: CPython manages memory with **reference counts** — plain integers incremented/decremented constantly, by every assignment. Making every one of those atomic, or locking every object, would slow single-threaded code dramatically and complicate every C extension. One global lock was the pragmatic 1992 answer, and it made C extensions trivially thread-safe, which is a big part of why the C-extension ecosystem (NumPy etc.) flourished.

**What it means for threads:**

```text
CPU-bound pure Python, 4 threads, 4 cores:      ~1× speed (no gain, slight loss from contention)
I/O-bound, 4 threads (HTTP calls):               ~4× throughput (each releases the GIL while waiting)
NumPy matrix multiply, 4 threads:                gains (NumPy releases the GIL inside C; BLAS is multi-threaded anyway)
```

**Java comparison.** The JVM has no GIL: `ExecutorService` with 8 threads on 8 cores gives 8× on CPU-bound work. Python threads behave like Java threads *for I/O* and like a single core *for Python-level CPU work*.

**Free-threaded Python.** Since 3.13 CPython can be built without the GIL (PEP 703; officially supported as of 3.14, binary `python3.14t`, installable with `uv python install 3.14t`). It uses per-object locking and biased reference counting; single-threaded code pays a modest overhead, and many C extensions are still being made thread-safe. In 2026 it is a real option for CPU-parallel workloads, but **the default interpreter everyone deploys still has the GIL**, and the guidance below assumes that. Also note PEP 734 (3.14): multiple interpreters in one process via `concurrent.interpreters`, each with its own GIL — a middle path between threads and processes.

## 17.3 The three tools

| Tool | Parallel for CPU? | Good for | Cost | Java analogue |
|---|---|---|---|---|
| **Threads** (`threading`, `concurrent.futures.ThreadPoolExecutor`) | No (GIL) — unless the work is in C | I/O-bound work with *blocking* libraries (boto3, requests, psycopg sync, file I/O); moderate concurrency (tens–hundreds) | one OS thread each; shared memory; locks needed | `ExecutorService` with platform threads |
| **Processes** (`multiprocessing`, `ProcessPoolExecutor`) | Yes — each process has its own interpreter and GIL | CPU-bound *pure Python* work: parsing, feature engineering, simulations | process startup, data pickled between processes, no shared memory by default | forking workers / a job cluster |
| **asyncio** (`async`/`await`, event loop) | No — single thread | I/O-bound work with *async* libraries at high concurrency (thousands of sockets, streaming LLM responses, fan-out) | cooperative: one blocking call stalls everything; needs async libraries | reactive (WebFlux/Reactor) *or* virtual threads, in spirit |

## 17.4 Threads

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import httpx

def fetch(url: str) -> int:
    return httpx.get(url, timeout=10).status_code

urls = [f"https://example.com/{i}" for i in range(50)]

with ThreadPoolExecutor(max_workers=10) as pool:              # like Executors.newFixedThreadPool(10)
    futures = {pool.submit(fetch, u): u for u in urls}         # Future objects, like Java's
    for fut in as_completed(futures):
        print(futures[fut], fut.result())                       # result() re-raises exceptions from the worker

# Or, simpler, when order matters and you want all results:
with ThreadPoolExecutor(max_workers=10) as pool:
    codes = list(pool.map(fetch, urls))
```

This gives real ~10× speedup because `httpx.get` releases the GIL while waiting on the socket. **Threads are the pragmatic choice when the libraries you call are blocking** — which includes **boto3** (sync only), most DB drivers in sync mode, and `requests`. Shared state needs `threading.Lock` exactly as in Java; Python has no `synchronized`, so use `with lock:`. `queue.Queue` is the `BlockingQueue`.

Low-level `threading.Thread(target=fn).start()` exists; prefer the executor.

## 17.5 Processes

```python
from concurrent.futures import ProcessPoolExecutor

def heavy(chunk: list[str]) -> dict[str, int]:      # must be a top-level function (picklable)
    counts: dict[str, int] = {}
    for text in chunk:
        for word in text.split():
            counts[word] = counts.get(word, 0) + 1
    return counts

if __name__ == "__main__":                           # REQUIRED on macOS/Windows (spawn start method re-imports the module)
    with ProcessPoolExecutor() as pool:              # default: one process per core
        partials = pool.map(heavy, chunks)
    total = merge(partials)
```

Data crosses process boundaries by **pickling** (serialising) — so pass small inputs and return small outputs, not gigabyte arrays. For numeric work, NumPy/pandas/PyTorch already parallelise in C and often make multiprocessing unnecessary. Use processes for CPU-bound *Python* loops you cannot vectorise. In services, "process parallelism" usually means **running several server workers** (`fastapi run --workers 4` / Gunicorn) — the standard way a Python web service uses all cores.

## 17.6 asyncio: `async`/`await`, coroutines, the event loop

**What is it?** A single-threaded concurrency model where functions declared `async def` are **coroutines**: they can *suspend* at `await` points while waiting for I/O, letting an **event loop** run other coroutines in the meantime. Nothing runs in parallel; things *interleave* at explicit `await`s.

**Why does Python have it?** Because the GIL makes threads a poor fit for *massive* I/O concurrency (memory per thread, lock contention, no control over switching), and because generators (Chapter 14) already gave Python a way to suspend and resume a function — coroutines are that mechanism turned into a first-class feature (2015, PEP 492).

**How does it work?**

```python
import asyncio, httpx

async def fetch(client: httpx.AsyncClient, url: str) -> int:    # a coroutine function
    r = await client.get(url)          # suspend here; the loop runs others until the response arrives
    return r.status_code

async def main() -> None:
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(*(fetch(client, u) for u in urls))   # start all, wait for all
        print(results)

asyncio.run(main())                    # create an event loop, run main() to completion, close the loop
```

1. `fetch(client, url)` does **not** run the function; it creates a coroutine object (like calling a generator function).
2. `await coro` hands the coroutine to the loop and suspends the *current* coroutine until it finishes.
3. `asyncio.gather(...)` schedules many coroutines concurrently and collects results in order. `asyncio.TaskGroup` (3.11+) is the structured alternative with better error handling.
4. The **event loop** is a `while True` that keeps a queue of ready tasks and a set of I/O sockets it is waiting on (via `epoll`/`kqueue`); when a socket becomes readable, the loop resumes the task awaiting it.

**The cardinal rule:** everything inside the loop must be *non-blocking*. A single `time.sleep(1)`, `requests.get`, sync `boto3` call, or heavy CPU loop inside a coroutine **freezes every other coroutine** for that duration — there is only one thread. The escape hatch:

```python
result = await asyncio.to_thread(boto3_client.converse, **kwargs)   # run a blocking call in a worker thread
```

which is exactly what FastAPI does automatically for plain `def` endpoints (Chapter 20).

**Async toolbox:**

```python
await asyncio.sleep(1)                                     # non-blocking sleep
task = asyncio.create_task(fetch(c, u))                    # schedule in background, await later
async with asyncio.TaskGroup() as tg:                      # structured concurrency; cancels siblings on failure
    t1 = tg.create_task(fetch(c, a)); t2 = tg.create_task(fetch(c, b))
print(t1.result(), t2.result())
await asyncio.wait_for(fetch(c, u), timeout=5)             # timeout → asyncio.TimeoutError
async with asyncio.timeout(5): ...                         # 3.11+ block form
sem = asyncio.Semaphore(10)                                # limit concurrency (rate-limit LLM calls)
async with sem: await call_llm(...)
async for chunk in stream: ...                             # async iteration (streaming responses)
async def gen(): yield ...                                 # async generator
```

## 17.7 `async` does not mean "faster"

A single request handled by an `async def` endpoint is **not** faster than the sync version — it is the same I/O wait, plus a little event-loop overhead. What async buys is **throughput under many concurrent waits** with few threads. Concretely:

- 1 request calling the DB once: sync ≈ async.
- 1 request fanning out to 20 downstream calls: async (gather) ≈ 20× faster than sequential sync; a thread pool gets similar gains with more memory.
- 5,000 concurrent clients holding long-lived SSE/WebSocket connections streaming LLM tokens: async is the only practical option (5,000 threads is painful).
- A CPU-heavy transformation: async makes it *worse* (it blocks the loop); use processes or C-backed libraries.

Also: async code is *contagious*. Once your handler is `async`, every I/O library it touches must be async (httpx.AsyncClient, `asyncpg`/SQLAlchemy async, `aioboto3` instead of boto3, the async SDK clients). Mixing styles is the number-one source of "my FastAPI app is slow" reports.

## 17.8 Comparison with Java's models

| Java | Python | Comment |
|---|---|---|
| `Thread`, `ExecutorService` (platform threads) | `threading`, `ThreadPoolExecutor` | same API shape; Python's threads do not run Python code in parallel |
| `CompletableFuture` | `concurrent.futures.Future` (threads) / `asyncio.Future`/`Task` (asyncio) | `.then*` chains ↔ `await` in sequence |
| `parallelStream()`, ForkJoin for CPU | `ProcessPoolExecutor`, or NumPy/PyTorch vectorisation | Python cannot use threads for this in the default build |
| Virtual threads (Loom) | *closest:* asyncio — both give cheap concurrency for blocking-style code; but Loom keeps synchronous *syntax* and works with existing blocking libraries, asyncio requires `async/await` syntax and async libraries | Java 21's model is more convenient; Python's is more explicit |
| Reactive (WebFlux, Reactor, RxJava) | asyncio + async generators | asyncio is far less ceremony than Reactor: `await` instead of `flatMap` chains |
| `synchronized`, `ReentrantLock` | `threading.Lock` via `with lock:` | asyncio has `asyncio.Lock` for coroutine-level exclusion |
| `BlockingQueue` | `queue.Queue` (threads) / `asyncio.Queue` | |

The important reframe: in Java the *default* for a web server is one (virtual or platform) thread per request, using blocking libraries, and it parallelises across cores for free. In Python the default modern web server is **an async event loop per process × several processes** (uvicorn workers), and *you* decide per endpoint whether it is `async def` (non-blocking I/O) or `def` (runs in a thread pool).

## 17.9 Decision guide

```text
Is the work waiting on I/O?
├── yes → are the libraries async-capable (httpx, asyncpg, SQLAlchemy async, aioboto3, anthropic.AsyncAnthropic)?
│         ├── yes, and concurrency is high or you stream → asyncio
│         └── no (boto3, requests, sync DB drivers), or a small script → threads (ThreadPoolExecutor) or just sequential
└── no (CPU) → is the heavy part in C/NumPy/PyTorch?
              ├── yes → threads are fine, or just call it (it may already be multi-core)
              └── no  → ProcessPoolExecutor / multiple worker processes / rewrite the hot loop with NumPy
```

## 17.10 A small realistic example: fan-out to an LLM with a concurrency limit

```python
import asyncio
from anthropic import AsyncAnthropic

client = AsyncAnthropic()
limiter = asyncio.Semaphore(5)                        # at most 5 in flight → respects rate limits

async def summarise(text: str) -> str:
    async with limiter:
        resp = await client.messages.create(
            model="claude-opus-5", max_tokens=1024,
            messages=[{"role": "user", "content": f"Summarise in one sentence:\n\n{text}"}],
        )
        return resp.content[0].text

async def main(docs: list[str]) -> list[str]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(summarise(d)) for d in docs]
    return [t.result() for t in tasks]

summaries = asyncio.run(main(documents))
```

With 100 documents and ~3 s per call, this finishes in roughly 100/5 × 3 s = 60 s on one thread, versus 300 s sequentially. The Java version would be an `ExecutorService` with 5 threads or a `Semaphore` around virtual threads — same idea, different mechanics.

## Knowledge check — Chapter 17

1. What is the GIL, why does CPython have one, and what is the single most important fact about *when a thread releases it*?
2. A colleague parallelises a pure-Python CSV parser with `ThreadPoolExecutor(8)` and sees no speedup. Explain why, and name two fixes.
3. Explain why `await` is not "asynchronous execution in the Java sense". What actually happens to the calling coroutine?
4. What goes wrong if you call `boto3_client.invoke_model(...)` directly inside an `async def` endpoint, and what is the fix?
5. When is async *not* faster than sync? Give two concrete cases.
6. Map Java's virtual threads and Reactor onto Python's options. Which is closer to asyncio, and in what way is it different?

---

# Part IV — Backend engineering in Python

---

# 18. Testing with pytest

Java testing is JUnit 5 + Mockito + AssertJ + Spring's test slices. Python's professional standard is **pytest** — one tool that covers the runner, assertions, fixtures (dependency injection for tests), parameterisation, and a plugin ecosystem. The `unittest` module (xUnit-style, JUnit-like classes) ships with Python and you will see it in older code, but pytest runs those tests too, and new code is written in pytest style.

## 18.1 The shape of a test

```python
# tests/test_pricing.py
from customer_api.pricing import apply_discount

def test_discount_is_applied():
    assert apply_discount(100.0, 10) == 90.0

def test_none_discount_returns_price():
    assert apply_discount(100.0, None) == 100.0
```

```bash
uv run pytest                 # discovers tests/test_*.py, functions named test_*
uv run pytest -x -q           # stop at first failure, quiet
uv run pytest tests/test_pricing.py::test_discount_is_applied
uv run pytest -k "discount"   # name filter
uv run pytest -vv --tb=short
```

Differences from JUnit:

- **Tests are plain functions** (classes are optional, used only for grouping). No `@Test` annotation; the `test_` prefix is the marker.
- **Plain `assert`** — no `assertEquals`, no AssertJ. pytest rewrites `assert` statements at import time so a failure prints both sides in detail:

  ```text
  >       assert apply_discount(100.0, 10) == 95.0
  E       assert 90.0 == 95.0
  E        +  where 90.0 = apply_discount(100.0, 10)
  ```
- **No compile step**: a typo in a test is discovered when the test runs. Run the type checker on `tests/` too.

## 18.2 Fixtures — pytest's dependency injection

**What is it?** A fixture is a function decorated `@pytest.fixture` whose *return value is injected* into any test (or other fixture) that names it as a parameter. It replaces `@BeforeEach`, `@BeforeAll`, test-scoped Spring beans, and most test base classes.

```python
# tests/conftest.py  ← fixtures here are available to every test in the directory tree (no import needed)
import pytest
from customer_api.repositories import InMemoryCustomerRepository
from customer_api.services import CustomerService

@pytest.fixture
def repo():
    return InMemoryCustomerRepository()

@pytest.fixture
def service(repo):                     # fixtures can depend on fixtures
    return CustomerService(repo)

@pytest.fixture
def tmp_config(tmp_path):              # tmp_path is a built-in fixture: a fresh temporary directory
    cfg = tmp_path / "cfg.json"
    cfg.write_text('{"debug": true}')
    return cfg
```

```python
# tests/test_customer_service.py
def test_create_customer(service, repo):          # ask for what you need by name
    created = service.create(name="Ada", email="ada@x.io")
    assert repo.get(created.id) == created
```

**Setup + teardown with `yield`** (Chapter 16 — a fixture is a context manager):

```python
@pytest.fixture
def db_session(engine):
    connection = engine.connect()
    txn = connection.begin()
    session = Session(bind=connection)
    yield session                                  # ← test runs here
    session.close(); txn.rollback(); connection.close()    # every test sees a clean DB
```

**Scopes**: `@pytest.fixture(scope="session")` runs once per test run (like `@BeforeAll` across the suite) — use for expensive things (a database container, a loaded ML model). Default `function` scope runs per test. `autouse=True` applies without being requested.

**Built-in fixtures** you will use: `tmp_path` (temp dir), `monkeypatch` (temporarily set env vars / attributes, auto-restored), `capsys` (captured stdout/stderr), `caplog` (captured log records), `request` (metadata).

## 18.3 Parameterised tests

```python
import pytest

@pytest.mark.parametrize(
    ("price", "pct", "expected"),
    [
        (100.0, 10, 90.0),
        (100.0, 0, 100.0),
        (100.0, None, 100.0),
        pytest.param(100.0, 150, None, marks=pytest.mark.xfail(raises=ValueError)),
    ],
    ids=["ten-percent", "zero", "none", "over-100"],
)
def test_apply_discount(price, pct, expected):
    assert apply_discount(price, pct) == expected
```

Same as JUnit's `@ParameterizedTest` + `@CsvSource`, but data is plain Python — you can generate cases with a comprehension. Fixtures can also be parameterised (`@pytest.fixture(params=[...])`) to run every dependent test against each variant (e.g. against SQLite and PostgreSQL).

## 18.4 Exceptions, markers, and skipping

```python
def test_rejects_negative():
    with pytest.raises(ValueError, match="negative"):
        apply_discount(100.0, -5)

@pytest.mark.slow                              # custom marker; register in pyproject [tool.pytest.ini_options] markers
def test_full_pipeline(): ...

@pytest.mark.skipif(sys.platform == "win32", reason="POSIX only")
def test_permissions(): ...

# run: uv run pytest -m "not slow"
```

## 18.5 Mocking

Python mocks are easier than Mockito because of duck typing: **any object with the right attributes is a valid substitute**, and modules are mutable so you can swap what a name points to. Three levels:

**1. Hand-written fakes** (preferred when a class has a clear interface):

```python
class FakeLLM:
    def __init__(self, reply: str): self.reply, self.calls = reply, []
    def complete(self, prompt: str) -> str:
        self.calls.append(prompt); return self.reply

def test_chat_service_uses_llm():
    llm = FakeLLM("Hi there")
    svc = ChatService(llm)
    assert svc.ask("hello") == "Hi there"
    assert llm.calls == ["hello"]
```

**2. `unittest.mock` / `pytest-mock`** — `Mock`, `MagicMock`, `patch` (Mockito's role):

```python
from unittest.mock import MagicMock, patch

def test_service_calls_repo():
    repo = MagicMock()
    repo.get.return_value = Customer(id=1, name="Ada", email="ada@x.io")
    svc = CustomerService(repo)
    assert svc.get(1).name == "Ada"
    repo.get.assert_called_once_with(1)

def test_bedrock_call(mocker):                                    # pytest-mock's fixture
    fake = mocker.patch("customer_api.ai.boto3.client")            # patch where it is *looked up*, not where it is defined
    fake.return_value.converse.return_value = {"output": {"message": {"content": [{"text": "ok"}]}}}
    assert ask_model("hi") == "ok"
```

The one rule that trips everyone: **patch the name in the module under test** (`customer_api.ai.boto3.client`), because that module already did `import boto3` and holds its own reference. Patching `boto3.client` globally works only if the module calls `boto3.client` via the `boto3` module attribute at call time.

**3. Library-specific test helpers**: `httpx.MockTransport` / `respx` for HTTP, `moto` for AWS (starts fake S3/DynamoDB/Bedrock-adjacent services in-process — `@mock_aws`), FastAPI's `app.dependency_overrides` for DI (Chapter 20), `freezegun`/`time-machine` for time.

**`monkeypatch`** for env and attributes without `patch`'s ceremony:

```python
def test_reads_region(monkeypatch):
    monkeypatch.setenv("AWS_REGION", "eu-west-1")
    assert Settings().aws_region == "eu-west-1"
```

## 18.6 Async tests

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"          # pytest-asyncio: async def tests just work
```

```python
async def test_fetch(httpx_mock):           # with pytest-httpx, or use respx
    ...
```

## 18.7 Integration tests

Python integration tests usually run against **real infrastructure in Docker** — Testcontainers exists for Python too:

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine

@pytest.fixture(scope="session")
def pg_url():
    with PostgresContainer("postgres:17") as pg:
        yield pg.get_connection_url().replace("psycopg2", "psycopg")

@pytest.fixture(scope="session")
def engine(pg_url):
    engine = create_engine(pg_url)
    Base.metadata.create_all(engine)          # or run Alembic migrations
    return engine
```

The API-level test uses FastAPI's `TestClient` (built on httpx) — an in-process HTTP client, so no server is started (like `MockMvc`/`WebTestClient` but exercising the real ASGI app):

```python
from fastapi.testclient import TestClient
from customer_api.main import app
from customer_api.db import get_session

@pytest.fixture
def client(db_session):
    app.dependency_overrides[get_session] = lambda: db_session      # swap the DI provider (like @MockBean/@TestConfiguration)
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()

def test_create_and_get_customer(client):
    r = client.post("/customers", json={"name": "Ada", "email": "ada@x.io"})
    assert r.status_code == 201
    cid = r.json()["id"]
    assert client.get(f"/customers/{cid}").json()["name"] == "Ada"

def test_validation_error(client):
    r = client.post("/customers", json={"name": "", "email": "nope"})
    assert r.status_code == 422                     # FastAPI's validation status (not 400)
    assert {e["loc"][-1] for e in r.json()["detail"]} == {"name", "email"}
```

## 18.8 pytest vs JUnit — summary

| JUnit / Spring Test | pytest |
|---|---|
| `@Test` methods in classes | `test_*` functions (classes optional) |
| `assertEquals`, AssertJ | `assert` with introspection |
| `@BeforeEach` / `@AfterEach` | fixture with `yield` |
| `@BeforeAll` | `scope="session"`/`"module"` fixture |
| `@ParameterizedTest` | `@pytest.mark.parametrize` |
| `assertThrows` | `with pytest.raises(...)` |
| `@Tag`, `@Disabled` | `@pytest.mark.x`, `@pytest.mark.skip` |
| Mockito `mock()`, `when().thenReturn()` | `MagicMock()`, `.return_value` / `.side_effect` |
| `@MockBean` | `app.dependency_overrides` / `monkeypatch` / hand-written fake |
| `@SpringBootTest` + `MockMvc` | `TestClient(app)` |
| Testcontainers | `testcontainers` (Python) |
| Surefire / Gradle test task | `pytest` (+ `pytest-cov` for coverage, `pytest-xdist` for parallel) |

Keep tests in `tests/`, mirror the package layout loosely, share fixtures in `conftest.py`, prefer fakes over mocks for your own interfaces, and run pytest in CI with `--cov`.

## Knowledge check — Chapter 18

1. How does pytest make plain `assert` produce useful failure messages, and why does that matter compared to `assertEquals`?
2. Explain fixtures as dependency injection. How does a fixture with `yield` relate to `@BeforeEach`/`@AfterEach` and to context managers?
3. Why must you `patch("customer_api.ai.boto3.client")` rather than `patch("boto3.client")`?
4. Why are hand-written fakes so cheap in Python compared to Java, and when would you still use `MagicMock`?
5. What does `TestClient(app)` do that `MockMvc` also does, and what does `app.dependency_overrides` replace from Spring's test toolkit?

---

# 19. Logging and Configuration

## 19.1 Python logging

**What is it?** The standard library `logging` module — a hierarchical logger tree with handlers, formatters and levels, designed after log4j. If you know Logback/SLF4J you know the model:

```text
Java:   Logger (per class name) → Appender → Layout → level per logger, root logger
Python: Logger (per module name) → Handler  → Formatter → level per logger, root logger
```

```python
import logging

log = logging.getLogger(__name__)          # module-qualified name, e.g. "customer_api.services.customer_service"
                                           # — the exact analogue of LoggerFactory.getLogger(MyClass.class)

def create(name: str) -> Customer:
    log.info("creating customer name=%s", name)        # %-style lazy formatting: args are only formatted if emitted
    try:
        ...
    except Exception:
        log.exception("failed to create customer")     # logs at ERROR with the traceback attached
        raise
```

Levels: `DEBUG < INFO < WARNING < ERROR < CRITICAL`. The root logger defaults to `WARNING` with a handler that prints to stderr — so **`log.info(...)` prints nothing until you configure logging**. That surprises everyone once.

**Why `%s` and not f-strings?** `log.debug(f"payload={big_json}")` formats the string even if DEBUG is off. `log.debug("payload=%s", big_json)` defers formatting until needed. Ruff flags f-strings in logging calls for this reason.

**Configuration — do it once, at the entry point** (`main.py` or the app factory), never in library modules:

```python
import logging.config

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {"format": "%(asctime)s %(levelname)s %(name)s [%(process)d] %(message)s"},
        "json": {"()": "pythonjsonlogger.json.JsonFormatter",                      # uv add python-json-logger
                 "fmt": "%(asctime)s %(levelname)s %(name)s %(message)s"},
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "json", "stream": "ext://sys.stdout"},
    },
    "root": {"level": "INFO", "handlers": ["console"]},
    "loggers": {
        "customer_api": {"level": "DEBUG"},           # per-package level, like <logger name="com.acme" level="DEBUG"/>
        "sqlalchemy.engine": {"level": "WARNING"},    # set to INFO to see SQL
        "httpx": {"level": "WARNING"},
        "botocore": {"level": "WARNING"},
    },
}
logging.config.dictConfig(LOGGING)
```

For a quick script: `logging.basicConfig(level=logging.INFO, format="%(levelname)s %(name)s: %(message)s")`.

**Structured logging** for production (JSON lines to stdout, picked up by CloudWatch/Loki) is either the JSON formatter above or **structlog**, which also gives context binding (`log = log.bind(request_id=...)`). Request-scoped context (request id, user id) travels through `contextvars.ContextVar` — the async-safe equivalent of `ThreadLocal`/MDC — and a logging `Filter` injects it into every record. Uvicorn/FastAPI access logs are separate loggers (`uvicorn.access`) you configure in the same dict.

**Third-party loggers** are noisy by default at DEBUG; set `botocore`, `urllib3`, `httpx`, `httpcore` to WARNING as above.

## 19.2 Configuration

**How Python apps are normally configured:** environment variables as the source of truth (12-factor), a `.env` file for local development, and a typed settings object built once at startup. There is no `application.yml` convention; there are no profiles as a first-class feature. That is simpler than Spring and covers 95% of services.

```text
Spring Boot                                   Python
application.yml + application-{profile}.yml   environment variables (+ .env locally)
@ConfigurationProperties / @Value              pydantic-settings BaseSettings
spring.profiles.active=prod                    APP_ENV=prod (a plain variable you branch on, if at all)
Spring Cloud Config / AWS Parameter Store      read Parameter Store/Secrets Manager with boto3 at startup, or inject as env vars by the platform (ECS/Lambda/K8s)
```

```python
# customer_api/config.py
from functools import lru_cache
from pydantic import PostgresDsn, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",              # loaded only if present; real env vars override it
        env_prefix="APP_",            # APP_DATABASE_URL, APP_LOG_LEVEL...
        env_nested_delimiter="__",    # APP_BEDROCK__MODEL_ID → bedrock.model_id
        extra="ignore",
    )
    env: str = "dev"
    log_level: str = "INFO"
    database_url: PostgresDsn
    api_key: SecretStr                # repr shows '**********'; .get_secret_value() to read — avoids leaking in logs
    aws_region: str = "us-east-1"
    bedrock_model_id: str = "amazon.nova-pro-v1:0"

@lru_cache(maxsize=1)
def get_settings() -> Settings:       # one instance, built lazily (not at import time → tests can set env first)
    return Settings()
```

```bash
# .env  (git-ignored; committed as .env.example with placeholders)
APP_DATABASE_URL=postgresql+psycopg://app:app@localhost:5432/customers
APP_API_KEY=dev-key
```

Usage: `settings = get_settings()` where needed, or as a FastAPI dependency (`Depends(get_settings)`) so tests can override it. Validation errors at startup (missing `APP_DATABASE_URL`) fail fast with a clear message — the same behaviour as a failed `@ConfigurationProperties` binding.

**Secrets.** Never in code, never in `.env` that is committed. In AWS: inject via the task/function definition from **Secrets Manager** / **SSM Parameter Store**, or fetch at startup:

```python
import boto3, json
def load_secret(name: str) -> dict:
    sm = boto3.client("secretsmanager")
    return json.loads(sm.get_secret_value(SecretId=name)["SecretString"])
```

Use `SecretStr` for anything sensitive so `print(settings)` and log records do not leak it.

**Environments/profiles.** Instead of `application-prod.yml`, keep one `Settings` class with sane defaults and let each environment set its variables. If behaviour must differ (e.g. disable docs in prod), branch on `settings.env == "prod"` in the app factory. Avoid recreating Spring's profile machinery.

## Knowledge check — Chapter 19

1. Why does `logging.getLogger(__name__).info("x")` print nothing in a fresh script, and where should the fix live?
2. Why do Python style guides prefer `log.info("id=%s", id)` over `log.info(f"id={id}")`?
3. Map `application.yml`, `@ConfigurationProperties`, profiles and MDC to their Python counterparts.
4. Why build `Settings` lazily behind `get_settings()` rather than as `settings = Settings()` at module import?
5. What does `SecretStr` protect against, and what does it *not* protect against?

---

# 20. Python Web Frameworks (FastAPI focus)

## 20.1 The landscape

| Framework | Style | Best for | Spring analogue |
|---|---|---|---|
| **Django** | "batteries included": ORM, admin UI, auth, migrations, templates, forms | content sites, admin-heavy apps, monoliths where the built-in ORM/admin pays off | the full Spring platform (Boot + Data + Security + Thymeleaf) |
| **Flask** | micro-framework: routing + request/response, everything else via extensions | small services, legacy APIs, teams that want to pick each piece | Spring MVC alone, no Boot |
| **FastAPI** | modern, async-capable, type-hint driven: Pydantic validation, DI, automatic OpenAPI | **JSON APIs, AI/ML services, anything new** | Spring Boot + Spring MVC/WebFlux + Bean Validation + springdoc, with less code |

Django REST Framework (DRF) and Django Ninja turn Django into an API backend; Litestar and Starlette (FastAPI's foundation) are alternatives you may meet. **This guide uses FastAPI** because it is the standard for Python AI services and its type-hint-first design lets you reuse everything from Chapters 8–9.

**How Python servers run.** Java has the Servlet API and an embedded Tomcat; Python has **ASGI** (async) and **WSGI** (sync, older — Flask/Django classic) as the interface between server and app. FastAPI is an ASGI app; **uvicorn** is the ASGI server (Tomcat's role). In production you run several uvicorn worker processes (`fastapi run --workers 4`, or Gunicorn with uvicorn workers) to use all cores — recall Chapter 17.

## 20.2 The smallest FastAPI app

```python
# app/main.py
from fastapi import FastAPI

app = FastAPI(title="Customer API", version="1.0.0")

@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

```bash
uv add "fastapi[standard]"
uv run fastapi dev app/main.py        # dev server with auto-reload at http://127.0.0.1:8000
# OpenAPI UI: http://127.0.0.1:8000/docs   (Swagger UI)  and /redoc ;  raw schema: /openapi.json
```

Yes — the interactive docs are already there. That is the Spring equivalent of adding springdoc, except it is generated from the same type hints that do validation.

## 20.3 Routes, path/query/body parameters

```python
from typing import Annotated
from fastapi import FastAPI, Query, Path, HTTPException, status
from pydantic import BaseModel, Field

class CreateCustomer(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: str

class CustomerOut(BaseModel):
    id: int
    name: str
    email: str

@app.get("/customers/{customer_id}", response_model=CustomerOut)
def get_customer(customer_id: Annotated[int, Path(gt=0)]):                # path param, validated int > 0
    customer = repo.get(customer_id)
    if customer is None:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="customer not found")
    return customer                                                        # dict/ORM/model → serialised via response_model

@app.get("/customers", response_model=list[CustomerOut])
def list_customers(
    q: Annotated[str | None, Query(max_length=50)] = None,                 # ?q=...
    limit: Annotated[int, Query(ge=1, le=100)] = 20,                        # ?limit=...
):
    return repo.search(q, limit)

@app.post("/customers", response_model=CustomerOut, status_code=status.HTTP_201_CREATED)
def create_customer(payload: CreateCustomer):                              # body: Pydantic model → JSON body
    return repo.add(payload)
```

How FastAPI decides where a parameter comes from (no annotations needed in the common case):

| Parameter looks like | Source |
|---|---|
| name appears in the path `{customer_id}` | path |
| simple type (`int`, `str`, `bool`, `float`, `UUID`, `datetime`) | query string |
| Pydantic model | JSON body |
| `Annotated[..., Header()]` / `Cookie()` / `Form()` / `File()` | explicit |
| `Request`, `Response`, `BackgroundTasks` | injected framework objects |

**Java comparison.**

```java
@GetMapping("/customers/{id}")
public CustomerDto get(@PathVariable @Positive long id) { ... }

@PostMapping("/customers") @ResponseStatus(CREATED)
public CustomerDto create(@RequestBody @Valid CreateCustomerRequest req) { ... }
```

Same information; FastAPI derives `@PathVariable` / `@RequestParam` / `@RequestBody` / `@Valid` from *types* rather than annotations, and the DTO is a Pydantic model. Validation failures return **422 Unprocessable Content** with a structured `detail` list (not 400 as in Spring by default) — remember that in tests.

## 20.4 Response models and status codes

`response_model=CustomerOut` (or a return annotation `-> CustomerOut`) filters and validates the output — an ORM object with 30 fields becomes a 3-field JSON document. This is the DTO-projection role of MapStruct/`@JsonView`. Other tools: `status_code=`, returning `JSONResponse(...)` directly, `Response` for headers, `StreamingResponse` for streams, `RedirectResponse`.

## 20.5 Dependency injection — `Depends`

**What is it?** FastAPI's DI is **per-request, function-based**: a dependency is any callable (function, class, generator); FastAPI calls it for each request, passing *its* dependencies recursively, and injects the result. There is no application-wide container of singleton beans; long-lived objects are created in `lifespan` or cached functions and *exposed* through dependencies.

```python
from collections.abc import Iterator
from fastapi import Depends
from sqlalchemy.orm import Session

def get_session() -> Iterator[Session]:                    # generator dependency: setup / yield / teardown per request
    with SessionLocal() as session:
        yield session

def get_customer_service(session: Annotated[Session, Depends(get_session)]) -> CustomerService:
    return CustomerService(CustomerRepository(session))

SessionDep = Annotated[Session, Depends(get_session)]        # reusable alias — idiomatic
ServiceDep = Annotated[CustomerService, Depends(get_customer_service)]

@app.get("/customers/{customer_id}")
def get_customer(customer_id: int, service: ServiceDep):
    ...

# Auth as a dependency — the FastAPI way to do @PreAuthorize
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
bearer = HTTPBearer()
def current_user(creds: Annotated[HTTPAuthorizationCredentials, Depends(bearer)]) -> User:
    return verify_jwt(creds.credentials)          # raise HTTPException(401) on failure
def require_admin(user: Annotated[User, Depends(current_user)]) -> User:
    if "admin" not in user.roles: raise HTTPException(403)
    return user

@app.delete("/customers/{customer_id}", dependencies=[Depends(require_admin)])   # dependency for its side effect only
def delete_customer(customer_id: int, service: ServiceDep): ...
```

**Compared to Spring's DI:**

| Spring | FastAPI |
|---|---|
| container-managed singletons, wired at startup | callables invoked per request; singletons are your `lru_cache`/lifespan objects |
| `@Autowired` by type | `Depends(fn)` by function (explicit provider) |
| `@Scope("request")` | the default |
| `@PreAuthorize`, filters | dependencies that raise `HTTPException` |
| `@MockBean` in tests | `app.dependency_overrides[fn] = fake` |
| interceptors on beans | none needed — the dependency *is* the code |

The result: DI without a container, fully explicit, trivially testable, and `Depends` doubles as the place for auth, pagination parsing, tenant resolution and rate limiting.

## 20.6 Async endpoints — `def` vs `async def`

```python
@app.get("/sync")
def sync_endpoint():                 # runs in a threadpool → blocking libraries (boto3, sync SQLAlchemy) are fine
    return boto3_client.list_buckets()

@app.get("/async")
async def async_endpoint():          # runs on the event loop → must only await non-blocking I/O
    async with httpx.AsyncClient() as c:
        r = await c.get("https://api.example.com")
    return r.json()
```

Rule (from Chapter 17): use `async def` when everything you call inside is async (httpx.AsyncClient, `AsyncAnthropic`, SQLAlchemy async, `aioboto3`); use plain `def` when you call blocking libraries — FastAPI runs it in a worker thread so it does not stall the loop. **Do not put blocking calls in `async def`.** Mixed apps are normal: sync CRUD endpoints, async streaming endpoints.

## 20.7 Validation, error handling, `@ControllerAdvice`

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class NotFoundError(Exception): ...
class ConflictError(Exception): ...

@app.exception_handler(NotFoundError)                         # global mapping domain error → HTTP
async def not_found_handler(request: Request, exc: NotFoundError):
    return JSONResponse(status_code=404, content={"detail": str(exc)})

@app.exception_handler(ConflictError)
async def conflict_handler(request: Request, exc: ConflictError):
    return JSONResponse(status_code=409, content={"detail": str(exc)})
```

Pydantic `ValidationError` on request bodies is handled by FastAPI automatically (422). Unhandled exceptions become 500 with a server-side traceback.

## 20.8 Middleware, lifespan, routers, background tasks

```python
from contextlib import asynccontextmanager
from fastapi import APIRouter, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
import time, uuid

@asynccontextmanager
async def lifespan(app: FastAPI):                     # startup/shutdown (ApplicationRunner + @PreDestroy)
    app.state.bedrock = boto3.client("bedrock-runtime")
    yield
    # close pools here

app = FastAPI(lifespan=lifespan)

app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

@app.middleware("http")                               # a servlet Filter; runs for every request
async def request_id_and_timing(request: Request, call_next):
    rid = request.headers.get("x-request-id", str(uuid.uuid4()))
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["x-request-id"] = rid
    response.headers["x-response-time-ms"] = f"{(time.perf_counter() - start) * 1000:.1f}"
    return response

router = APIRouter(prefix="/customers", tags=["customers"])   # group routes per module (a @RestController class)
@router.get("/{customer_id}")
def get_customer(...): ...
app.include_router(router)

@app.post("/customers/{customer_id}/welcome")
def send_welcome(customer_id: int, tasks: BackgroundTasks):
    tasks.add_task(send_email, customer_id)           # runs after the response is sent (light @Async); use Celery/SQS for real jobs
    return {"queued": True}
```

## 20.9 FastAPI vs Spring Boot — the mental map

```text
Spring Boot                                    FastAPI
────────────────────────────────────────────────────────────────────────
Embedded Tomcat (Servlet)                      uvicorn (ASGI)
@RestController + @RequestMapping              APIRouter + @router.get/post
@PathVariable / @RequestParam / @RequestBody   derived from parameter types (+ Path/Query/Body for constraints)
DTO + @Valid + Bean Validation                 Pydantic model
@ResponseStatus / ResponseEntity               status_code= / Response / JSONResponse
@ControllerAdvice + @ExceptionHandler          @app.exception_handler
@Autowired / constructor injection             Depends()
@Configuration + @Bean singletons              lifespan + app.state / lru_cache'd factories
Filter / HandlerInterceptor                    middleware
springdoc-openapi                              built in (/docs, /openapi.json)
Spring Security filter chain                   security dependencies (HTTPBearer, OAuth2PasswordBearer) + your checks
@Async / @Scheduled                            BackgroundTasks / external scheduler (APScheduler, Celery, EventBridge)
WebFlux for reactive                           async def endpoints — same framework
MockMvc                                        TestClient
application.yml                                pydantic-settings
```

FastAPI apps are typically **a fraction of the size** of the Spring equivalent because validation, docs, serialisation and DI all derive from the same type hints, and because the framework does not need a container. The trade-off: fewer conventions, so architecture is yours to decide (Chapter 24).

## Knowledge check — Chapter 20

1. How does FastAPI decide whether a parameter comes from the path, query string or body?
2. What HTTP status does FastAPI return for a body that fails validation, and what does the response body look like?
3. Explain FastAPI's `Depends` to a Spring developer: what is the scope, who creates the objects, and where do singletons live?
4. When should an endpoint be `async def` and when plain `def`? What happens if you get it wrong in each direction?
5. What are ASGI and uvicorn in Servlet/Tomcat terms, and how does a FastAPI service use all CPU cores?

---

# 21. FastAPI Project — Customer API

A complete, small, production-shaped backend: REST endpoints, Pydantic models, validation, a service layer, a repository layer, PostgreSQL via SQLAlchemy, and pytest. The point is to see every previous chapter land in one codebase — and to see how a Python service is *shaped differently* from the Spring version.

## 21.1 Why the architecture is smaller than Spring's

A Java developer's reflex is: `Controller → Service → Repository (interface) → JpaRepository`, a DTO per direction, a mapper, an exception hierarchy per layer, and configuration classes. Python projects usually look flatter, for concrete reasons:

- **No container, no interfaces required.** A repository is a class you instantiate with a session; the "interface" is a `Protocol` if and only if you need the checker's help.
- **Pydantic + FastAPI collapse three layers** (DTO, validation, serialisation) into one class per direction.
- **Modules are namespaces**, so `schemas.py`, `models.py`, `repository.py`, `service.py`, `api.py` — one file each — *is* the layering. One class per file is not a Python norm.
- **Functions are first-class**: some "services" are functions, some "repositories" are a handful of functions over a session. A class is added when there is state or multiple related operations.

Keep the *ideas* (thin HTTP layer, business logic isolated from HTTP and from SQL, persistence behind a narrow surface) and drop the *ceremony*. A 5-endpoint service should be ~300 lines, not 30 files.

## 21.2 Layout

```text
customer-api/
├── pyproject.toml
├── .env.example
├── alembic.ini
├── alembic/                     # migrations (Chapter 22)
├── src/
│   └── customer_api/
│       ├── __init__.py
│       ├── main.py              # app factory, routers, handlers, lifespan
│       ├── config.py            # Settings
│       ├── db.py                # engine, session factory, get_session dependency
│       ├── models.py            # SQLAlchemy ORM entities
│       ├── schemas.py           # Pydantic request/response models
│       ├── repository.py        # data access
│       ├── service.py           # business rules
│       ├── errors.py            # domain exceptions
│       └── api/
│           ├── __init__.py
│           └── customers.py     # router
└── tests/
    ├── conftest.py
    ├── test_service.py          # unit tests with a fake repository
    └── test_api.py              # API tests against PostgreSQL (Testcontainers)
```

```toml
# pyproject.toml (relevant parts)
[project]
name = "customer-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.141",
    "sqlalchemy>=2.0,<3",
    "psycopg[binary]>=3.2",
    "alembic>=1.16",
    "pydantic[email]>=2.13",
    "pydantic-settings>=2.6",
]
[dependency-groups]
dev = ["pytest>=9", "pytest-cov", "testcontainers[postgres]", "ruff", "mypy"]
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
[tool.pytest.ini_options]
testpaths = ["tests"]
```

## 21.3 Configuration and database plumbing

```python
# src/customer_api/config.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_", extra="ignore")
    database_url: str = "postgresql+psycopg://app:app@localhost:5432/customers"
    echo_sql: bool = False

@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

```python
# src/customer_api/db.py
from collections.abc import Iterator
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker
from customer_api.config import get_settings

settings = get_settings()
engine = create_engine(settings.database_url, echo=settings.echo_sql, pool_pre_ping=True)   # ≈ DataSource + pool (HikariCP role)
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)                            # ≈ EntityManagerFactory

def get_session() -> Iterator[Session]:
    """FastAPI dependency: one session per request, closed afterwards."""
    with SessionLocal() as session:
        yield session
```

## 21.4 Entities and schemas

```python
# src/customer_api/models.py
from datetime import datetime
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class Customer(Base):                                   # ≈ @Entity
    __tablename__ = "customers"

    id: Mapped[int] = mapped_column(primary_key=True)                     # ≈ @Id @GeneratedValue
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

```python
# src/customer_api/schemas.py
from datetime import datetime
from pydantic import BaseModel, ConfigDict, EmailStr, Field

class CustomerCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr

class CustomerUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=1, max_length=100)
    email: EmailStr | None = None

class CustomerOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)     # build from ORM objects
    id: int
    name: str
    email: EmailStr
    created_at: datetime
```

Note the split: **`models.py` = database shape** (SQLAlchemy), **`schemas.py` = API shape** (Pydantic). Keeping them separate is the one piece of "Java discipline" worth importing — it lets the API evolve independently of the table and avoids leaking columns.

## 21.5 Repository and service

```python
# src/customer_api/errors.py
class NotFoundError(Exception):
    def __init__(self, entity: str, key: object):
        super().__init__(f"{entity} {key!r} not found")

class ConflictError(Exception):
    pass
```

```python
# src/customer_api/repository.py
from collections.abc import Sequence
from sqlalchemy import select
from sqlalchemy.orm import Session
from customer_api.models import Customer

class CustomerRepository:
    """All SQL for customers lives here. Nothing else in the app writes queries."""

    def __init__(self, session: Session):
        self.session = session

    def get(self, customer_id: int) -> Customer | None:
        return self.session.get(Customer, customer_id)                 # ≈ em.find()

    def get_by_email(self, email: str) -> Customer | None:
        return self.session.scalar(select(Customer).where(Customer.email == email))

    def search(self, q: str | None, limit: int, offset: int) -> Sequence[Customer]:
        stmt = select(Customer).order_by(Customer.id).limit(limit).offset(offset)
        if q:
            stmt = stmt.where(Customer.name.ilike(f"%{q}%"))
        return self.session.scalars(stmt).all()

    def add(self, customer: Customer) -> Customer:
        self.session.add(customer)
        self.session.flush()                                            # get the id now; commit is the service's call
        return customer

    def delete(self, customer: Customer) -> None:
        self.session.delete(customer)
```

```python
# src/customer_api/service.py
from customer_api.errors import ConflictError, NotFoundError
from customer_api.models import Customer
from customer_api.repository import CustomerRepository
from customer_api.schemas import CustomerCreate, CustomerUpdate

class CustomerService:
    def __init__(self, repo: CustomerRepository):
        self.repo = repo

    def get(self, customer_id: int) -> Customer:
        customer = self.repo.get(customer_id)
        if customer is None:
            raise NotFoundError("customer", customer_id)
        return customer

    def search(self, q: str | None = None, limit: int = 20, offset: int = 0):
        return self.repo.search(q, limit, offset)

    def create(self, data: CustomerCreate) -> Customer:
        if self.repo.get_by_email(data.email) is not None:
            raise ConflictError(f"email {data.email} already registered")
        customer = self.repo.add(Customer(name=data.name, email=data.email))
        self.repo.session.commit()                                      # transaction boundary = the service method (≈ @Transactional)
        return customer

    def update(self, customer_id: int, data: CustomerUpdate) -> Customer:
        customer = self.get(customer_id)
        for field, value in data.model_dump(exclude_unset=True).items():   # partial update: only fields the client sent
            setattr(customer, field, value)
        self.repo.session.commit()
        return customer

    def delete(self, customer_id: int) -> None:
        self.repo.delete(self.get(customer_id))
        self.repo.session.commit()
```

The service depends on a **concrete** `CustomerRepository`. For unit tests we will pass a hand-written fake with the same methods — duck typing (Chapter 7). If the codebase grows, declare `class CustomerRepo(Protocol)` and annotate `repo: CustomerRepo`; nothing else changes.

## 21.6 API layer

```python
# src/customer_api/api/customers.py
from typing import Annotated
from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.orm import Session
from customer_api.db import get_session
from customer_api.repository import CustomerRepository
from customer_api.schemas import CustomerCreate, CustomerOut, CustomerUpdate
from customer_api.service import CustomerService

router = APIRouter(prefix="/customers", tags=["customers"])

def get_service(session: Annotated[Session, Depends(get_session)]) -> CustomerService:
    return CustomerService(CustomerRepository(session))

ServiceDep = Annotated[CustomerService, Depends(get_service)]

@router.get("", response_model=list[CustomerOut])
def list_customers(
    service: ServiceDep,
    q: Annotated[str | None, Query(max_length=50)] = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
):
    return service.search(q, limit, offset)

@router.get("/{customer_id}", response_model=CustomerOut)
def get_customer(customer_id: int, service: ServiceDep):
    return service.get(customer_id)

@router.post("", response_model=CustomerOut, status_code=status.HTTP_201_CREATED)
def create_customer(payload: CustomerCreate, service: ServiceDep):
    return service.create(payload)

@router.patch("/{customer_id}", response_model=CustomerOut)
def update_customer(customer_id: int, payload: CustomerUpdate, service: ServiceDep):
    return service.update(customer_id, payload)

@router.delete("/{customer_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_customer(customer_id: int, service: ServiceDep) -> None:
    service.delete(customer_id)
```

```python
# src/customer_api/main.py
import logging
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from customer_api.api.customers import router as customers_router
from customer_api.errors import ConflictError, NotFoundError

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")

def create_app() -> FastAPI:                                # app factory — easy to build variants in tests
    app = FastAPI(title="Customer API", version="1.0.0")
    app.include_router(customers_router)

    @app.exception_handler(NotFoundError)
    async def not_found(_: Request, exc: NotFoundError):
        return JSONResponse(status_code=404, content={"detail": str(exc)})

    @app.exception_handler(ConflictError)
    async def conflict(_: Request, exc: ConflictError):
        return JSONResponse(status_code=409, content={"detail": str(exc)})

    @app.get("/health", tags=["ops"])
    def health() -> dict[str, str]:
        return {"status": "ok"}

    return app

app = create_app()
```

Run: `uv run alembic upgrade head` (Chapter 22) then `uv run fastapi dev src/customer_api/main.py`, open `/docs`, create a customer.

## 21.7 Tests

```python
# tests/test_service.py — unit tests, no database
import pytest
from customer_api.errors import ConflictError, NotFoundError
from customer_api.models import Customer
from customer_api.schemas import CustomerCreate
from customer_api.service import CustomerService

class FakeSession:
    def commit(self): self.committed = True

class FakeRepo:
    def __init__(self):
        self.rows: dict[int, Customer] = {}
        self.session = FakeSession()
    def get(self, cid): return self.rows.get(cid)
    def get_by_email(self, email): return next((c for c in self.rows.values() if c.email == email), None)
    def add(self, c):
        c.id = len(self.rows) + 1; self.rows[c.id] = c; return c
    def delete(self, c): del self.rows[c.id]
    def search(self, q, limit, offset): return list(self.rows.values())[offset:offset + limit]

@pytest.fixture
def service():
    return CustomerService(FakeRepo())

def test_create_returns_customer_with_id(service):
    c = service.create(CustomerCreate(name="Ada", email="ada@x.io"))
    assert c.id == 1 and c.name == "Ada"

def test_duplicate_email_is_conflict(service):
    service.create(CustomerCreate(name="Ada", email="ada@x.io"))
    with pytest.raises(ConflictError):
        service.create(CustomerCreate(name="Ada 2", email="ada@x.io"))

def test_get_missing_raises(service):
    with pytest.raises(NotFoundError):
        service.get(999)
```

```python
# tests/conftest.py — real PostgreSQL for API tests
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from testcontainers.postgres import PostgresContainer
from customer_api.db import get_session
from customer_api.main import app
from customer_api.models import Base

@pytest.fixture(scope="session")
def engine():
    with PostgresContainer("postgres:17", driver="psycopg") as pg:
        engine = create_engine(pg.get_connection_url())
        Base.metadata.create_all(engine)
        yield engine

@pytest.fixture
def session(engine):
    conn = engine.connect(); txn = conn.begin()
    with Session(bind=conn, expire_on_commit=False, join_transaction_mode="create_savepoint") as s:
        yield s                                      # service commits become savepoints
    txn.rollback(); conn.close()                     # ...and everything is rolled back after each test

@pytest.fixture
def client(session):
    app.dependency_overrides[get_session] = lambda: session
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

```python
# tests/test_api.py
def test_customer_lifecycle(client):
    r = client.post("/customers", json={"name": "Ada", "email": "ada@x.io"})
    assert r.status_code == 201
    cid = r.json()["id"]

    assert client.get(f"/customers/{cid}").json()["email"] == "ada@x.io"

    r = client.patch(f"/customers/{cid}", json={"name": "Ada L."})
    assert r.json()["name"] == "Ada L."

    assert client.post("/customers", json={"name": "X", "email": "ada@x.io"}).status_code == 409
    assert client.post("/customers", json={"name": "", "email": "bad"}).status_code == 422

    assert client.delete(f"/customers/{cid}").status_code == 204
    assert client.get(f"/customers/{cid}").status_code == 404
```

```bash
uv run pytest -q
uv run ruff check . && uv run mypy src/
```

## 21.8 What to notice

- Roughly 250 lines for a validated, documented, tested CRUD service with PostgreSQL.
- The HTTP layer contains no logic; the service contains no HTTP or SQL; the repository contains all SQL.
- Transactions are explicit `commit()` calls at the service boundary — no `@Transactional` proxy magic; Chapter 22 explains the session model that makes this safe.
- Tests replace dependencies by *providing a different callable*, not by a mocking framework.
- The API is fully described in `/docs` without writing a single line of documentation.

## Knowledge check — Chapter 21

1. Why keep `models.py` (SQLAlchemy) and `schemas.py` (Pydantic) separate even in a small app?
2. Where is the transaction boundary in this design and why? What plays the role of `@Transactional`?
3. How does `FakeRepo` satisfy `CustomerService` without implementing an interface? What would you add when the team grows?
4. Explain the API-test fixture chain: engine → session → client. What guarantees isolation between tests?
5. Name three Spring components this service does not need at all, and say what replaces each.

---

# 22. Databases — SQLAlchemy and Alembic

## 22.1 The Python database stack

```text
Java:     JPA (spec) → Hibernate (impl) → JDBC (driver API) → PostgreSQL JDBC driver → PostgreSQL
Python:   SQLAlchemy ORM → SQLAlchemy Core → DB-API 2.0 (driver spec) → psycopg 3 → PostgreSQL
```

- **DB-API 2.0 (PEP 249)** is Python's JDBC: a spec every driver implements (`connect()`, `cursor()`, `execute()`, `fetchall()`, parameter placeholders). You can use it directly:

  ```python
  import psycopg
  with psycopg.connect("postgresql://app:app@localhost/customers") as conn:
      with conn.cursor() as cur:
          cur.execute("SELECT id, name FROM customers WHERE email = %s", (email,))   # parameters, never f-strings (SQL injection)
          row = cur.fetchone()
      conn.commit()
  ```

  Plain DB-API is fine for scripts and Lambdas. For applications, SQLAlchemy adds pooling, an expression language and the ORM.
- **Drivers for PostgreSQL**: `psycopg` (v3, sync + async, recommended), `psycopg2` (legacy, everywhere in old code), `asyncpg` (fast async-only, used with SQLAlchemy async).
- **SQLAlchemy** is *two layers*: **Core** (SQL expression language + connection pool + dialects — think jOOQ/JDBC-template) and **ORM** (sessions, mapped classes, relationships, unit of work — think Hibernate). Unlike JPA, you use them together freely; the ORM is optional.
- **Alembic** is SQLAlchemy's migration tool (Flyway/Liquibase).

## 22.2 SQLAlchemy ORM (2.0 style)

**Mapping** — declarative classes with `Mapped[...]` annotations (already seen in Chapter 21):

```python
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase): ...

class Customer(Base):
    __tablename__ = "customers"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(unique=True)
    orders: Mapped[list["Order"]] = relationship(back_populates="customer", cascade="all, delete-orphan")

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[int] = mapped_column(ForeignKey("customers.id"))
    total_cents: Mapped[int]
    note: Mapped[str | None]                       # nullable inferred from the Optional type
    customer: Mapped[Customer] = relationship(back_populates="orders")
```

`Mapped[str]` ↔ `@Column(nullable=false) String`; `Mapped[str | None]` ↔ nullable; `relationship` ↔ `@OneToMany`/`@ManyToOne` with `back_populates` ↔ `mappedBy`. Legacy 1.x style uses `Column(Integer, primary_key=True)` without annotations — same concept, less typing support.

**Sessions and the unit of work.** A `Session` is Hibernate's `Session`/JPA `EntityManager`: an **identity map** (one object per row per session), **change tracking** (modify attributes → UPDATE at flush), and a transaction. Objects are *attached* to a session while it is open:

```python
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session, sessionmaker

engine = create_engine("postgresql+psycopg://app:app@localhost/customers", pool_size=5, echo=False)
SessionLocal = sessionmaker(engine, expire_on_commit=False)

with SessionLocal() as session:                  # session per unit of work; closed at the end
    with session.begin():                        # transaction: commit on success, rollback on exception
        c = Customer(name="Ada", email="ada@x.io")
        session.add(c)                           # pending → INSERT at flush
        session.flush()                          # send SQL now (c.id populated); still inside the transaction
        c.name = "Ada L."                        # dirty → UPDATE at commit/flush (no explicit save!)
    # committed here

with SessionLocal() as session:
    ada = session.get(Customer, 1)                                   # by PK (identity map hit if loaded)
    adas = session.scalars(select(Customer).where(Customer.name.like("Ada%")).order_by(Customer.id)).all()
    row = session.execute(select(Customer.id, Customer.email).where(Customer.id == 1)).one()   # tuples of columns
    count = session.scalar(select(func.count()).select_from(Customer))
    session.delete(ada); session.commit()
```

Key points versus Hibernate/Spring Data:

- **No repository interfaces with derived queries** (`findByEmailAndActiveTrue`). You write `select(...)` expressions — it is closer to JPA Criteria/jOOQ, but readable. Most Python teams put those queries in small repository classes or functions (Chapter 21).
- **Explicit transaction boundaries**: `session.begin()` or `session.commit()`; there is no `@Transactional` proxy. FastAPI's dependency gives you one session per request; the service commits.
- **Lazy loading** exists and causes the same N+1 problems. Fix with `selectinload`/`joinedload`: `select(Customer).options(selectinload(Customer.orders))`. Since the session is closed at the end of the request, accessing a lazy relationship after that raises `DetachedInstanceError` — the Python cousin of `LazyInitializationException`. `expire_on_commit=False` keeps loaded attributes readable after commit.
- **Autoflush**: queries flush pending changes first, as in Hibernate.
- `session.merge`, `session.refresh`, `session.expunge`, cascades, `Enum` columns, `JSONB` (`sqlalchemy.dialects.postgresql.JSONB`), `ARRAY`, hybrid properties — all present; find them as needed.

**Core when you want SQL without objects** (reports, bulk ops):

```python
from sqlalchemy import text, insert, update
with engine.begin() as conn:                                            # Connection-level transaction
    conn.execute(insert(Customer), [{"name": "A", "email": "a@x"}, {"name": "B", "email": "b@x"}])   # executemany bulk insert
    conn.execute(update(Customer).where(Customer.id == 1).values(name="Z"))
    rows = conn.execute(text("SELECT count(*) FROM customers WHERE created_at > :since"), {"since": since}).scalar()
```

**Async SQLAlchemy** (for `async def` endpoints):

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
engine = create_async_engine("postgresql+psycopg://app:app@localhost/customers")     # or postgresql+asyncpg://
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncIterator[AsyncSession]:
    async with AsyncSessionLocal() as session:
        yield session

async def find(session: AsyncSession, cid: int) -> Customer | None:
    return await session.get(Customer, cid)
res = (await session.scalars(select(Customer))).all()
```

Same mapping, same expressions; `await` on every I/O call, and lazy loading is *not* allowed implicitly (use eager loading options) — which is a good discipline anyway.

## 22.3 JPA/Hibernate ↔ SQLAlchemy

| JPA / Hibernate / Spring Data | SQLAlchemy |
|---|---|
| `@Entity`, `@Table` | `class X(Base): __tablename__ = ...` |
| `@Id @GeneratedValue` | `mapped_column(primary_key=True)` (autoincrement by default) |
| `@Column(nullable=false, length=100)` | `Mapped[str] = mapped_column(String(100))` |
| `@OneToMany(mappedBy)` / `@ManyToOne` | `relationship(back_populates=...)` + `ForeignKey` |
| `EntityManager` / `Session` | `Session` |
| `EntityManagerFactory` + `DataSource` + HikariCP | `Engine` (+ built-in pool) + `sessionmaker` |
| `em.find(Cls, id)` | `session.get(Cls, id)` |
| JPQL / Criteria API | `select(...)` expression language |
| `@Query(nativeQuery=true)` | `text("...")` |
| `JpaRepository` derived queries | your repository functions/classes |
| `@Transactional` | `with session.begin():` / explicit `commit()` |
| `LazyInitializationException` | `DetachedInstanceError` |
| `@EntityGraph` / `JOIN FETCH` | `selectinload` / `joinedload` |
| `FetchType.LAZY` default for collections | lazy by default (`lazy="selectin"` etc. to change) |
| Flyway / Liquibase | Alembic |
| `spring.jpa.show-sql` | `create_engine(..., echo=True)` |
| Hibernate Validator | Pydantic (at the API layer, not the entity) |

## 22.4 Migrations with Alembic

```bash
uv add alembic
uv run alembic init alembic            # creates alembic.ini and alembic/ (env.py, versions/)
```

Point Alembic at your metadata and URL (`alembic/env.py`):

```python
from customer_api.config import get_settings
from customer_api.models import Base
config.set_main_option("sqlalchemy.url", get_settings().database_url)
target_metadata = Base.metadata          # enables autogenerate
```

Workflow (identical in spirit to Flyway with generated diffs):

```bash
uv run alembic revision --autogenerate -m "create customers"   # diff models vs DB → alembic/versions/xxxx_create_customers.py
uv run alembic upgrade head                                     # apply
uv run alembic downgrade -1                                     # roll back one
uv run alembic history; uv run alembic current
```

A generated migration is Python, and you review/edit it like a Flyway SQL file:

```python
def upgrade() -> None:
    op.create_table(
        "customers",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("name", sa.String(100), nullable=False),
        sa.Column("email", sa.String(255), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.text("now()"), nullable=False),
    )
    op.create_index("ix_customers_email", "customers", ["email"], unique=True)

def downgrade() -> None:
    op.drop_table("customers")
```

Autogenerate detects tables/columns/indexes/FKs; it does *not* reliably detect renames or data migrations — write those by hand with `op.execute(...)`. Run `alembic upgrade head` in the deployment pipeline (or an init container), never `Base.metadata.create_all()` in production.

## 22.5 A minimal database-backed API (self-contained)

Chapter 21 is the full version. Here is the shortest end-to-end form, useful as a template for internal tools and Lambda-backed APIs:

```python
# app.py — run: uv run fastapi dev app.py
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException
from pydantic import BaseModel, ConfigDict
from sqlalchemy import String, create_engine, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column, sessionmaker

engine = create_engine("postgresql+psycopg://app:app@localhost:5432/notes")
SessionLocal = sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase): ...
class Note(Base):
    __tablename__ = "notes"
    id: Mapped[int] = mapped_column(primary_key=True)
    text: Mapped[str] = mapped_column(String(500))

class NoteIn(BaseModel):
    text: str
class NoteOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    text: str

def db():
    with SessionLocal() as s:
        yield s
DB = Annotated[Session, Depends(db)]

app = FastAPI()
Base.metadata.create_all(engine)       # demo only — use Alembic in real projects

@app.post("/notes", response_model=NoteOut, status_code=201)
def create(note: NoteIn, s: DB):
    n = Note(text=note.text); s.add(n); s.commit(); return n

@app.get("/notes", response_model=list[NoteOut])
def list_notes(s: DB):
    return s.scalars(select(Note).order_by(Note.id)).all()

@app.get("/notes/{note_id}", response_model=NoteOut)
def get(note_id: int, s: DB):
    n = s.get(Note, note_id)
    if n is None: raise HTTPException(404)
    return n
```

## 22.6 pgvector (preview for Project 4)

PostgreSQL + the `pgvector` extension stores embeddings next to your data. With SQLAlchemy:

```python
from pgvector.sqlalchemy import Vector          # uv add pgvector
class Chunk(Base):
    __tablename__ = "chunks"
    id: Mapped[int] = mapped_column(primary_key=True)
    content: Mapped[str]
    embedding: Mapped[list[float]] = mapped_column(Vector(1024))

nearest = session.scalars(select(Chunk).order_by(Chunk.embedding.cosine_distance(query_vec)).limit(5)).all()
```

## Knowledge check — Chapter 22

1. What is DB-API 2.0 and which Java API is it? Where does SQLAlchemy Core sit relative to it?
2. Explain identity map + change tracking in SQLAlchemy, and why `c.name = "x"; session.commit()` issues an UPDATE with no `save()`.
3. Where is the transaction boundary in a FastAPI + SQLAlchemy service, and why is there no `@Transactional`?
4. What is `DetachedInstanceError`, which Hibernate exception is it, and what are two ways to avoid it?
5. Compare Alembic with Flyway: what does autogenerate give you, and what must you still write by hand?

---

# 23. Dependency Management

Chapter 2 gave you the workflow. This chapter explains *how the ecosystem works underneath*, so you can debug it and read any repository's setup.

## 23.1 PyPI, distributions, wheels

- **PyPI** (pypi.org) is Maven Central. Anyone can publish; there is no group-id namespace, so names are flat and first-come (`requests`, `fastapi`). Corporate mirrors/proxies (Artifactory, AWS CodeArtifact) work just like for Maven.
- A **distribution** is what you install (`pip install fastapi`). It contains one or more importable packages; the names can differ (`pip install pillow` → `import PIL`, `pip install beautifulsoup4` → `import bs4`, `pip install psycopg[binary]` → `import psycopg`).
- Distributions ship as **wheels** (`.whl`, pre-built zip archives — the JAR analogue) or **sdists** (source tarballs that must be built on install, possibly compiling C — slow and fragile). Packages with native code publish platform-specific wheels (`numpy-2.5.1-cp314-cp314-manylinux_2_28_x86_64.whl`); if no wheel matches your Python version/OS, you get an sdist build or an error. This is why "the newest Python isn't supported by library X yet" is a real thing for a few months after each release.
- **Extras** (`fastapi[standard]`, `pydantic[email]`, `anthropic[bedrock]`) install optional dependency sets.

## 23.2 Resolution and locking

Maven resolves with "nearest wins" and no lockfile (you pin via dependencyManagement/BOMs). Gradle has lockfiles optionally. Python's modern tools do **full backtracking resolution** against version constraints (PEP 440 specifiers) and produce a **cross-platform lockfile**:

```text
pyproject.toml   ─ what you *want* (ranges)        →  like pom.xml
uv.lock          ─ what you *got* (exact versions,  →  like a Gradle lockfile / package-lock.json
                   hashes, per-platform markers)
```

- `uv lock` resolves; `uv sync` installs exactly the lock; `uv lock --upgrade` moves everything within ranges; `uv add pkg` edits pyproject and re-locks.
- Only one version of each package can be installed in an environment (no classpath shading). If two dependencies require incompatible versions of a third, resolution **fails** with a clear message — this is stricter than Maven's silent nearest-wins and is the source of most "dependency hell" stories in the NumPy/PyTorch/Transformers world. The fix is usually to loosen your own pin or upgrade the older dependency.
- `requirements.txt` has no resolver semantics of its own: `pip install -r requirements.txt` installs what is listed. Teams that still use it generate it from a lock (`uv export --format requirements-txt > requirements.txt`) for Docker/Lambda images.

## 23.3 Reading a repository you did not create

| You see | It means | You run |
|---|---|---|
| `pyproject.toml` + `uv.lock` | uv project | `uv sync` |
| `pyproject.toml` + `poetry.lock` | Poetry project | `pipx install poetry && poetry install` (or `uv sync` often works too) |
| `pyproject.toml` only, `[project.dependencies]` | PEP 621 project, any tool | `uv sync` (uv locks it) or `pip install -e .` in a venv |
| `requirements.txt` (+ `requirements-dev.txt`) | classic pip project | `uv venv && uv pip install -r requirements.txt` |
| `setup.py` / `setup.cfg` | older packaging | `pip install -e .` |
| `Pipfile` / `Pipfile.lock` | pipenv (2017-era) | `pipenv install` |
| `environment.yml` | conda | `conda env create -f environment.yml` |
| `.python-version` | pyenv/uv interpreter pin | uv honours it |

## 23.4 Publishing a library (briefly)

```bash
uv build                # → dist/my_lib-0.1.0-py3-none-any.whl + .tar.gz
uv publish              # → PyPI (or --index for a private index)
```

Versioning is SemVer by convention (`__version__`, or read from package metadata). Internal shared code between Python services is usually published to a private index or referenced as a git dependency (`uv add git+https://...`) — the equivalent of an internal Maven artifact.

## 23.5 Security and hygiene

- `uv lock` stores hashes; `uv sync --frozen` verifies them (supply-chain protection like Maven checksum verification).
- Audit: `uvx pip-audit` (checks installed packages against vulnerability DBs; the OWASP dependency-check role).
- Do not install with `sudo`, do not `pip install` into a venv managed by uv (use `uv add`), commit `uv.lock`, ignore `.venv/`.
- In Docker, `UV_NO_DEV=1` / `uv sync --no-dev` excludes test tooling (the `test`/`provided` scope split).

## 23.6 Maven/Gradle ↔ Python

| Maven/Gradle | Python (uv) |
|---|---|
| Maven Central | PyPI |
| JAR | wheel |
| `groupId:artifactId:version` | `name==version` (flat namespace) |
| `pom.xml`/`build.gradle` | `pyproject.toml` |
| BOM / `dependencyManagement` | constraint files / lockfile; no direct equivalent to BOMs |
| Gradle lockfile | `uv.lock` |
| `<scope>test</scope>` | `[dependency-groups] dev = [...]` |
| `<optional>` / profiles | extras `pkg[extra]` |
| `mvn dependency:tree` | `uv tree` |
| shading/relocation | not possible; one version per env |
| Nexus/Artifactory | same, as a PyPI index |
| `~/.m2` | uv cache (`~/.cache/uv`) |

## Knowledge check — Chapter 23

1. What is the difference between a distribution and a package, and why can `pip install X` lead to `import Y`?
2. Why is a wheel like a JAR — and in what way is a NumPy wheel *unlike* a JAR?
3. Two libraries require incompatible versions of `pydantic`. What happens in Maven, and what happens with uv? Which is safer?
4. What is `requirements.txt` in the modern workflow, and how do you produce one from a uv project?
5. Why does Python's "one version per environment" rule make virtual environments non-negotiable?

---

# 24. Python Project Structure

Spring Boot projects look alike because the framework and Maven impose a shape. Python has **more freedom and fewer conventions**; the price is that you must choose deliberately. Here are the structures that work, and what each directory is for.

## 24.1 A script or tiny tool

```text
tool/
├── pyproject.toml
├── README.md
├── main.py              # everything; run with `uv run main.py`
└── tests/
    └── test_main.py
```

Perfectly professional for a one-file CLI, a Lambda function, a data script. Do not create packages for 150 lines.

## 24.2 A small service — flat layout

```text
customer-api/
├── pyproject.toml
├── .env.example
├── Dockerfile
├── customer_api/            # the package (importable as customer_api)
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── db.py
│   ├── models.py
│   ├── schemas.py
│   ├── repository.py
│   ├── service.py
│   └── api/
│       ├── __init__.py
│       └── customers.py
├── alembic/ + alembic.ini
└── tests/
    ├── conftest.py
    └── test_*.py
```

Good for services up to a few thousand lines. Layers are *files*; sub-packages appear when a file grows.

## 24.3 A larger service — `src/` layout with feature or layer packages

```text
my-app/
├── pyproject.toml
├── README.md
├── Dockerfile
├── src/
│   └── my_app/
│       ├── __init__.py
│       ├── main.py               # app factory + wiring
│       ├── config.py             # Settings
│       ├── api/                  # HTTP layer: one router module per resource
│       │   ├── __init__.py
│       │   ├── deps.py           # shared Depends() providers (session, auth, current user)
│       │   ├── customers.py
│       │   └── chat.py
│       ├── services/             # business logic, framework-free
│       │   ├── __init__.py
│       │   ├── customers.py
│       │   └── chat.py
│       ├── models/               # persistence (SQLAlchemy) — or split "domain/" vs "db/"
│       │   ├── __init__.py       # re-exports Base and all entities (so Alembic sees them)
│       │   └── customer.py
│       ├── schemas/              # Pydantic API models
│       │   └── customer.py
│       ├── repositories/         # data access
│       │   └── customers.py
│       ├── ai/                   # LLM/Bedrock clients, prompts, tools (AI apps)
│       │   ├── llm.py
│       │   └── prompts/
│       └── core/                 # cross-cutting: logging, errors, security utilities
│           ├── errors.py
│           └── logging.py
├── alembic/
├── scripts/                      # one-off admin/dev scripts
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

**Purpose of each directory:**

| Directory | Purpose | Spring analogue |
|---|---|---|
| `src/my_app/` | the importable package; `src/` keeps it out of `sys.path` until installed (Chapter 10) | `src/main/java/com/acme/app` |
| `main.py` | app factory, router registration, exception handlers, lifespan | `Application.java` + `WebConfig` |
| `api/` | routers, request parsing, HTTP status mapping — *no logic* | `controller/` |
| `api/deps.py` | reusable `Depends()` providers | security config + `@Bean` helpers |
| `services/` | use cases; depends on repositories/clients, not on FastAPI | `service/` |
| `models/` | ORM entities | `entity/` / `domain/` |
| `schemas/` | Pydantic models for I/O | `dto/` |
| `repositories/` | queries | `repository/` |
| `ai/` | model clients, prompt templates, tool definitions, RAG pipeline | (new) |
| `core/` | logging setup, error types, utilities | `common/`, `config/` |
| `tests/` | pytest; `conftest.py` per directory for shared fixtures | `src/test/java` |
| `scripts/` | maintenance scripts run with `uv run scripts/x.py` | `Makefile` targets / Gradle tasks |
| `alembic/` | migrations | `db/migration` |

**Feature-based alternative** (`src/my_app/customers/{api,service,models,schemas,repository}.py`, `src/my_app/chat/...`) — "package by feature" — scales better for many bounded contexts and is common in Django and larger FastAPI apps. Both are fine; pick one and be consistent.

## 24.4 An AI / RAG application

```text
doc-qa/
├── pyproject.toml
├── src/doc_qa/
│   ├── main.py                 # FastAPI: /ask, /ingest
│   ├── config.py
│   ├── ingestion/              # loaders, chunking, embedding, indexing
│   │   ├── loaders.py
│   │   ├── chunking.py
│   │   └── pipeline.py
│   ├── retrieval/              # vector store access, hybrid search, reranking
│   │   └── store.py
│   ├── generation/             # prompt templates, LLM client, answer synthesis
│   │   ├── prompts.py
│   │   └── llm.py
│   ├── schemas.py
│   └── evaluation/             # offline eval scripts + datasets
├── notebooks/                  # exploratory Jupyter notebooks (not production code)
├── data/                       # sample docs (small) — large data stays in S3
└── tests/
```

Distinct from a CRUD service: a **`notebooks/`** directory for exploration, an **offline pipeline** (ingestion) separate from the **online path** (retrieval + generation), and an **evaluation** module — AI apps are judged on quality, not just correctness.

## 24.5 A library

```text
my-lib/
├── pyproject.toml         # [build-system], [project] with classifiers, urls
├── README.md, LICENSE, CHANGELOG.md
├── src/my_lib/
│   ├── __init__.py        # public API re-exports + __version__
│   ├── py.typed           # marker: this package ships type hints
│   └── ...
├── tests/
└── docs/
```

## 24.6 Principles that survive every layout

1. **Framework at the edges.** `api/` knows FastAPI; `services/` does not import it. That keeps services testable and reusable from a CLI or a Lambda.
2. **Configuration in one place** (`config.py`), read once, injected.
3. **No import-time side effects** (Chapter 10): clients created in lifespan or lazily.
4. **Tests mirror the source loosely**, fixtures in `conftest.py`, integration tests separated by directory or marker.
5. **Flat until it hurts.** Start with files; split into packages when a file exceeds a few hundred lines or has distinct concerns.
6. **One `pyproject.toml` per deployable** (a monorepo can hold several with `[tool.uv.workspace]`).

There is no "correct" Python architecture. Spring's conventions exist because a container needs to find things; Python needs only that modules import cleanly and that a reader can find the code. Optimise for the reader.

## Knowledge check — Chapter 24

1. When is a single `main.py` the right structure, and what is the cost of "over-structuring" a small Python project?
2. Why does the `src/` layout exist, and what problem from Chapter 10 does it prevent?
3. Which directory must never import FastAPI, and why does that matter for reusing code in a Lambda or CLI?
4. How does an AI/RAG project's structure differ from a CRUD service's?
5. Compare "package by layer" and "package by feature" in Python — why does Python make both easy where Spring nudges toward one?

---

# Part V — Python for data, ML and GenAI

---

# 25. Python for Data and AI — NumPy, pandas, Jupyter

You now know Python the language. This part is about *why it is the language of AI* and how to read the code you will meet in AI projects. The goal is not to make you a data scientist; it is to make `import numpy as np` and `df.groupby(...)` as unmysterious as `List<Integer>` and `Collectors.groupingBy`.

## 25.1 Why Python owns the data/AI stack

Three historical facts:

1. **NumPy (2006, from Numeric 1995)** gave Python a fast n-dimensional array with the heavy lifting in C/Fortran (BLAS/LAPACK). Scientists could write vectorised math in a scripting language and get C speed.
2. Everything since — pandas (2008), scikit-learn (2007), matplotlib (2003), Jupyter (2014), TensorFlow (2015), PyTorch (2016), Transformers (2018) — was built *on* NumPy's array model or *for* NumPy users. The network effect is the moat: researchers publish Python code; libraries wrap Python; new models ship with Python examples first.
3. Python's REPL/notebook culture (Chapter 3) fits exploratory work: run a cell, look at the data, tweak, rerun. Java's compile cycle and verbosity are wrong for that loop.

So Python is the *interface* to the AI stack; the math runs in C, C++, CUDA and Rust underneath. Chapter 1's mental model again.

## 25.2 Jupyter — the exploration environment

```bash
uv add --dev jupyterlab ipykernel
uv run jupyter lab              # opens a browser UI; notebooks are .ipynb (JSON of cells + outputs)
```

A notebook is a sequence of **cells** executed in a persistent kernel (a running Python process). State persists between cells, so you load a dataset once and iterate on analysis cells. VS Code and PyCharm open `.ipynb` files natively. Conventions:

- Notebooks are for **exploration, prototyping, reports**; production code lives in `.py` modules that notebooks import. Do not ship notebooks as services.
- Cells can run out of order; "restart kernel and run all" before trusting a notebook.
- Magics: `%time`, `%%time`, `!pip list` (shell), `%load_ext autoreload` / `%autoreload 2` (reload edited modules).
- Strip outputs before committing (`nbstripout`) or use Jupytext to pair with `.py`.

For a Java developer the closest analogy is a REPL with a saved transcript and inline charts — `jshell` with pictures. It is the standard way to *try* an AI library before writing a service.

## 25.3 NumPy — the array

```python
import numpy as np

vector = np.array([1, 2, 3])                   # ndarray, dtype int64, shape (3,)
matrix = np.array([[1.0, 2.0], [3.0, 4.0]])    # shape (2, 2), dtype float64
vector.shape, vector.dtype, vector.ndim, matrix.size
```

**Why is it different from a list?** A Python list is an array of pointers to arbitrary heap objects (Chapter 4). A NumPy array is a **single contiguous block of typed memory** (like a C `double[]`/Java `double[]`), plus metadata (shape, strides, dtype). Consequences:

| | `list` | `np.ndarray` |
|---|---|---|
| Element type | anything, mixed | one fixed dtype (`float32`, `int64`, `bool`, …) |
| Memory for 1M floats | ~32 MB (pointers + float objects) | 8 MB (float64) or 4 MB (float32) |
| `[1,2,3] + [4,5,6]` | concatenation → `[1,2,3,4,5,6]` | element-wise → `[5,7,9]` |
| `xs * 2` | repetition | element-wise multiply |
| Loop over 1M elements | ~50–100 ms in Python | ~1 ms in C (vectorised) |
| Multidimensional | nested lists | native `shape=(rows, cols, ...)` |

**Vectorisation** is the core skill: express the computation as whole-array operations so the loop runs in C:

```python
prices = np.array([10.0, 20.0, 30.0])
taxed = prices * 1.2                          # broadcasting a scalar
discounted = np.where(prices > 15, prices * 0.9, prices)     # vectorised if/else
prices.sum(), prices.mean(), prices.std(), prices.max(), prices.argmax()
big = prices[prices > 15]                     # boolean mask indexing → array([20., 30.])
matrix @ vector[:2]                           # matrix multiplication (@ operator)
matrix.T                                      # transpose
np.zeros((3, 4)); np.ones(5); np.arange(0, 1, 0.1); np.linspace(0, 1, 11); np.random.default_rng(0).normal(size=(2, 3))
matrix.reshape(4), matrix.reshape(4, 1)       # views, no copy
arr.astype(np.float32)                        # dtype conversion (ML models often want float32)
```

**Broadcasting** stretches smaller arrays to match larger ones by rule (trailing dimensions must be equal or 1): `matrix - matrix.mean(axis=0)` subtracts the column means from every row. `axis=0` = down the rows (per column); `axis=1` = across columns (per row).

**Where you will meet NumPy in GenAI:** embeddings are arrays (`shape=(n_texts, 1024)`), similarity is a dot product, top-k retrieval is `argsort`, model inputs/outputs in Transformers/PyTorch convert to/from NumPy, images are `(H, W, 3)` uint8 arrays.

```python
def cosine_similarity(a: np.ndarray, b: np.ndarray) -> np.ndarray:
    """Cosine similarity of each row of a (n, d) against each row of b (m, d) → (n, m)."""
    a_n = a / np.linalg.norm(a, axis=1, keepdims=True)
    b_n = b / np.linalg.norm(b, axis=1, keepdims=True)
    return a_n @ b_n.T

query_vec = embed(["python generators"])           # (1, 1024)
scores = cosine_similarity(query_vec, doc_vecs)[0]  # (n_docs,)
top5 = np.argsort(-scores)[:5]                      # indices of the 5 most similar docs
```

That is a complete in-memory vector search — the thing pgvector/OpenSearch do at scale.

## 25.4 pandas — tables

**What is it?** A library for tabular data: a `DataFrame` is a table (dict of typed columns, each a NumPy/Arrow array) with a row index; a `Series` is one column. It is the SQL-in-Python of data work: filtering, joins, group-by, pivots, time series, I/O to CSV/Parquet/SQL/JSON.

```python
import pandas as pd

df = pd.read_csv("customers.csv")                    # also read_parquet, read_json, read_sql(query, engine), read_excel
df.head(); df.shape; df.dtypes; df.describe(); df.info()

df["age"]                                            # Series
df[["name", "age"]]                                  # DataFrame with two columns
df[df["age"] > 30]                                   # boolean filter (WHERE)
df.query("age > 30 and country == 'DE'")             # same, string form
df.loc[df["active"], "email"]                        # label-based select rows+cols
df.iloc[0:5, 0:2]                                    # position-based
df["age_group"] = pd.cut(df["age"], [0, 30, 60, 120], labels=["young", "mid", "senior"])   # new column
df.assign(total=df["qty"] * df["price"])             # non-mutating column add

df.groupby("country")["revenue"].sum()               # GROUP BY … SUM
df.groupby(["country", "plan"]).agg(customers=("id", "count"), avg_age=("age", "mean")).reset_index()
df.sort_values("revenue", ascending=False).head(10)  # ORDER BY … LIMIT
df.merge(orders, on="customer_id", how="left")       # JOIN
pd.concat([df1, df2])                                # UNION ALL
df.pivot_table(index="country", columns="plan", values="revenue", aggfunc="sum", fill_value=0)
df.isna().sum(); df.dropna(subset=["email"]); df.fillna({"age": df["age"].median()})
df["created_at"] = pd.to_datetime(df["created_at"]); df.set_index("created_at").resample("ME")["revenue"].sum()
df.to_dict(orient="records")                         # → list[dict] — hand to Pydantic/JSON
df.to_parquet("out.parquet"); df.to_sql("customers", engine, if_exists="append", index=False)
```

pandas 3.0 notes (January 2026): strings are a dedicated `str` dtype (Arrow-backed when PyArrow is installed) instead of `object`; **Copy-on-Write** is always on, so chained assignment like `df[df.age > 30]["x"] = 1` never silently mutates (it raises/warns) — use `df.loc[mask, "x"] = 1`.

**Java comparison.** There is no mainstream Java equivalent; the nearest are SQL, Spark's `Dataset`, or Tablesaw. Think of a `DataFrame` as an in-memory SQL table with a fluent API and a NumPy engine. `Collectors.groupingBy` ↔ `groupby`; streams over POJOs ↔ column operations — but *columnar*, not row-by-row.

**Where you will meet pandas in GenAI:** preparing evaluation datasets, analysing LLM outputs (costs, latencies, scores), loading CSV/Excel documents for RAG, feature tables for classical ML, and the ubiquitous "load → inspect → clean → export" in notebooks. In services you often don't need it; in notebooks you always do.

## 25.5 matplotlib — plots

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(df["date"], df["latency_ms"], label="p50")
ax.set(title="LLM latency", xlabel="date", ylabel="ms"); ax.legend()
fig.savefig("latency.png", dpi=150)          # or fig.show() / display inline in Jupyter
df["latency_ms"].hist(bins=30)               # pandas wraps matplotlib
```

You will mostly *read* matplotlib in notebooks and training scripts (loss curves, confusion matrices). seaborn and plotly are higher-level alternatives.

## 25.6 scikit-learn (preview)

Classical machine learning with one consistent API: every model has `.fit(X, y)` and `.predict(X)`. Chapter 26 walks through it. It sits on NumPy/pandas; its `Pipeline` and preprocessing are used even in LLM projects (baselines, classifiers over embeddings, evaluation metrics).

## 25.7 The array mindset for a backend developer

The thing to internalise: **in data/AI Python, you do not write loops over items; you write operations over whole arrays/columns.** A Java developer's `for (var row : rows) { row.x = row.x * 2; }` becomes `df["x"] *= 2`. Once that clicks, PyTorch (Chapter 30) is "NumPy with autograd and GPUs", and most AI code reads naturally.

## Knowledge check — Chapter 25

1. Why is `[1,2,3] * 2` different from `np.array([1,2,3]) * 2`, and what does that reveal about how each stores data?
2. What does "vectorised" mean, and why is it the key to Python being fast enough for numeric work?
3. Explain `axis=0` vs `axis=1` and broadcasting with `matrix - matrix.mean(axis=0)`.
4. Map `SELECT country, SUM(revenue) FROM t WHERE age > 30 GROUP BY country ORDER BY 2 DESC` to pandas.
5. When would you use a notebook, and when must the code move to a module?
6. Write cosine similarity between one query embedding and a matrix of document embeddings without a Python loop.

---

# 26. Python and Machine Learning — scikit-learn

The goal: understand the vocabulary and the shape of ML code so that a training script, an evaluation notebook, or a scikit-learn baseline inside a GenAI project is readable.

## 26.1 Vocabulary

| Term | Meaning | Backend analogy |
|---|---|---|
| **Dataset** | a table of examples; rows = samples | the training table |
| **Features (X)** | input columns, numeric (after encoding); shape `(n_samples, n_features)` | request fields |
| **Label / target (y)** | what we want to predict; shape `(n_samples,)` | the expected response |
| **Training** | fit model parameters to minimise error on the training set | building the index |
| **Model** | the learned function `X → y` (weights + architecture) | the compiled artefact |
| **Inference / prediction** | apply the model to new inputs | serving a request |
| **Evaluation** | measure quality on held-out data (test set) with metrics | integration tests + SLOs |
| **Overfitting** | great on training data, bad on new data | tests that pass only on the fixtures |
| **Train/test split** | hold out ~20% for honest evaluation | staging vs prod data |
| **Hyperparameters** | settings chosen by you (learning rate, tree depth) | tuning knobs |
| **Classification / regression** | predict a category / predict a number | |
| **Embedding** | a dense vector representing an item (word, sentence, image) such that similar items are near each other | a hash that preserves meaning |

A supervised ML workflow:

```text
raw data → features (X) + labels (y) → split → fit(model, X_train, y_train) → predict(X_test) → metrics → save model → serve
```

## 26.2 A very small scikit-learn example

Classify support tickets into "billing" vs "technical" from text — a task you might later replace with an LLM, and a good baseline to compare against.

```python
# uv add scikit-learn pandas joblib
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline
from sklearn.metrics import accuracy_score, classification_report
import joblib

df = pd.DataFrame({
    "text": ["invoice is wrong", "charged twice this month", "app crashes on login",
             "cannot reset my password", "refund not received", "error 500 when uploading"],
    "label": ["billing", "billing", "technical", "technical", "billing", "technical"],
})
X_train, X_test, y_train, y_test = train_test_split(df["text"], df["label"], test_size=0.33, random_state=42)

model = Pipeline([
    ("tfidf", TfidfVectorizer(ngram_range=(1, 2))),     # text → sparse numeric features
    ("clf", LogisticRegression(max_iter=1000)),         # linear classifier
])
model.fit(X_train, y_train)                             # training

preds = model.predict(X_test)                           # inference
print(accuracy_score(y_test, preds))
print(classification_report(y_test, preds))

joblib.dump(model, "ticket_classifier.joblib")          # persist (pickle-based)
model = joblib.load("ticket_classifier.joblib")
model.predict(["double charge on my card"])             # array(['billing'], dtype=object)
model.predict_proba(["double charge on my card"])       # class probabilities
```

Everything in scikit-learn follows this **estimator API**: `fit`, `predict`/`transform`, `score`, composed with `Pipeline`. Swap `LogisticRegression` for `RandomForestClassifier` or `GradientBoostingClassifier` and nothing else changes. `GridSearchCV` tunes hyperparameters with cross-validation.

## 26.3 Serving a model (the backend developer's part)

```python
from fastapi import FastAPI
from pydantic import BaseModel
import joblib

model = joblib.load("ticket_classifier.joblib")     # load once at startup (lifespan in a real app)
app = FastAPI()

class Ticket(BaseModel):
    text: str

@app.post("/classify")
def classify(t: Ticket) -> dict[str, str | float]:
    label = model.predict([t.text])[0]
    confidence = float(model.predict_proba([t.text]).max())
    return {"label": label, "confidence": confidence}
```

This "load artefact, wrap in an endpoint" shape is the same for scikit-learn, PyTorch and Transformers models. Model files are versioned artefacts (S3/model registry), not code. In AWS terms: SageMaker endpoints do this hosting for you; for small models a FastAPI container on ECS or a Lambda is enough.

## 26.4 Where classical ML sits next to LLMs

- Cheap, fast, deterministic **baselines** and **routers** (classify intent before choosing a prompt/model).
- **Evaluation** metrics (precision/recall/F1 from `sklearn.metrics`) are used to score LLM outputs against labelled sets.
- **Embeddings + a linear classifier** is often better than a prompt for high-volume classification: get embeddings from Bedrock/OpenAI/sentence-transformers, `LogisticRegression().fit(embeddings, labels)`.
- Structured/tabular prediction (churn, fraud) is still gradient-boosted trees (XGBoost/LightGBM), not LLMs.

## Knowledge check — Chapter 26

1. Define features, labels, training and inference in one sentence each. Which of these happens in the request path of a service?
2. What is the estimator API and why does it make scikit-learn code look the same across models?
3. Why is a train/test split necessary, and what failure does it detect?
4. Given precomputed embeddings for 50,000 support tickets and labels, sketch the cheapest classifier you could build and serve.
5. Name two roles classical ML still plays inside a GenAI system.

---

# 27. Python for Generative AI — the ecosystem

This chapter is a *map*, not a tutorial: what each library is for, where it sits, and which you actually need. Chapters 28–30 and the projects go deeper on the ones that matter for you.

## 27.1 The layers

```text
┌───────────────────────────────────────────────────────────────────────────┐
│ Application frameworks / orchestration                                    │
│   LangChain · LlamaIndex · Pydantic AI · Haystack · Strands Agents (AWS)   │
│   (prompts, chains, tools, agents, RAG pipelines, memory)                  │
├───────────────────────────────────────────────────────────────────────────┤
│ Model access — hosted APIs                                                 │
│   anthropic · openai · google-genai · boto3 (Amazon Bedrock) · litellm     │
├───────────────────────────────────────────────────────────────────────────┤
│ Model access — local / self-hosted                                         │
│   transformers (Hugging Face) · sentence-transformers · vLLM · Ollama      │
├───────────────────────────────────────────────────────────────────────────┤
│ Data / retrieval infrastructure                                            │
│   pgvector · OpenSearch · FAISS · Chroma · Qdrant · Pinecone · S3 · DynamoDB│
├───────────────────────────────────────────────────────────────────────────┤
│ Deep learning runtime                                                      │
│   PyTorch (torch) · (JAX, TensorFlow less common now) · CUDA               │
├───────────────────────────────────────────────────────────────────────────┤
│ Foundations                                                                │
│   NumPy · pandas · Pydantic · httpx · asyncio · FastAPI                    │
└───────────────────────────────────────────────────────────────────────────┘
```

A backend/GenAI developer lives in the top four layers and reads the fifth. Only ML engineers train in the fifth.

## 27.2 The libraries, one paragraph each

**`anthropic` — Anthropic Python SDK (1.x).** Typed client for the Claude Messages API: `client.messages.create(...)`, streaming helpers, `messages.parse()` for Pydantic-typed structured outputs, a `@beta_tool` decorator + `tool_runner` for agentic loops, async client, Bedrock/Vertex/Foundry variants. Built on `httpx2`, Pydantic response objects.

**`openai` — OpenAI Python SDK (2.x).** Client for the Responses API (`client.responses.create`) and the older Chat Completions API, embeddings, files, batches, streaming, structured outputs via Pydantic. Many other providers expose an "OpenAI-compatible" endpoint, so this SDK also talks to vLLM, Ollama, and various hosts.

**`google-genai` — Gemini SDK.** `genai.Client().models.generate_content(...)`; Gemini models, Vertex AI backend option.

**`boto3` — AWS SDK for Python.** The generic AWS client: S3, DynamoDB, Lambda, SQS… and **Amazon Bedrock** (`bedrock`, `bedrock-runtime`, `bedrock-agent-runtime`). Not GenAI-specific — the same dict-in/dict-out API as everything else in AWS. Chapter 29.

**Amazon Bedrock** (a service, not a library): unified access to Anthropic, Amazon Nova, Meta, Mistral, Cohere… models via `Converse`/`ConverseStream`/`InvokeModel`, plus Knowledge Bases (managed RAG), Agents, Guardrails, embeddings (Titan, Cohere). From Python you reach it through boto3 or through provider SDK adapters (`anthropic[bedrock]`, `langchain-aws`).

**`transformers` — Hugging Face Transformers (5.x).** Load and run thousands of open models (LLMs, embedding models, classifiers, vision, speech) by name from the Hugging Face Hub: `pipeline(...)`, `AutoTokenizer`, `AutoModelFor...`. PyTorch-only since v5. This is how you run a model *locally* rather than call an API.

**Hugging Face Hub / `datasets` / `tokenizers` / `sentence-transformers` / `peft` / `accelerate`.** The surrounding ecosystem: model & dataset hosting, fast tokenisers, embedding models (`SentenceTransformer("all-MiniLM-L6-v2").encode(texts)`), parameter-efficient fine-tuning (LoRA), multi-GPU helpers.

**`torch` — PyTorch (2.x).** The deep-learning runtime: tensors (NumPy on GPU), autograd, `nn.Module`, optimisers. Transformers, sentence-transformers and most research code are PyTorch. You need the vocabulary (Chapter 30); you rarely write training loops as a backend developer.

**`langchain` (1.x) + `langchain-core` + `langchain-aws` / `langchain-anthropic` / `langchain-openai` + `langgraph`.** An orchestration framework: uniform model interfaces, prompt templates, output parsers, tools, retrievers, vector store adapters, and `create_agent` (built on LangGraph, a graph runtime for stateful multi-step agents). Big, opinionated, very common in tutorials and enterprises. Chapter 30.

**`llama-index` (0.14.x).** A data framework focused on RAG: document loaders, node parsers (chunking), indexes, retrievers, query engines, plus agents. Best when the problem is "get my documents into an LLM well". Chapter 30.

**`pydantic-ai` (2.x).** A newer, type-first agent framework from the Pydantic team: agents with typed dependencies and typed outputs, tools as decorated functions, model-agnostic. Attractive for developers who like Pydantic and FastAPI.

**Strands Agents (AWS).** AWS's open-source model-driven agent SDK with first-class Bedrock support; you will see it in AWS samples alongside the Bedrock Agents managed service.

**`litellm`.** A thin "call any provider with one function" shim; useful for switching models, plus a proxy for cost tracking.

**Vector stores.** `pgvector` (PostgreSQL extension; `pgvector` Python package + SQLAlchemy/psycopg), **Amazon OpenSearch** (`opensearch-py`, k-NN), FAISS (in-memory library from Meta), Chroma (embedded dev store), Qdrant/Weaviate/Pinecone/Milvus (dedicated services). For a Java/PostgreSQL developer, **pgvector is the natural first choice**.

**Evaluation & observability.** `ragas`, `deepeval`, LangSmith, Langfuse, Arize Phoenix, and Bedrock's own evaluations; `tiktoken`/provider tokenisers for counting; `tenacity` for retries; `structlog` for logs.

## 27.3 Which ones you actually need

| Goal | Learn now | Learn later / optional |
|---|---|---|
| Call an LLM from a service | `anthropic` **or** `openai`, and `boto3` for Bedrock | `google-genai`, `litellm` |
| Build RAG on your documents | `boto3` (Bedrock embeddings) or `sentence-transformers`, `pgvector`, chunking (own code or `langchain-text-splitters`) | `llama-index`, OpenSearch |
| Agents / tools | SDK-native tool use (`@beta_tool`, Bedrock Converse tools) | `langchain`/`langgraph`, `pydantic-ai`, Strands, Bedrock Agents |
| Read AI code in the wild | `langchain` concepts, `transformers.pipeline`, PyTorch tensor basics | training loops, `peft`, `accelerate` |
| Run open models locally | Ollama (outside Python) + `openai`-compatible client, or `transformers.pipeline` | vLLM serving |

The rule: **learn the provider SDK and Bedrock first, add a framework when you need its abstractions (many models, many tools, complex agent graphs), and understand the framework's pieces as thin wrappers over what you already know.**

## Knowledge check — Chapter 27

1. Place `boto3`, `langchain`, `transformers`, `torch`, and `pgvector` on the layer diagram.
2. What is the difference between calling a model through `anthropic`/`openai` and running it with `transformers`?
3. When does LangChain earn its place, and when is the raw SDK the better choice?
4. Why is pgvector a sensible first vector store for a PostgreSQL-fluent backend developer?
5. Which of these libraries does a backend developer *read* but rarely *write*: `torch`, `boto3`, `fastapi`?

---

# 28. Calling LLMs from Python

The core loop of every GenAI application:

```text
Python
  ↓  build a request: model id, system prompt, message list, parameters, (tools, output schema)
LLM API   (Anthropic / OpenAI / Gemini / Bedrock)
  ↓  HTTP + JSON; the model generates tokens
Response
  ↓  content blocks (text / tool calls), stop reason, usage (tokens)
Python   parses, validates, stores, streams to the client, or loops with tool results
```

Everything else — RAG, agents, memory — is manipulation of that message list. This chapter shows the *same operations* across SDKs so you can read any of them, then compares with Java.

## 28.1 Anthropic SDK (`anthropic` 1.x)

```bash
uv add anthropic            # ANTHROPIC_API_KEY in the environment
```

**Basic call with a system prompt:**

```python
import anthropic

client = anthropic.Anthropic()                 # reads ANTHROPIC_API_KEY; create once, reuse (it holds a connection pool)

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    system="You are a concise assistant for a customer-support team. Answer in plain English.",
    messages=[
        {"role": "user", "content": "Explain what a virtual environment is in two sentences."},
    ],
)
for block in response.content:                 # content is a list of typed blocks (text, tool_use, thinking...)
    if block.type == "text":
        print(block.text)
print(response.stop_reason, response.usage.input_tokens, response.usage.output_tokens)
```

**Multi-turn:** you resend the history — the API is stateless.

```python
history: list[anthropic.types.MessageParam] = []

def chat(user_text: str) -> str:
    history.append({"role": "user", "content": user_text})
    resp = client.messages.create(model="claude-opus-5", max_tokens=1024, system=SYSTEM, messages=history)
    reply = "".join(b.text for b in resp.content if b.type == "text")
    history.append({"role": "assistant", "content": resp.content})      # append the content blocks as returned
    return reply
```

**Streaming** (tokens as they are generated — essential for chat UIs):

```python
with client.messages.stream(model="claude-opus-5", max_tokens=2048, messages=[{"role": "user", "content": "Write a haiku about Python."}]) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    final = stream.get_final_message()          # full Message with usage after the stream ends
```

**Structured output** — get a validated Pydantic object instead of parsing text:

```python
from pydantic import BaseModel
from typing import Literal

class TicketAnalysis(BaseModel):
    category: Literal["billing", "technical", "other"]
    urgency: int                                 # 1–5
    summary: str

resp = client.messages.parse(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": f"Analyse this support ticket:\n\n{ticket_text}"}],
    output_format=TicketAnalysis,                # the SDK converts the model to a JSON schema and validates the reply
)
analysis: TicketAnalysis = resp.parsed_output
```

**Tool use** — let the model call your functions; the SDK's tool runner drives the loop:

```python
from anthropic import beta_tool

@beta_tool
def get_order_status(order_id: str) -> str:
    """Look up the shipping status of an order.

    Args:
        order_id: The order identifier, e.g. ORD-1234.
    """
    return repo.status(order_id)                 # your code; return a string the model can read

runner = client.beta.messages.tool_runner(
    model="claude-opus-5",
    max_tokens=2048,
    tools=[get_order_status],                    # schema generated from type hints + docstring (Chapters 3, 8)
    messages=[{"role": "user", "content": "Where is order ORD-1234?"}],
)
final = runner.until_done()                      # loops: model → tool call → your function → model → ... → final answer
print("".join(b.text for b in final.content if b.type == "text"))
```

**Error handling** — catch a chain, most specific first:

```python
import anthropic

try:
    resp = client.messages.create(...)
except anthropic.RateLimitError as e:            # 429 → back off (the SDK already retried twice)
    ...
except anthropic.APIStatusError as e:            # other 4xx/5xx; e.status_code, e.message
    ...
except anthropic.APIConnectionError:             # network
    ...
```

The SDK retries connection errors/408/409/429/5xx twice with backoff by default (`max_retries=`), has a 10-minute timeout (`timeout=`), and exposes `client.with_options(...)` for per-call overrides. `AsyncAnthropic` mirrors every call with `await` (used in Chapter 17's fan-out and in FastAPI streaming endpoints).

## 28.2 OpenAI SDK (`openai` 2.x) — same shapes, different names

```python
from openai import OpenAI
client = OpenAI()                                          # OPENAI_API_KEY

resp = client.responses.create(                            # Responses API (the current primary API)
    model="gpt-5",
    instructions="You are a concise assistant.",           # ≈ system prompt
    input="Explain what a virtual environment is in two sentences.",
)
print(resp.output_text)

# Streaming
for event in client.responses.create(model="gpt-5", input="Write a haiku.", stream=True):
    if event.type == "response.output_text.delta":
        print(event.delta, end="", flush=True)

# Structured output
resp = client.responses.parse(model="gpt-5", input=[{"role": "user", "content": ticket_text}], text_format=TicketAnalysis)
analysis = resp.output_parsed

# Embeddings
vec = client.embeddings.create(model="text-embedding-3-small", input=["hello world"]).data[0].embedding   # list[float]
```

Older code uses `client.chat.completions.create(model=..., messages=[{"role": "system", ...}, {"role": "user", ...}])` and reads `resp.choices[0].message.content` — the Chat Completions API, still supported and widely used by OpenAI-compatible servers (vLLM, Ollama).

## 28.3 Gemini (`google-genai`)

```python
from google import genai
client = genai.Client()                                    # GEMINI_API_KEY (or Vertex AI config)
resp = client.models.generate_content(model="gemini-2.5-flash", contents="Explain virtual environments briefly.")
print(resp.text)
```

## 28.4 The provider-independent skeleton

Whatever SDK, your service code should depend on a *small interface you own*, so providers are swappable and tests are trivial (Chapter 7):

```python
from typing import Protocol

class LLM(Protocol):
    def complete(self, system: str, user: str) -> str: ...

class AnthropicLLM:
    def __init__(self, model: str = "claude-opus-5"):
        self.client, self.model = anthropic.Anthropic(), model
    def complete(self, system: str, user: str) -> str:
        r = self.client.messages.create(model=self.model, max_tokens=1024, system=system,
                                        messages=[{"role": "user", "content": user}])
        return "".join(b.text for b in r.content if b.type == "text")

class FakeLLM:
    def complete(self, system: str, user: str) -> str: return "fake answer"
```

Chapter 29 adds a `BedrockLLM` with the same shape; Project 3 turns this into a FastAPI service.

## 28.5 The same operations in Java

Anthropic Java SDK:

```java
import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.models.messages.*;

AnthropicClient client = AnthropicOkHttpClient.fromEnv();      // ANTHROPIC_API_KEY

MessageCreateParams params = MessageCreateParams.builder()
        .model("claude-opus-5")
        .maxTokens(1024)
        .system("You are a concise assistant.")
        .addUserMessage("Explain what a virtual environment is in two sentences.")
        .build();

Message message = client.messages().create(params);
message.content().forEach(block -> block.text().ifPresent(t -> System.out.println(t.text())));

// Streaming
try (var stream = client.messages().createStreaming(params)) {
    stream.stream()
          .flatMap(event -> event.contentBlockDelta().stream())
          .flatMap(delta -> delta.delta().text().stream())
          .forEach(t -> System.out.print(t.text()));
}
```

Spring AI (from your Spring AI guide):

```java
String reply = chatClient.prompt()
        .system("You are a concise assistant.")
        .user("Explain what a virtual environment is in two sentences.")
        .call()
        .content();

TicketAnalysis analysis = chatClient.prompt().user(ticketText).call().entity(TicketAnalysis.class);   // structured output
```

**What differs and what doesn't:**

| Aspect | Java | Python |
|---|---|---|
| Request building | builders, typed enums, `Optional` accessors | keyword arguments and dicts/TypedDicts |
| Response | typed objects with `Optional` getters | Pydantic objects; `block.type` discriminates |
| Structured output | `.entity(Class)` / JSON schema | `messages.parse(output_format=PydanticModel)` |
| Tools | annotated methods / `@Tool` beans | `@beta_tool` functions; docstrings become descriptions |
| Streaming | `Stream`/`Flux` | `for text in stream.text_stream` / `async for` |
| Concurrency | threads / virtual threads / Reactor | `AsyncAnthropic` + `asyncio.gather` |
| Lines of code | ~2–3× more | — |

The *concepts* are identical. If you can call a model from Spring AI, the Python code above is a translation, not new knowledge. The practical difference: Python examples for every new model feature appear first, and notebooks make prompt iteration faster.

## 28.6 Practical guidance that applies in both languages

- **Create the client once** (module-level or lifespan) — it holds the HTTP pool.
- **Never log full prompts with PII** by default; log token usage and latency.
- **Set `max_tokens` deliberately**; hitting it truncates output (`stop_reason == "max_tokens"`).
- **Stream for anything user-facing** or long; use the SDK helper, not hand-rolled SSE parsing.
- **Prefer structured outputs** to regex-parsing text.
- **Retries and timeouts** are configured on the client; add your own idempotency where it matters.
- **Version prompts** like code (keep them in files under `ai/prompts/`, load at startup).
- **Count cost**: `usage.input_tokens/output_tokens` × price; cache system prompts where the provider supports prompt caching.

## Knowledge check — Chapter 28

1. Why does a multi-turn chat require the client to resend the whole history? What does this imply for context-window management?
2. What does `messages.parse(output_format=Model)` do that `messages.create` does not, and what Chapter 9 feature makes it possible?
3. Explain how `@beta_tool` turns a plain Python function into something the model can call. Which two pieces of the function are used?
4. Compare error handling: what should you catch first, and why is `except Exception` insufficient?
5. Translate `chatClient.prompt().system(s).user(u).call().content()` into the Anthropic Python SDK.

---

# 29. Python + AWS for AI — boto3 and Amazon Bedrock

Most Python AWS/GenAI code you will encounter — in AWS samples, Lambda functions, notebooks, the AIP-C01 exam — is boto3. This chapter makes that code readable and shows how it maps to the AWS SDK for Java you already know.

```text
Python application
       ↓  boto3.client("bedrock-runtime")  — builds a signed HTTPS request (SigV4) from a dict
boto3 / botocore
       ↓  JSON over HTTPS
Amazon Bedrock (Runtime API: Converse / ConverseStream / InvokeModel)
       ↓  routes to the model
Foundation Model (Anthropic Claude, Amazon Nova, Meta Llama, …)
       ↓  generated text / tool calls / embeddings
Python application ← plain dicts
```

## 29.1 boto3 in three concepts

```bash
uv add boto3 "boto3-stubs[bedrock-runtime,s3,dynamodb,lambda]"    # stubs → typed clients and auto-complete
```

1. **Session** — credentials + region. Usually implicit; boto3 builds a default session from the standard credential chain (env vars `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN`, `~/.aws/credentials` + `AWS_PROFILE`, SSO, ECS/Lambda task role, EC2 instance role). **Never put keys in code.**
2. **Client** — low-level, one method per API operation, **dict in → dict out**, generated from the service's JSON model (`botocore`). This is 95% of what you use: `boto3.client("bedrock-runtime")`, `boto3.client("s3")`.
3. **Resource** — a higher-level object-oriented layer for a few services (`boto3.resource("dynamodb").Table("x")`, `s3.Bucket("b").objects.all()`). Convenient for S3/DynamoDB; not available for Bedrock.

```python
import boto3
from botocore.config import Config

config = Config(region_name="us-east-1", retries={"max_attempts": 5, "mode": "adaptive"}, read_timeout=120)
bedrock = boto3.client("bedrock-runtime", config=config)       # create once, reuse (thread-safe)
s3 = boto3.client("s3")
```

**Java comparison.** boto3 ≈ AWS SDK for Java v2. `BedrockRuntimeClient.builder().region(...).build()` ↔ `boto3.client("bedrock-runtime", region_name=...)`; `ConverseRequest.builder()...build()` ↔ a dict of keyword arguments; typed `ConverseResponse` ↔ a dict. Java gives you compile-time structure; Python gives you speed of writing and, with `boto3-stubs`, most of the typing back. The credential chain, retries, and signing are identical concepts.

## 29.2 Bedrock: which API

| API | Use |
|---|---|
| **`converse` / `converse_stream`** | chat with *any* Bedrock chat model through one message format; supports system prompts, multi-turn, tools, images/documents, guardrails. **Use this for text generation.** |
| `invoke_model` / `invoke_model_with_response_stream` | raw, model-specific request/response bodies. Needed for **embeddings** (Titan, Cohere), image models, and model-specific features not exposed via Converse. |
| `bedrock` (control plane) | `list_foundation_models`, `get_foundation_model`, model customisation jobs |
| `bedrock-agent-runtime` | Knowledge Bases (`retrieve`, `retrieve_and_generate`) and Agents (`invoke_agent`) |

Model IDs: base IDs like `amazon.nova-pro-v1:0` and **cross-region inference profile IDs** like `us.anthropic.claude-sonnet-4-5-20250929-v1:0` (the `us.`/`eu.`/`global.` prefix routes across regions for throughput). Check the console/`list_foundation_models` for what is enabled in your account and region.

## 29.3 Text generation with Converse

```python
MODEL_ID = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"     # or "amazon.nova-pro-v1:0"

def ask(prompt: str, system: str | None = None) -> str:
    kwargs = {
        "modelId": MODEL_ID,
        "messages": [{"role": "user", "content": [{"text": prompt}]}],     # content is a LIST of blocks
        "inferenceConfig": {"maxTokens": 1024, "temperature": 0.2},
    }
    if system:
        kwargs["system"] = [{"text": system}]
    response = bedrock.converse(**kwargs)
    return response["output"]["message"]["content"][0]["text"]

print(ask("Explain what a virtual environment is in two sentences.", system="Be concise."))
```

The response is a nested dict (Chapter 4 warned you):

```python
{
  "output": {"message": {"role": "assistant", "content": [{"text": "..."}]}},
  "stopReason": "end_turn",                       # or "max_tokens", "tool_use", "guardrail_intervened"
  "usage": {"inputTokens": 31, "outputTokens": 58, "totalTokens": 89},
  "metrics": {"latencyMs": 812},
}
```

**Multi-turn**: append the assistant's `output["message"]` to `messages` and send the next user turn — same statelessness as every LLM API.

**Streaming**:

```python
def stream(prompt: str):
    resp = bedrock.converse_stream(modelId=MODEL_ID, messages=[{"role": "user", "content": [{"text": prompt}]}])
    for event in resp["stream"]:                                   # a generator of event dicts (Chapter 14)
        if "contentBlockDelta" in event:
            yield event["contentBlockDelta"]["delta"].get("text", "")
        elif "metadata" in event:
            usage = event["metadata"]["usage"]                     # tokens at the end

for chunk in stream("Write a haiku about PostgreSQL."):
    print(chunk, end="", flush=True)
```

**Tool use with Converse** (the provider-neutral version of Chapter 28's tool runner — you write the loop):

```python
import json

tools = [{"toolSpec": {
    "name": "get_order_status",
    "description": "Look up the shipping status of an order.",
    "inputSchema": {"json": {"type": "object", "properties": {"order_id": {"type": "string"}}, "required": ["order_id"]}},
}}]

def run_with_tools(user_text: str) -> str:
    messages = [{"role": "user", "content": [{"text": user_text}]}]
    while True:
        resp = bedrock.converse(modelId=MODEL_ID, messages=messages, toolConfig={"tools": tools})
        message = resp["output"]["message"]
        messages.append(message)
        if resp["stopReason"] != "tool_use":
            return "".join(b.get("text", "") for b in message["content"])
        results = []
        for block in message["content"]:
            if "toolUse" in block:
                call = block["toolUse"]
                output = get_order_status(**call["input"])                      # your function
                results.append({"toolResult": {"toolUseId": call["toolUseId"], "content": [{"json": {"status": output}}]}})
        messages.append({"role": "user", "content": results})
```

**Guardrails**: add `guardrailConfig={"guardrailIdentifier": "...", "guardrailVersion": "1"}` to `converse`; a blocked request returns `stopReason == "guardrail_intervened"`.

**Documents/images**: content blocks `{"image": {"format": "png", "source": {"bytes": data}}}` and `{"document": {"format": "pdf", "name": "spec", "source": {"bytes": data}}}`.

## 29.4 Embeddings with `invoke_model`

```python
import json

def embed(text: str, model_id: str = "amazon.titan-embed-text-v2:0") -> list[float]:
    body = json.dumps({"inputText": text, "dimensions": 1024, "normalize": True})   # Titan v2 request shape
    resp = bedrock.invoke_model(modelId=model_id, body=body, contentType="application/json", accept="application/json")
    return json.loads(resp["body"].read())["embedding"]

def embed_many(texts: list[str]) -> list[list[float]]:
    return [embed(t) for t in texts]           # Titan v2 is one text per call; parallelise with a ThreadPoolExecutor (Chapter 17)
```

`invoke_model` takes and returns raw JSON bytes whose shape is *model-specific* (Titan: `inputText`/`embedding`; Cohere Embed: `texts`/`embeddings`; Anthropic native: the Messages format). That is exactly why Converse exists for chat. Embedding vectors are `list[float]` → convert to NumPy for math (Chapter 25) or store in pgvector/OpenSearch.

## 29.5 Knowledge Bases (managed RAG)

```python
kb = boto3.client("bedrock-agent-runtime")

# Retrieval only — you build the prompt
hits = kb.retrieve(knowledgeBaseId=KB_ID, retrievalQuery={"text": query},
                   retrievalConfiguration={"vectorSearchConfiguration": {"numberOfResults": 5}})
chunks = [h["content"]["text"] for h in hits["retrievalResults"]]

# Retrieval + generation in one call
answer = kb.retrieve_and_generate(
    input={"text": query},
    retrieveAndGenerateConfiguration={"type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {"knowledgeBaseId": KB_ID, "modelArn": MODEL_ARN}},
)["output"]["text"]
```

Project 4 builds the do-it-yourself version with pgvector so you understand what the managed service does.

## 29.6 The rest of the AWS cast, briefly

**S3** — documents in, artefacts out:

```python
s3.put_object(Bucket="docs", Key="reports/q3.pdf", Body=pdf_bytes, ContentType="application/pdf")
obj = s3.get_object(Bucket="docs", Key="reports/q3.pdf")
data: bytes = obj["Body"].read()                                  # streaming body; .iter_chunks() for large objects
for page in s3.get_paginator("list_objects_v2").paginate(Bucket="docs", Prefix="reports/"):   # paginators = generators
    for item in page.get("Contents", []):
        print(item["Key"], item["Size"])
s3.download_file("docs", "reports/q3.pdf", "/tmp/q3.pdf"); s3.upload_file("/tmp/out.json", "docs", "out.json")
```

**Lambda** — the handler is a function; boto3 clients are created *outside* it so warm invocations reuse them; the runtime injects credentials from the execution role:

```python
import json, os, boto3
bedrock = boto3.client("bedrock-runtime")                     # module level: created once per container
MODEL_ID = os.environ["MODEL_ID"]

def handler(event, context):                                  # API Gateway / function URL event
    body = json.loads(event.get("body") or "{}")
    resp = bedrock.converse(modelId=MODEL_ID, messages=[{"role": "user", "content": [{"text": body["prompt"]}]}],
                            inferenceConfig={"maxTokens": 512})
    return {"statusCode": 200, "headers": {"content-type": "application/json"},
            "body": json.dumps({"answer": resp["output"]["message"]["content"][0]["text"]})}
```

Package dependencies with `uv export --format requirements-txt` + `pip install -t` into the zip, or use a container image / Lambda layer. Python cold starts are ~100–300 ms versus JVM seconds — a key reason Python dominates small AI Lambdas.

**DynamoDB** — conversation history, session state:

```python
table = boto3.resource("dynamodb").Table("chat-sessions")
table.put_item(Item={"session_id": sid, "ts": int(time.time()), "role": "user", "content": text})
resp = table.query(KeyConditionExpression=Key("session_id").eq(sid), ScanIndexForward=True, Limit=50)
history = resp["Items"]                                  # note: numbers come back as Decimal
```

**OpenSearch** (vector search at scale) via `opensearch-py`:

```python
from opensearchpy import OpenSearch, AWSV4SignerAuth, RequestsHttpConnection
client = OpenSearch(hosts=[{"host": HOST, "port": 443}], http_auth=AWSV4SignerAuth(boto3.Session().get_credentials(), REGION, "aoss"),
                    use_ssl=True, connection_class=RequestsHttpConnection)
client.index(index="chunks", body={"text": chunk, "embedding": vec, "source": src})
hits = client.search(index="chunks", body={"size": 5, "query": {"knn": {"embedding": {"vector": qvec, "k": 5}}}})["hits"]["hits"]
```

**SQS / EventBridge / Step Functions** — same dict style; used to decouple ingestion pipelines from the API.

## 29.7 Error handling with boto3

boto3 raises **one exception type for API errors**, `botocore.exceptions.ClientError`, with the service's error code inside; plus per-client exception classes for convenience:

```python
from botocore.exceptions import ClientError, BotoCoreError

try:
    resp = bedrock.converse(...)
except bedrock.exceptions.ThrottlingException:                 # 429-equivalent; adaptive retry mode already backed off
    ...
except bedrock.exceptions.ValidationException as e:            # bad model id / malformed request
    ...
except bedrock.exceptions.AccessDeniedException:                # IAM, or model access not enabled in this region
    ...
except ClientError as e:                                       # anything else from the service
    code = e.response["Error"]["Code"]; msg = e.response["Error"]["Message"]
    ...
except BotoCoreError:                                          # client-side: no credentials, endpoint resolution, timeouts
    ...
```

Configure retries (`Config(retries={"mode": "adaptive", "max_attempts": 5})`) rather than writing retry loops; set `read_timeout` high for long generations; log `resp["ResponseMetadata"]["RequestId"]` when opening support cases.

## 29.8 boto3 is sync — async and threads

boto3 blocks. Options in an async FastAPI app: run it in a thread (`await asyncio.to_thread(bedrock.converse, **kwargs)` or a plain `def` endpoint), or use **`aioboto3`/`aiobotocore`** for a native async client. The **Anthropic SDK's Bedrock client** (`anthropic[bedrock]`: `AnthropicBedrock(aws_region=...)` / `AsyncAnthropicBedrock`) is another route for Claude-on-Bedrock that gives you the SDK's typed API, streaming helpers and async support while still signing with your AWS credentials; **LangChain's `ChatBedrockConverse`** (Chapter 30) wraps Converse with async support.

## 29.9 The same architecture from Java

```java
BedrockRuntimeClient client = BedrockRuntimeClient.builder().region(Region.US_EAST_1).build();

Message userMessage = Message.builder()
        .role(ConversationRole.USER)
        .content(ContentBlock.fromText("Explain what a virtual environment is in two sentences."))
        .build();

ConverseResponse response = client.converse(ConverseRequest.builder()
        .modelId("us.anthropic.claude-sonnet-4-5-20250929-v1:0")
        .messages(userMessage)
        .system(SystemContentBlock.fromText("Be concise."))
        .inferenceConfig(c -> c.maxTokens(1024).temperature(0.2f))
        .build());

String text = response.output().message().content().get(0).text();
```

Or with Spring AI: `spring-ai-starter-model-bedrock-converse` + `ChatClient`. Same request, same response tree, same IAM, same model IDs; the Java version is typed and verbose, the Python version is dicts and short. When reading Python AWS code, mentally map each dict key to the builder method you know: `modelId` → `.modelId()`, `inferenceConfig.maxTokens` → `.inferenceConfig(c -> c.maxTokens())`, `response["output"]["message"]["content"][0]["text"]` → `response.output().message().content().get(0).text()`.

## Knowledge check — Chapter 29

1. What are the three core boto3 concepts, and which one do you use for Bedrock?
2. Why does Bedrock have both `converse` and `invoke_model`? When must you use `invoke_model`?
3. Walk through the tool-use loop with Converse: what does the model return, what do you send back, and how does the loop end?
4. What single exception type carries all AWS service errors in boto3, and how do you distinguish a throttle from a validation error?
5. Why are boto3 clients created at module level in Lambda functions, and why is Python attractive for small AI Lambdas?
6. Translate `response["output"]["message"]["content"][0]["text"]` to the AWS SDK for Java v2.

---

# 30. Python AI Frameworks — LangChain, LlamaIndex, Hugging Face, PyTorch

Each section gives the concepts and the minimum code to read real projects. None of these is required to *call* an LLM; all of them appear in the code you will meet.

## 30.1 LangChain (1.x)

**What is it?** A framework that standardises the pieces of an LLM application so they compose: **models** (one interface over Anthropic/OpenAI/Bedrock/…), **prompts** (templates), **output parsers**, **tools**, **retrievers** and **vector stores** (RAG), and **agents** (built on **LangGraph**, a runtime for stateful, multi-step graphs). Version 1.0 (October 2025) trimmed the package to these essentials and moved legacy chains (`LLMChain`, `AgentExecutor`) to `langchain-classic` — if you see those, you are reading pre-1.0 code.

**Java comparison.** LangChain4j and Spring AI *are* this idea in Java (your Spring AI guide covers them). LangChain Python is older, larger, and more research-adjacent; the concepts map one-to-one: `ChatModel` ↔ `ChatModel`, `PromptTemplate` ↔ `PromptTemplate`, `@tool` ↔ `@Tool`, `create_agent` ↔ `AiServices`/agents, `VectorStore`/`Retriever` ↔ `EmbeddingStore`/`ContentRetriever`.

```bash
uv add langchain langchain-aws langchain-anthropic langgraph langchain-postgres langchain-text-splitters
```

**Models**

```python
from langchain.chat_models import init_chat_model
from langchain_aws import ChatBedrockConverse

llm = init_chat_model("anthropic:claude-opus-5")                                   # provider:model string
llm = ChatBedrockConverse(model="us.anthropic.claude-sonnet-4-5-20250929-v1:0", region_name="us-east-1")
llm = init_chat_model("bedrock_converse:amazon.nova-pro-v1:0")

reply = llm.invoke("Explain virtual environments in one sentence.")               # AIMessage
reply.content
for chunk in llm.stream("Write a haiku"): print(chunk.content, end="")             # streaming
await llm.ainvoke("...")                                                           # async
```

**Messages and prompts**

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
from langchain_core.prompts import ChatPromptTemplate

llm.invoke([SystemMessage("You are terse."), HumanMessage("What is uv?")])

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a support assistant for {product}. Answer from the context only."),
    ("human", "Context:\n{context}\n\nQuestion: {question}"),
])
```

**Chains (LCEL — the `|` operator)**. Runnables compose with `|`; each has `invoke`/`stream`/`batch`/`ainvoke`:

```python
from langchain_core.output_parsers import StrOutputParser

chain = prompt | llm | StrOutputParser()
answer = chain.invoke({"product": "Customer API", "context": docs_text, "question": "How do I paginate?"})
```

Read `a | b | c` as "output of a feeds b feeds c" — a pipeline of callables, implemented with the `__or__` dunder (Chapter 6).

**Structured output**

```python
structured = llm.with_structured_output(TicketAnalysis)         # Pydantic model → tool/JSON schema under the hood
structured.invoke(f"Analyse: {ticket}")                          # → TicketAnalysis instance
```

**Tools and agents**

```python
from langchain.tools import tool
from langchain.agents import create_agent

@tool
def get_order_status(order_id: str) -> str:
    """Look up the shipping status of an order by its id."""       # docstring = tool description (again)
    return repo.status(order_id)

agent = create_agent(model=llm, tools=[get_order_status], system_prompt="You are a support agent.")
result = agent.invoke({"messages": [{"role": "user", "content": "Where is ORD-1234?"}]})
print(result["messages"][-1].content)
```

`create_agent` returns a compiled **LangGraph** graph: state = the message list; nodes = "call model" and "run tools"; edges loop until the model stops calling tools. Middleware hooks let you add human-in-the-loop, summarisation, guardrails. That is the same loop as Chapter 28's tool runner and Chapter 29's hand-written Converse loop — now with persistence (checkpointers), streaming of intermediate steps and multi-agent composition.

**Retrievers and RAG**

```python
from langchain_aws import BedrockEmbeddings
from langchain_postgres import PGVector
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document

embeddings = BedrockEmbeddings(model_id="amazon.titan-embed-text-v2:0")
store = PGVector(embeddings=embeddings, collection_name="docs", connection="postgresql+psycopg://app:app@localhost/rag")

splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)
chunks = splitter.split_documents([Document(page_content=text, metadata={"source": "manual.md"})])
store.add_documents(chunks)

retriever = store.as_retriever(search_kwargs={"k": 4})
rag_chain = (
    {"context": retriever | (lambda docs: "\n\n".join(d.page_content for d in docs)), "question": lambda x: x, "product": lambda x: "Customer API"}
    | prompt | llm | StrOutputParser()
)
rag_chain.invoke("How do I paginate?")
```

A `Retriever` is anything with `invoke(query) -> list[Document]`; a `VectorStore` adds `add_documents`/`similarity_search`. Loaders (`langchain_community.document_loaders`: PDFs, S3, web pages, Confluence…) produce `Document`s.

**When to use LangChain**: many models/providers to swap, agents with tools and persistence, quick RAG prototypes, teams already on it, LangSmith tracing. **When not to**: a single provider with one or two calls (the SDK is simpler and more transparent), or when you need to control every token of the prompt and every retry.

## 30.2 LlamaIndex (0.14.x)

**What is it?** A data framework centred on RAG. Its abstractions follow the data path:

```text
Documents → Nodes (chunks, via node parsers) → Index (vector / keyword / graph) → Retriever → Query engine / Chat engine → Response
```

```bash
uv add llama-index-core llama-index-llms-bedrock-converse llama-index-embeddings-bedrock llama-index-vector-stores-postgres
```

```python
from llama_index.core import SimpleDirectoryReader, VectorStoreIndex, Settings, StorageContext
from llama_index.llms.bedrock_converse import BedrockConverse
from llama_index.embeddings.bedrock import BedrockEmbedding
from llama_index.vector_stores.postgres import PGVectorStore

Settings.llm = BedrockConverse(model="us.anthropic.claude-sonnet-4-5-20250929-v1:0")
Settings.embed_model = BedrockEmbedding(model_name="amazon.titan-embed-text-v2:0")

documents = SimpleDirectoryReader("./docs").load_data()                       # Document objects (pdf, md, txt, docx...)
vector_store = PGVectorStore.from_params(database="rag", host="localhost", user="app", password="app", table_name="chunks", embed_dim=1024)
index = VectorStoreIndex.from_documents(documents, storage_context=StorageContext.from_defaults(vector_store=vector_store))

query_engine = index.as_query_engine(similarity_top_k=4)
response = query_engine.query("How do I paginate the customers endpoint?")
print(response.response); print([n.metadata["file_name"] for n in response.source_nodes])

chat_engine = index.as_chat_engine()                                          # multi-turn with memory
```

- **Documents**: text + metadata. **Nodes**: chunks with relationships (prev/next, parent). Node parsers do sentence/token/semantic/hierarchical chunking — this is where LlamaIndex is richer than most.
- **Indexing**: `VectorStoreIndex` (embeddings), plus keyword, summary, knowledge-graph indexes; hybrid retrieval and rerankers are built in.
- **Retrieval**: `index.as_retriever()`; **query engines** add synthesis (how retrieved chunks are stuffed/refined/tree-summarised into the prompt).
- Also has agents and workflows, but its edge is data ingestion and retrieval quality. Typical pairing: LlamaIndex for ingestion/retrieval, your own code or LangChain for the app.

## 30.3 Hugging Face — models, tokenizers, datasets, Transformers

**What is it?** The Hub (huggingface.co) is the GitHub of models and datasets; `transformers` is the library that downloads and runs them locally; `tokenizers`, `datasets`, `sentence-transformers`, `peft`, `accelerate` complete the toolkit.

```bash
uv add transformers torch sentence-transformers datasets
```

```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis")                                      # downloads a default model on first use (~/.cache/huggingface)
classifier(["I love this API", "the docs are confusing"])
# [{'label': 'POSITIVE', 'score': 0.9998}, {'label': 'NEGATIVE', 'score': 0.9991}]

generator = pipeline("text-generation", model="Qwen/Qwen2.5-0.5B-Instruct", device_map="auto")
generator([{"role": "user", "content": "Explain uv in one sentence."}], max_new_tokens=60)
```

`pipeline` bundles: **tokenizer** (text → token ids) → **model** (ids → logits/hidden states) → post-processing. Under the hood:

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

name = "distilbert-base-uncased-finetuned-sst-2-english"
tokenizer = AutoTokenizer.from_pretrained(name)                # the model's vocabulary + rules
model = AutoModelForSequenceClassification.from_pretrained(name)  # weights, as a PyTorch nn.Module
model.eval()

inputs = tokenizer(["I love this API"], padding=True, truncation=True, return_tensors="pt")   # dict of tensors: input_ids, attention_mask
with torch.no_grad():                                                                         # inference: no gradients
    logits = model(**inputs).logits                                                           # shape (batch, num_labels)
probs = logits.softmax(dim=-1)
label = model.config.id2label[int(probs.argmax())]
```

**Tokenizers** matter to you because *tokens are what you pay for and what fills the context window*: `len(tokenizer("...")["input_ids"])` counts tokens for open models (each provider has its own tokenizer; Anthropic's SDK has `client.messages.count_tokens`).

**Embeddings locally** (no API cost, runs on CPU for small models):

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("BAAI/bge-small-en-v1.5")
vecs = model.encode(["python generators", "java streams"], normalize_embeddings=True)   # np.ndarray (2, 384)
vecs @ vecs.T                                                                          # cosine similarities (Chapter 25)
```

**Datasets**: `from datasets import load_dataset; ds = load_dataset("imdb", split="train[:1000]")` — Arrow-backed, streaming-capable, integrates with pandas and training loops.

**Where a backend developer meets this:** embedding services, classifiers, rerankers (`cross-encoder/...`), local evaluation, and reading the `model.generate(...)` code in research repos. Serving open LLMs in production is usually vLLM/TGI/Ollama or Bedrock — not raw `transformers`.

## 30.4 PyTorch — only the fundamentals

**What is it?** The deep-learning framework under Transformers and most research code. Three ideas explain almost any PyTorch snippet:

1. **Tensors** = NumPy arrays that can live on a GPU and track gradients.

   ```python
   import torch
   x = torch.tensor([[1.0, 2.0], [3.0, 4.0]])          # like np.array; .shape, .dtype, slicing, broadcasting all the same
   x @ x.T; x.mean(); x.to(torch.float16)
   device = "cuda" if torch.cuda.is_available() else "cpu"   # or "mps" on Apple silicon
   x = x.to(device)                                      # move to GPU; all operands must be on the same device
   x.cpu().numpy()                                       # back to NumPy (CPU only)
   ```
2. **Models are `nn.Module`s** — Python classes with parameters and a `forward` method; calling the model runs `forward` (via `__call__`, Chapter 6):

   ```python
   import torch.nn as nn
   class TinyClassifier(nn.Module):
       def __init__(self, dim: int, classes: int):
           super().__init__()
           self.net = nn.Sequential(nn.Linear(dim, 128), nn.ReLU(), nn.Linear(128, classes))
       def forward(self, x):
           return self.net(x)
   model = TinyClassifier(384, 2).to(device)
   ```
3. **Training vs inference.** Training = forward → loss → `backward()` (autograd computes gradients) → `optimizer.step()`; inference = `model.eval()` + `torch.no_grad()` + forward.

   ```python
   optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3)
   loss_fn = nn.CrossEntropyLoss()
   model.train()
   for epoch in range(5):
       for xb, yb in loader:                            # DataLoader yields batches (a generator!)
           xb, yb = xb.to(device), yb.to(device)
           optimizer.zero_grad()
           loss = loss_fn(model(xb), yb)
           loss.backward()
           optimizer.step()

   model.eval()
   with torch.no_grad():
       preds = model(x_test.to(device)).argmax(dim=1)
   torch.save(model.state_dict(), "model.pt")           # weights only; reload into the same class
   ```

**GPU**: tensors and models are explicitly moved with `.to(device)`; mismatches raise. `torch.cuda.is_available()`, mixed precision (`torch.autocast`), and `device_map="auto"` in Transformers handle most deployment cases. You do not need CUDA knowledge to *read* this.

**What you will not do as a backend developer**: design architectures, write training loops from scratch, tune optimisers. **What you will do**: recognise the loop, load a `state_dict`, move tensors to the right device, wrap inference in a FastAPI endpoint, and understand why a model server needs a GPU.

## Knowledge check — Chapter 30

1. Name LangChain's main abstractions and map each to LangChain4j/Spring AI.
2. What is `prompt | llm | StrOutputParser()`, mechanically? Which Python feature makes `|` work?
3. What is the difference between LangChain's and LlamaIndex's centre of gravity?
4. Explain tokenizer → model → post-processing in a Transformers pipeline. Why do tokens matter to a backend developer?
5. What do `model.eval()` and `torch.no_grad()` do, and why do serving endpoints always use them?
6. Which three PyTorch ideas let you read any training script?

---

# 31. Python vs Java for AI Development

## 31.1 Why Python dominates AI/ML

Not because the language is faster or "better" — Chapter 1 showed it is far slower at pure computation. The reasons are ecosystem and workflow:

1. **The stack is Python-first.** NumPy → SciPy → scikit-learn → PyTorch → Transformers → every model's reference implementation. Papers ship Python. New provider features ship in Python SDKs first. Bedrock, SageMaker and the AWS AI samples are Python-first.
2. **The heavy computation is not in Python.** CUDA kernels, BLAS, Rust (`pydantic-core`, `tokenizers`, uv), C++ (PyTorch core). Python is the control plane, so its slowness rarely matters where it counts.
3. **Exploration speed.** REPL + notebooks + dynamic typing + concise syntax = iterate on a prompt, a chunking strategy or a model in minutes. The JVM's compile step and verbosity are wrong for that loop.
4. **Glue is what AI apps mostly are.** Read documents, call APIs, reshape JSON, chain calls, evaluate outputs. Python is the best glue language on the market.
5. **Talent and community.** Data scientists and ML engineers write Python; hiring, hand-off and shared code favour Python for anything model-adjacent.

## 31.2 Why Java remains strong for backends

1. **Performance and predictability**: JIT, real multithreading, mature GC — for CPU-heavy, high-throughput services Java wins clearly.
2. **Type safety at scale**: large teams, long-lived codebases, refactoring safety, compile-time contracts.
3. **The Spring ecosystem**: security, transactions, observability, messaging, cloud integration — coherent, battle-tested, well staffed in enterprises.
4. **Operational maturity**: JVM tooling (profilers, heap analysis), platform threads/virtual threads, deployment patterns your organisation already runs.
5. **AI access is now good in Java**: Spring AI 2.x, LangChain4j 1.x, the AWS SDK's Bedrock Converse API, Anthropic/OpenAI Java SDKs. Calling a model is no longer a reason to switch languages.

```text
Java                                    Python
  ↓                                       ↓
Enterprise backend                      AI/ML models and research
Microservices, high-throughput APIs     Data science, notebooks
Transactions, security, integration     LLM ecosystem, RAG/agent frameworks
Spring ecosystem                        Rapid experimentation, glue
Long-lived, large-team codebases        Small focused services, pipelines, Lambdas
```

These are **tendencies, not rules**. FastAPI services run fine at serious scale (Instagram, Uber, Netflix run vast Python fleets); Java can do RAG. Choose per component.

## 31.3 Feature-by-feature

| Concern | Java | Python |
|---|---|---|
| Calling hosted LLMs (Anthropic/OpenAI/Bedrock) | Excellent (Spring AI, LangChain4j, SDKs) | Excellent, always first to get new features |
| RAG pipeline (loaders, chunking, embeddings, vector store) | Good (LangChain4j, Spring AI) | Best (LangChain, LlamaIndex, dozens of loaders) |
| Agents / tools | Good | Best; more frameworks, more examples |
| Running open models locally | Limited (ONNX Runtime, DJL, Jlama) | Native (Transformers, vLLM, Ollama) |
| Training / fine-tuning | Rare | Standard |
| Classical ML | Weak ecosystem | scikit-learn, XGBoost |
| Data wrangling | SQL or Spark | pandas, Polars |
| Evaluation tooling | Emerging | Mature (ragas, deepeval, LangSmith, Phoenix) |
| Type safety | Compile-time | Optional; mypy/pyright + Pydantic |
| Concurrency for I/O fan-out | Virtual threads, Reactor | asyncio |
| CPU parallelism | Threads | Processes / native libs (GIL) |
| Cold start (Lambda) | Seconds (SnapStart helps) | ~100–300 ms |
| Team fit | Backend teams | Data/ML teams, and backend teams doing AI |

## 31.4 Coexistence: the polyglot architecture

```text
                       Frontend
                          |
                          v
                   Java / Spring API           ← auth, business rules, transactions, existing domain
                          |
              +-----------+-----------+
              |                       |
              v                       v
      Java business logic       Python AI service (FastAPI)
      (orders, billing, ...)      - prompt construction, RAG, tools, agents
                                  - model routing / fallbacks
                                  - evaluation hooks, tracing
                                        |
                                        v
                               LLM / ML (Bedrock, Anthropic, local model)
```

**When it makes sense:**

- The AI logic is **non-trivial**: RAG with custom chunking/reranking, agent loops with many tools, local models, classical ML, or heavy use of Python-only libraries. You want the AI team to ship in the language the ecosystem speaks.
- **Different teams / release cadences**: the AI service changes daily (prompts, models); the core backend changes weekly.
- **Different runtime needs**: the AI service needs GPUs or a specific Python/CUDA stack; the backend does not.
- **Reuse**: several Java (or other) services need the same AI capability behind one API.

**When it does not:**

- The need is "call a model with a prompt and return text" — Spring AI / the AWS SDK in Java is simpler than adding a service, a deployment, a network hop and a second language to operate.
- The team is small and Java-only; operating a Python service you cannot debug is a liability.
- Latency budgets are tight and the hop adds meaningful overhead (usually it does not — the model call dominates).
- Strict transactional coupling between AI output and domain state — keep them in one process or design explicit compensation.

**How they talk:** REST/JSON with an OpenAPI contract (FastAPI generates it; Java clients can be generated from it), or gRPC for streaming/perf, or asynchronously through SQS/Kafka for batch jobs. Streaming tokens through a Java gateway means SSE/WebSocket passthrough (Project 6 shows the REST version).

**Rule of thumb:** *Put the model call where the prompt logic lives. If prompt logic is thin, put it in Java. If prompt logic is thick (RAG, tools, evaluation, Python-only libraries), give it a Python service.* Many real systems start with Java-only and extract a Python AI service when the AI part grows — that is a healthy path, not a failure.

## Knowledge check — Chapter 31

1. Give three concrete reasons Python dominates AI that have nothing to do with the speed of the interpreter.
2. When is "call Bedrock directly from Spring AI" the better design than a Python service? Name two conditions.
3. What does a Python AI service typically own in a polyglot architecture, and what stays in Java?
4. Which two operational costs does adding a Python service introduce, and how do you mitigate them?
5. State the rule of thumb for where the model call should live.

---

# Part VI — Real projects

---

# 32. Real Projects

Six projects, in increasing order of scope. Each one is complete enough to run, small enough to read in one sitting, and deliberately reuses the earlier chapters. Build them in order; each one assumes the previous.

---

## Project 1 — Python CLI

**Goal:** a task-tracker command-line tool. It teaches project structure, functions, classes, modules, packages and dependencies — the Python equivalent of your first Maven project with a `main` method, but idiomatic.

### Design

```text
tasks-cli/
├── pyproject.toml
├── README.md
├── src/
│   └── tasks_cli/
│       ├── __init__.py
│       ├── models.py        # Task dataclass
│       ├── storage.py       # JSON file persistence
│       ├── service.py       # use cases
│       └── cli.py           # Typer commands (the UI)
└── tests/
    ├── test_service.py
    └── test_cli.py
```

Dependencies: **Typer** (CLI framework driven by type hints — from FastAPI's author; the CLI equivalent of Spring Shell/picocli) and **Rich** (pretty terminal output). The standard library `argparse` works too; Typer shows off how type hints become a user interface.

```bash
uv init tasks-cli --python 3.14 && cd tasks-cli
uv add typer rich
uv add --dev pytest
```

```toml
# pyproject.toml — add:
[project.scripts]
tasks = "tasks_cli.cli:app"          # `uv run tasks ...` and, when installed, plain `tasks ...`

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### Code

```python
# src/tasks_cli/models.py
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from enum import StrEnum

class Priority(StrEnum):              # Python enums: members are singletons; StrEnum compares equal to its string value
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

@dataclass
class Task:
    id: int
    title: str
    priority: Priority = Priority.MEDIUM
    done: bool = False
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())

    def to_dict(self) -> dict:
        return asdict(self)

    @classmethod
    def from_dict(cls, data: dict) -> "Task":        # alternative constructor (Chapter 6)
        return cls(**{**data, "priority": Priority(data["priority"])})
```

```python
# src/tasks_cli/storage.py
import json
from pathlib import Path
from tasks_cli.models import Task

class JsonStorage:
    """Persists tasks to a JSON file. Small enough that a real DB would be over-engineering."""

    def __init__(self, path: Path):
        self.path = path

    def load(self) -> list[Task]:
        if not self.path.exists():
            return []
        return [Task.from_dict(d) for d in json.loads(self.path.read_text(encoding="utf-8"))]

    def save(self, tasks: list[Task]) -> None:
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.path.write_text(json.dumps([t.to_dict() for t in tasks], indent=2), encoding="utf-8")
```

```python
# src/tasks_cli/service.py
from tasks_cli.models import Priority, Task
from tasks_cli.storage import JsonStorage

class TaskNotFound(LookupError):
    pass

class TaskService:
    def __init__(self, storage: JsonStorage):
        self.storage = storage

    def add(self, title: str, priority: Priority = Priority.MEDIUM) -> Task:
        tasks = self.storage.load()
        next_id = max((t.id for t in tasks), default=0) + 1        # generator + default (Chapter 14)
        task = Task(id=next_id, title=title.strip(), priority=priority)
        self.storage.save([*tasks, task])
        return task

    def list(self, include_done: bool = False) -> list[Task]:
        tasks = self.storage.load()
        return [t for t in tasks if include_done or not t.done]

    def complete(self, task_id: int) -> Task:
        tasks = self.storage.load()
        for task in tasks:
            if task.id == task_id:
                task.done = True
                self.storage.save(tasks)
                return task
        raise TaskNotFound(task_id)
```

```python
# src/tasks_cli/cli.py
from pathlib import Path
from typing import Annotated
import typer
from rich.console import Console
from rich.table import Table
from tasks_cli.models import Priority
from tasks_cli.service import TaskNotFound, TaskService
from tasks_cli.storage import JsonStorage

app = typer.Typer(help="Tiny task tracker.", no_args_is_help=True)
console = Console()

def get_service() -> TaskService:
    data_file = Path(typer.get_app_dir("tasks-cli")) / "tasks.json"
    return TaskService(JsonStorage(data_file))

@app.command()
def add(title: str, priority: Annotated[Priority, typer.Option("--priority", "-p")] = Priority.MEDIUM) -> None:
    """Add a task."""
    task = get_service().add(title, priority)
    console.print(f"[green]Added[/] #{task.id}: {task.title} ({task.priority})")

@app.command("list")
def list_tasks(all: Annotated[bool, typer.Option("--all", "-a", help="Include completed tasks")] = False) -> None:
    """List tasks."""
    table = Table("id", "title", "priority", "done")
    for t in get_service().list(include_done=all):
        table.add_row(str(t.id), t.title, t.priority, "✓" if t.done else "")
    console.print(table)

@app.command()
def done(task_id: int) -> None:
    """Mark a task as completed."""
    try:
        task = get_service().complete(task_id)
    except TaskNotFound:
        console.print(f"[red]No task with id {task_id}[/]")
        raise typer.Exit(code=1)
    console.print(f"[green]Completed[/] #{task.id}: {task.title}")

if __name__ == "__main__":
    app()
```

```bash
uv run tasks add "Read chapter 14" -p high
uv run tasks add "Write tests"
uv run tasks list
uv run tasks done 1
uv run tasks --help          # generated from type hints + docstrings
```

### Tests

```python
# tests/test_service.py
import pytest
from tasks_cli.models import Priority
from tasks_cli.service import TaskNotFound, TaskService
from tasks_cli.storage import JsonStorage

@pytest.fixture
def service(tmp_path):                                   # tmp_path: fresh temp dir per test
    return TaskService(JsonStorage(tmp_path / "tasks.json"))

def test_add_assigns_incrementing_ids(service):
    a = service.add("first"); b = service.add("second", Priority.HIGH)
    assert (a.id, b.id) == (1, 2)
    assert b.priority is Priority.HIGH

def test_complete_hides_task_from_default_list(service):
    t = service.add("x")
    service.complete(t.id)
    assert service.list() == []
    assert [x.id for x in service.list(include_done=True)] == [t.id]

def test_complete_missing_raises(service):
    with pytest.raises(TaskNotFound):
        service.complete(42)
```

```python
# tests/test_cli.py
from typer.testing import CliRunner
from tasks_cli import cli

def test_add_and_list(tmp_path, monkeypatch):
    monkeypatch.setattr(cli.typer, "get_app_dir", lambda _: str(tmp_path))     # redirect data dir
    runner = CliRunner()
    assert runner.invoke(cli.app, ["add", "Buy milk", "-p", "low"]).exit_code == 0
    result = runner.invoke(cli.app, ["list"])
    assert "Buy milk" in result.stdout
```

### What it teaches

- A **package with layers as modules** (`models`, `storage`, `service`, `cli`) — Chapter 10 and 24.
- **Dataclass + enum** for the domain, **JSON** for persistence (Chapters 9, 12).
- **Type hints as UI**: Typer turns `priority: Priority` into a validated `--priority` option with completions and help — the same trick FastAPI uses for HTTP.
- **Dependencies and entry points**: `[project.scripts]` produces a real command; `uv build` produces an installable wheel.
- Tests inject a temp directory instead of mocking the filesystem — Chapter 18.

---

## Project 2 — REST API

**Goal:** the Customer API from Chapter 21, finished for deployment: PostgreSQL via Docker Compose, Alembic migrations, health/readiness, a Dockerfile, and a pagination envelope. If you built Chapter 21, this is the delta.

### Add to the project

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:17
    environment: {POSTGRES_USER: app, POSTGRES_PASSWORD: app, POSTGRES_DB: customers}
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck: {test: ["CMD-SHELL", "pg_isready -U app"], interval: 5s, retries: 10}
  api:
    build: .
    environment:
      APP_DATABASE_URL: postgresql+psycopg://app:app@db:5432/customers
    ports: ["8000:8000"]
    depends_on: {db: {condition: service_healthy}}
    command: ["sh", "-c", "alembic upgrade head && fastapi run src/customer_api/main.py --port 8000 --workers 2"]
volumes: {pgdata: {}}
```

Pagination envelope (a Pydantic *generic* model — Chapter 8/9):

```python
# src/customer_api/schemas.py — add
class Page[T](BaseModel):
    items: list[T]
    total: int
    limit: int
    offset: int
```

```python
# repository.py — add
def count(self, q: str | None) -> int:
    stmt = select(func.count()).select_from(Customer)
    if q: stmt = stmt.where(Customer.name.ilike(f"%{q}%"))
    return self.session.scalar(stmt) or 0

# api/customers.py — change list endpoint
@router.get("", response_model=Page[CustomerOut])
def list_customers(service: ServiceDep, q: ... = None, limit: ... = 20, offset: ... = 0):
    items = service.search(q, limit, offset)
    return Page(items=items, total=service.count(q), limit=limit, offset=offset)
```

Readiness (checks the DB) alongside liveness:

```python
@app.get("/ready", tags=["ops"])
def ready(session: Annotated[Session, Depends(get_session)]) -> dict[str, str]:
    session.execute(text("SELECT 1"))
    return {"status": "ready"}
```

Structured request logging middleware (Chapter 20) and JSON logs (Chapter 19) complete the production shape. Run `docker compose up --build`, open `http://localhost:8000/docs`.

### What it teaches

- The full Chapter 20–22 stack in one deployable unit: FastAPI + Pydantic + SQLAlchemy + Alembic + PostgreSQL + pytest.
- **Where Python differs from Spring** in practice: no container, no interfaces, explicit transactions, ~1/3 the files — with the same separation of concerns.
- Workers (`--workers 2`) as the answer to "how does this use my cores" (Chapter 17).

---

## Project 3 — AI Chat API

**Goal:** a FastAPI service that exposes a chat endpoint backed by an LLM SDK, with **conversation sessions**, **streaming** (SSE), a **provider-independent interface** with Anthropic and Bedrock implementations, and tests with a fake model.

```text
Client (curl / frontend / Java service)
  ↓  POST /chat  {session_id, message}         or  GET /chat/stream?session_id=&message=  (SSE)
FastAPI
  ↓  ChatService: loads history, builds messages, calls the model, stores turns
Python LLM SDK  (anthropic.AsyncAnthropic  |  boto3 bedrock-runtime via asyncio.to_thread)
  ↓
LLM
```

```text
ai-chat-api/
├── pyproject.toml
├── src/ai_chat/
│   ├── __init__.py
│   ├── main.py            # app, routes, lifespan
│   ├── config.py          # Settings
│   ├── schemas.py         # ChatRequest / ChatResponse
│   ├── llm.py             # LLM protocol + AnthropicLLM + BedrockLLM
│   ├── memory.py          # SessionStore (in-memory; DynamoDB variant sketched)
│   └── service.py         # ChatService
└── tests/
    ├── conftest.py
    └── test_chat.py
```

```bash
uv init ai-chat-api --python 3.14 && cd ai-chat-api
uv add "fastapi[standard]" pydantic-settings anthropic boto3
uv add --dev pytest pytest-asyncio httpx
```

### Code

```python
# src/ai_chat/config.py
from functools import lru_cache
from typing import Literal
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="CHAT_", extra="ignore")
    provider: Literal["anthropic", "bedrock", "fake"] = "anthropic"
    anthropic_model: str = "claude-opus-5"
    bedrock_model_id: str = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"
    aws_region: str = "us-east-1"
    system_prompt: str = "You are a helpful, concise assistant for a software company."
    max_tokens: int = 1024
    max_history_turns: int = 20
    api_key: SecretStr | None = None            # optional bearer token to protect the API

@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

```python
# src/ai_chat/schemas.py
from pydantic import BaseModel, Field

class ChatRequest(BaseModel):
    session_id: str = Field(min_length=1, max_length=64)
    message: str = Field(min_length=1, max_length=8000)

class ChatResponse(BaseModel):
    session_id: str
    reply: str
    input_tokens: int
    output_tokens: int
```

```python
# src/ai_chat/llm.py
from collections.abc import AsyncIterator
from dataclasses import dataclass
from typing import Protocol
import asyncio

Message = dict[str, str]                       # {"role": "user" | "assistant", "content": "..."}

@dataclass
class Completion:
    text: str
    input_tokens: int
    output_tokens: int

class LLM(Protocol):                           # the contract the service depends on (Chapter 7)
    async def complete(self, system: str, messages: list[Message], max_tokens: int) -> Completion: ...
    def stream(self, system: str, messages: list[Message], max_tokens: int) -> AsyncIterator[str]: ...


class AnthropicLLM:
    def __init__(self, model: str):
        from anthropic import AsyncAnthropic
        self.client, self.model = AsyncAnthropic(), model

    async def complete(self, system, messages, max_tokens) -> Completion:
        r = await self.client.messages.create(model=self.model, max_tokens=max_tokens, system=system, messages=messages)
        return Completion("".join(b.text for b in r.content if b.type == "text"), r.usage.input_tokens, r.usage.output_tokens)

    async def stream(self, system, messages, max_tokens) -> AsyncIterator[str]:
        async with self.client.messages.stream(model=self.model, max_tokens=max_tokens, system=system, messages=messages) as s:
            async for text in s.text_stream:
                yield text


class BedrockLLM:
    def __init__(self, model_id: str, region: str):
        import boto3
        from botocore.config import Config
        self.client = boto3.client("bedrock-runtime", region_name=region, config=Config(retries={"mode": "adaptive", "max_attempts": 5}))
        self.model_id = model_id

    @staticmethod
    def _to_bedrock(messages: list[Message]) -> list[dict]:
        return [{"role": m["role"], "content": [{"text": m["content"]}]} for m in messages]

    async def complete(self, system, messages, max_tokens) -> Completion:
        def call():                                                     # boto3 is blocking → run in a thread (Chapter 17)
            return self.client.converse(modelId=self.model_id, system=[{"text": system}],
                                        messages=self._to_bedrock(messages), inferenceConfig={"maxTokens": max_tokens})
        r = await asyncio.to_thread(call)
        return Completion(r["output"]["message"]["content"][0]["text"], r["usage"]["inputTokens"], r["usage"]["outputTokens"])

    async def stream(self, system, messages, max_tokens) -> AsyncIterator[str]:
        def start():
            return self.client.converse_stream(modelId=self.model_id, system=[{"text": system}],
                                               messages=self._to_bedrock(messages), inferenceConfig={"maxTokens": max_tokens})["stream"]
        events = await asyncio.to_thread(start)
        it = iter(events)
        while True:                                                     # pull each blocking event in a thread
            event = await asyncio.to_thread(next, it, None)
            if event is None:
                return
            if "contentBlockDelta" in event:
                yield event["contentBlockDelta"]["delta"].get("text", "")


class FakeLLM:                                                          # for tests and local dev without credentials
    async def complete(self, system, messages, max_tokens) -> Completion:
        return Completion(f"echo: {messages[-1]['content']}", 10, 5)
    async def stream(self, system, messages, max_tokens) -> AsyncIterator[str]:
        for word in f"echo: {messages[-1]['content']}".split():
            yield word + " "
```

```python
# src/ai_chat/memory.py
from collections import defaultdict
from ai_chat.llm import Message

class SessionStore:
    """In-memory history. Swap for DynamoDB (Chapter 29) or Redis in production — same two methods."""
    def __init__(self) -> None:
        self._sessions: dict[str, list[Message]] = defaultdict(list)

    def history(self, session_id: str) -> list[Message]:
        return list(self._sessions[session_id])

    def append(self, session_id: str, role: str, content: str) -> None:
        self._sessions[session_id].append({"role": role, "content": content})
```

```python
# src/ai_chat/service.py
from collections.abc import AsyncIterator
from ai_chat.config import Settings
from ai_chat.llm import LLM, Completion
from ai_chat.memory import SessionStore

class ChatService:
    def __init__(self, llm: LLM, store: SessionStore, settings: Settings):
        self.llm, self.store, self.settings = llm, store, settings

    def _context(self, session_id: str, user_message: str) -> list[dict[str, str]]:
        history = self.store.history(session_id)[-self.settings.max_history_turns * 2:]     # crude window; summarise in real apps
        return [*history, {"role": "user", "content": user_message}]

    async def reply(self, session_id: str, user_message: str) -> Completion:
        completion = await self.llm.complete(self.settings.system_prompt, self._context(session_id, user_message), self.settings.max_tokens)
        self.store.append(session_id, "user", user_message)
        self.store.append(session_id, "assistant", completion.text)
        return completion

    async def stream_reply(self, session_id: str, user_message: str) -> AsyncIterator[str]:
        parts: list[str] = []
        async for chunk in self.llm.stream(self.settings.system_prompt, self._context(session_id, user_message), self.settings.max_tokens):
            parts.append(chunk)
            yield chunk
        self.store.append(session_id, "user", user_message)
        self.store.append(session_id, "assistant", "".join(parts))
```

```python
# src/ai_chat/main.py
import json
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, Request, status
from fastapi.responses import StreamingResponse
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from ai_chat.config import Settings, get_settings
from ai_chat.llm import LLM, AnthropicLLM, BedrockLLM, FakeLLM
from ai_chat.memory import SessionStore
from ai_chat.schemas import ChatRequest, ChatResponse
from ai_chat.service import ChatService

def build_llm(settings: Settings) -> LLM:
    match settings.provider:                                     # Chapter 3: match as a factory switch
        case "anthropic": return AnthropicLLM(settings.anthropic_model)
        case "bedrock":   return BedrockLLM(settings.bedrock_model_id, settings.aws_region)
        case "fake":      return FakeLLM()

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    app.state.service = ChatService(build_llm(settings), SessionStore(), settings)   # singletons live here
    yield

app = FastAPI(title="AI Chat API", lifespan=lifespan)
bearer = HTTPBearer(auto_error=False)

def get_service(request: Request) -> ChatService:
    return request.app.state.service

def check_auth(creds: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer)],
               settings: Annotated[Settings, Depends(get_settings)]) -> None:
    if settings.api_key and (creds is None or creds.credentials != settings.api_key.get_secret_value()):
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "invalid api key")

ServiceDep = Annotated[ChatService, Depends(get_service)]

@app.post("/chat", response_model=ChatResponse, dependencies=[Depends(check_auth)])
async def chat(req: ChatRequest, service: ServiceDep):
    c = await service.reply(req.session_id, req.message)
    return ChatResponse(session_id=req.session_id, reply=c.text, input_tokens=c.input_tokens, output_tokens=c.output_tokens)

@app.post("/chat/stream", dependencies=[Depends(check_auth)])
async def chat_stream(req: ChatRequest, service: ServiceDep):
    async def events() -> AsyncIterator[str]:                                        # Server-Sent Events
        async for chunk in service.stream_reply(req.session_id, req.message):
            yield f"data: {json.dumps({'text': chunk})}\n\n"      # JSON-encode: chunks may contain newlines
        yield "event: done\ndata: {}\n\n"
    return StreamingResponse(events(), media_type="text/event-stream", headers={"Cache-Control": "no-cache"})

@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

```bash
CHAT_PROVIDER=fake uv run fastapi dev src/ai_chat/main.py
curl -s localhost:8000/chat -H 'content-type: application/json' -d '{"session_id":"s1","message":"hello"}'
curl -N localhost:8000/chat/stream -H 'content-type: application/json' -d '{"session_id":"s1","message":"stream this"}'
# real model: export ANTHROPIC_API_KEY=... ; CHAT_PROVIDER=anthropic uv run fastapi dev ...
# Bedrock:    CHAT_PROVIDER=bedrock AWS_PROFILE=dev uv run fastapi dev ...
```

### Tests

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from ai_chat.main import app, get_settings
from ai_chat.config import Settings

@pytest.fixture
def client(monkeypatch):
    monkeypatch.setenv("CHAT_PROVIDER", "fake")
    get_settings.cache_clear()                                  # lru_cache'd settings must be rebuilt for the new env
    with TestClient(app) as c:                                  # `with` runs lifespan → builds FakeLLM
        yield c

# tests/test_chat.py
def test_chat_roundtrip_keeps_history(client):
    r1 = client.post("/chat", json={"session_id": "s", "message": "hi"})
    assert r1.status_code == 200 and r1.json()["reply"] == "echo: hi"
    r2 = client.post("/chat", json={"session_id": "s", "message": "again"})
    assert r2.json()["reply"] == "echo: again"

def test_stream_emits_sse(client):
    with client.stream("POST", "/chat/stream", json={"session_id": "s", "message": "a b"}) as r:
        body = "".join(r.iter_text())
    assert body.startswith('data: {"text": "echo:') and "event: done" in body

def test_validation(client):
    assert client.post("/chat", json={"session_id": "", "message": "x"}).status_code == 422
```

### What it teaches

- **Protocol-based provider abstraction** (Chapter 7) — Anthropic and Bedrock behind one interface; `FakeLLM` makes tests and local dev credential-free.
- **Async correctly**: the Anthropic client is natively async; boto3 is wrapped with `asyncio.to_thread` (Chapter 17).
- **Streaming end-to-end**: async generator → `StreamingResponse` → SSE (Chapters 14, 20).
- **Singletons via lifespan + `app.state`**, per-request DI via `Depends`, auth as a dependency (Chapter 20).
- Statelessness of LLM APIs made explicit by the `SessionStore` — the thing Spring AI's `ChatMemory` hides.

---

## Project 4 — RAG Document Q&A

**Goal:** a document question-answering application built by hand — so you understand what Bedrock Knowledge Bases, LangChain and LlamaIndex automate. Pipeline:

```text
Ingestion (offline / CLI):
  documents (md, txt, pdf) → load → chunk → embed (Bedrock Titan v2) → store (PostgreSQL + pgvector)

Query (online / FastAPI):
  question → embed → nearest chunks (cosine) → prompt = context + question → LLM (Bedrock Converse) → answer + sources
```

```text
doc-qa/
├── pyproject.toml
├── docker-compose.yml           # postgres with pgvector
├── docs/                        # sample documents
├── src/doc_qa/
│   ├── __init__.py
│   ├── config.py
│   ├── db.py                    # engine, Chunk model (Vector column)
│   ├── loaders.py               # files → (text, source)
│   ├── chunking.py              # text → chunks
│   ├── embeddings.py            # Bedrock Titan embeddings (batched, threaded)
│   ├── store.py                 # add chunks / similarity search
│   ├── rag.py                   # prompt + generation
│   ├── ingest.py                # CLI: python -m doc_qa.ingest ./docs
│   └── main.py                  # FastAPI: /ask, /ingest
└── tests/
```

```bash
uv init doc-qa --python 3.14 && cd doc-qa
uv add "fastapi[standard]" pydantic-settings sqlalchemy "psycopg[binary]" pgvector boto3 pypdf
uv add --dev pytest
```

```yaml
# docker-compose.yml
services:
  db:
    image: pgvector/pgvector:pg17
    environment: {POSTGRES_USER: app, POSTGRES_PASSWORD: app, POSTGRES_DB: rag}
    ports: ["5432:5432"]
```

### Code

```python
# src/doc_qa/config.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="RAG_", extra="ignore")
    database_url: str = "postgresql+psycopg://app:app@localhost:5432/rag"
    aws_region: str = "us-east-1"
    embedding_model_id: str = "amazon.titan-embed-text-v2:0"
    embedding_dim: int = 1024
    chat_model_id: str = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"
    chunk_size: int = 800            # characters
    chunk_overlap: int = 120
    top_k: int = 5

@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

```python
# src/doc_qa/db.py
from pgvector.sqlalchemy import Vector
from sqlalchemy import Index, String, Text, create_engine, text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, sessionmaker
from doc_qa.config import get_settings

settings = get_settings()
engine = create_engine(settings.database_url)
SessionLocal = sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase): ...

class Chunk(Base):
    __tablename__ = "chunks"
    id: Mapped[int] = mapped_column(primary_key=True)
    source: Mapped[str] = mapped_column(String(512), index=True)
    position: Mapped[int]
    content: Mapped[str] = mapped_column(Text)
    embedding: Mapped[list[float]] = mapped_column(Vector(settings.embedding_dim))

    __table_args__ = (
        Index("ix_chunks_embedding_hnsw", "embedding", postgresql_using="hnsw",
              postgresql_with={"m": 16, "ef_construction": 64}, postgresql_ops={"embedding": "vector_cosine_ops"}),
    )

def init_db() -> None:                                   # demo; Alembic in a real project
    with engine.begin() as conn:
        conn.execute(text("CREATE EXTENSION IF NOT EXISTS vector"))
    Base.metadata.create_all(engine)
```

```python
# src/doc_qa/loaders.py
from collections.abc import Iterator
from pathlib import Path
from pypdf import PdfReader

def load_documents(root: Path) -> Iterator[tuple[str, str]]:
    """Yield (source, text) for every supported file under root — lazily (Chapter 14)."""
    for path in sorted(root.rglob("*")):
        if path.suffix.lower() in {".md", ".txt"}:
            yield str(path.relative_to(root)), path.read_text(encoding="utf-8", errors="ignore")
        elif path.suffix.lower() == ".pdf":
            pages = (page.extract_text() or "" for page in PdfReader(path).pages)
            yield str(path.relative_to(root)), "\n\n".join(pages)
```

```python
# src/doc_qa/chunking.py
def chunk_text(text: str, size: int, overlap: int) -> list[str]:
    """Split on paragraph boundaries, then pack paragraphs into ~size-char windows with overlap.

    Real systems use token-aware or semantic splitters; this shows the idea:
    chunks must be small enough to embed well and large enough to carry meaning.
    """
    paragraphs = [p.strip() for p in text.split("\n\n") if p.strip()]
    chunks: list[str] = []
    current = ""
    for para in paragraphs:
        if len(current) + len(para) + 2 <= size:
            current = f"{current}\n\n{para}" if current else para
            continue
        if current:
            chunks.append(current)
        # carry the tail of the previous chunk as overlap so context is not cut mid-idea
        tail = current[-overlap:] if overlap and current else ""
        current = f"{tail}\n\n{para}" if tail else para
        while len(current) > size:                       # a single huge paragraph: hard split
            chunks.append(current[:size]); current = current[size - overlap:]
    if current:
        chunks.append(current)
    return chunks
```

```python
# src/doc_qa/embeddings.py
import json
from concurrent.futures import ThreadPoolExecutor
import boto3
from botocore.config import Config
from doc_qa.config import get_settings

class BedrockEmbedder:
    def __init__(self) -> None:
        s = get_settings()
        self.client = boto3.client("bedrock-runtime", region_name=s.aws_region, config=Config(retries={"mode": "adaptive", "max_attempts": 8}))
        self.model_id, self.dim = s.embedding_model_id, s.embedding_dim

    def embed(self, text: str) -> list[float]:
        body = json.dumps({"inputText": text[:8000], "dimensions": self.dim, "normalize": True})
        resp = self.client.invoke_model(modelId=self.model_id, body=body, contentType="application/json", accept="application/json")
        return json.loads(resp["body"].read())["embedding"]

    def embed_many(self, texts: list[str], workers: int = 8) -> list[list[float]]:
        with ThreadPoolExecutor(max_workers=workers) as pool:         # I/O-bound → threads (Chapter 17)
            return list(pool.map(self.embed, texts))
```

```python
# src/doc_qa/store.py
from dataclasses import dataclass
from sqlalchemy import delete, select
from sqlalchemy.orm import Session
from doc_qa.db import Chunk

@dataclass
class Hit:
    source: str
    content: str
    score: float

class ChunkStore:
    def __init__(self, session: Session):
        self.session = session

    def replace_source(self, source: str, chunks: list[str], vectors: list[list[float]]) -> int:
        self.session.execute(delete(Chunk).where(Chunk.source == source))          # re-ingest = replace
        self.session.add_all(Chunk(source=source, position=i, content=c, embedding=v)
                             for i, (c, v) in enumerate(zip(chunks, vectors, strict=True)))
        self.session.commit()
        return len(chunks)

    def search(self, query_vec: list[float], k: int) -> list[Hit]:
        distance = Chunk.embedding.cosine_distance(query_vec).label("distance")
        rows = self.session.execute(select(Chunk, distance).order_by(distance).limit(k)).all()
        return [Hit(source=c.source, content=c.content, score=1 - d) for c, d in rows]
```

```python
# src/doc_qa/rag.py
from dataclasses import dataclass
import boto3
from doc_qa.config import get_settings
from doc_qa.embeddings import BedrockEmbedder
from doc_qa.store import ChunkStore, Hit

SYSTEM = """You answer questions using ONLY the provided context. If the context does not contain the answer,
say you don't know. Cite sources as [n] using the numbers of the context passages."""

@dataclass
class Answer:
    text: str
    sources: list[Hit]

class RagService:
    def __init__(self, embedder: BedrockEmbedder, store: ChunkStore):
        s = get_settings()
        self.embedder, self.store, self.k = embedder, store, s.top_k
        self.llm = boto3.client("bedrock-runtime", region_name=s.aws_region)
        self.model_id = s.chat_model_id

    def ask(self, question: str) -> Answer:
        hits = self.store.search(self.embedder.embed(question), self.k)                   # retrieval
        context = "\n\n".join(f"[{i + 1}] ({h.source})\n{h.content}" for i, h in enumerate(hits))
        prompt = f"Context:\n{context}\n\nQuestion: {question}"
        resp = self.llm.converse(                                                          # generation
            modelId=self.model_id, system=[{"text": SYSTEM}],
            messages=[{"role": "user", "content": [{"text": prompt}]}],
            inferenceConfig={"maxTokens": 800, "temperature": 0.1},
        )
        return Answer(resp["output"]["message"]["content"][0]["text"], hits)
```

```python
# src/doc_qa/ingest.py  —  uv run python -m doc_qa.ingest ./docs
import sys
from pathlib import Path
from doc_qa.chunking import chunk_text
from doc_qa.config import get_settings
from doc_qa.db import SessionLocal, init_db
from doc_qa.embeddings import BedrockEmbedder
from doc_qa.loaders import load_documents
from doc_qa.store import ChunkStore

def ingest(root: Path) -> None:
    s = get_settings()
    init_db()
    embedder = BedrockEmbedder()
    with SessionLocal() as session:
        store = ChunkStore(session)
        for source, text in load_documents(root):
            chunks = chunk_text(text, s.chunk_size, s.chunk_overlap)
            if not chunks:
                continue
            n = store.replace_source(source, chunks, embedder.embed_many(chunks))
            print(f"{source}: {n} chunks")

if __name__ == "__main__":
    ingest(Path(sys.argv[1]))
```

```python
# src/doc_qa/main.py
from typing import Annotated
from fastapi import Depends, FastAPI
from pydantic import BaseModel
from sqlalchemy.orm import Session
from doc_qa.db import SessionLocal
from doc_qa.embeddings import BedrockEmbedder
from doc_qa.rag import RagService
from doc_qa.store import ChunkStore

app = FastAPI(title="Doc Q&A")
embedder = BedrockEmbedder()                       # one client for the process

def get_session():
    with SessionLocal() as s:
        yield s

def get_rag(session: Annotated[Session, Depends(get_session)]) -> RagService:
    return RagService(embedder, ChunkStore(session))

class AskRequest(BaseModel):
    question: str

class SourceOut(BaseModel):
    source: str
    score: float
    snippet: str

class AskResponse(BaseModel):
    answer: str
    sources: list[SourceOut]

@app.post("/ask", response_model=AskResponse)
def ask(req: AskRequest, rag: Annotated[RagService, Depends(get_rag)]):    # sync def: boto3 + sync SQLAlchemy → threadpool
    a = rag.ask(req.question)
    return AskResponse(answer=a.text, sources=[SourceOut(source=h.source, score=round(h.score, 3), snippet=h.content[:200]) for h in a.sources])
```

```bash
docker compose up -d
uv run python -m doc_qa.ingest ./docs
uv run fastapi dev src/doc_qa/main.py
curl -s localhost:8000/ask -H 'content-type: application/json' -d '{"question":"How do I paginate the customers endpoint?"}' | jq
```

### Tests worth writing

```python
# tests/test_chunking.py — pure function, no infra
from doc_qa.chunking import chunk_text

def test_chunks_respect_size_and_overlap():
    text = "\n\n".join(f"Paragraph {i} " + "x" * 200 for i in range(10))
    chunks = chunk_text(text, size=500, overlap=50)
    assert all(len(c) <= 500 for c in chunks)
    assert chunks[1].startswith(chunks[0][-50:])          # overlap carried

# tests/test_rag.py — fake embedder + fake store, no AWS/DB
class FakeEmbedder:
    def embed(self, text): return [0.0] * 4
class FakeStore:
    def search(self, vec, k): return [Hit("a.md", "The customers endpoint accepts limit and offset.", 0.9)]

def test_prompt_contains_context_and_calls_model(monkeypatch):
    calls = {}
    class FakeBedrock:
        def converse(self, **kw):
            calls.update(kw); return {"output": {"message": {"content": [{"text": "Use limit and offset [1]."}]}}}
    svc = RagService.__new__(RagService)                   # bypass __init__ (avoids creating real clients)
    svc.embedder, svc.store, svc.k, svc.llm, svc.model_id = FakeEmbedder(), FakeStore(), 5, FakeBedrock(), "m"
    a = svc.ask("How do I paginate?")
    assert "[1] (a.md)" in calls["messages"][0]["content"][0]["text"]
    assert a.text.endswith("[1].") and a.sources[0].source == "a.md"
```

(The `__new__` trick works but signals the class should take its clients as constructor parameters — a good refactor to do yourself.)

### What it teaches

- **RAG mechanics without a framework**: loading, chunking, embeddings, vector storage, retrieval, prompt assembly, generation, citations.
- **pgvector with SQLAlchemy** — cosine distance ordering and an HNSW index; PostgreSQL knowledge carries over directly.
- **Generators for ingestion**, **threads for embedding fan-out**, **sync FastAPI endpoints** for blocking clients.
- What LangChain/LlamaIndex/Bedrock Knowledge Bases abstract: every line here maps to a `DocumentLoader`, `TextSplitter`, `Embeddings`, `VectorStore`, `Retriever`, prompt template and model — or to a Knowledge Base's data source, chunking strategy and `retrieve_and_generate`.

---

## Project 5 — AWS GenAI Application

**Goal:** a small but production-shaped AWS application: **support-ticket triage**. Tickets land in S3; a Lambda (or a container) reads each ticket, asks a Bedrock model for a **structured** analysis, stores the result in DynamoDB, and exposes a query endpoint. It includes configuration, error handling, retries, and local testing with `moto`.

```text
S3 (tickets/*.json)  ──event──▶  Lambda: triage handler
                                    │  1. read ticket from S3           (boto3 s3)
                                    │  2. build prompt + JSON schema
                                    │  3. bedrock.converse → structured JSON   (boto3 bedrock-runtime)
                                    │  4. validate with Pydantic
                                    │  5. put_item                       (boto3 dynamodb)
                                    ▼
                                 DynamoDB (ticket_id → category, urgency, summary)
                                    ▲
API Gateway / Lambda function URL: GET /tickets/{id}
```

```text
ticket-triage/
├── pyproject.toml
├── template.yaml           # AWS SAM (or CDK) — infra
├── src/triage/
│   ├── __init__.py
│   ├── config.py
│   ├── schemas.py          # Ticket, TriageResult
│   ├── llm.py              # Bedrock structured analysis
│   ├── repository.py       # DynamoDB
│   ├── handler.py          # Lambda entry points
│   └── local.py            # run the same code against a local file (dev)
└── tests/
```

```bash
uv add boto3 pydantic pydantic-settings
uv add --dev pytest "moto[s3,dynamodb]" "boto3-stubs[bedrock-runtime,s3,dynamodb]"
```

### Code

```python
# src/triage/config.py
from functools import lru_cache
from pydantic_settings import BaseSettings

class Settings(BaseSettings):                   # in Lambda: plain environment variables from the template
    aws_region: str = "us-east-1"
    model_id: str = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"
    table_name: str = "tickets"
    max_tokens: int = 600

@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

```python
# src/triage/schemas.py
from typing import Literal
from pydantic import BaseModel, Field

class Ticket(BaseModel):
    id: str
    customer_email: str
    subject: str
    body: str

class TriageResult(BaseModel):
    category: Literal["billing", "technical", "account", "other"]
    urgency: int = Field(ge=1, le=5)
    summary: str = Field(max_length=300)
    suggested_reply: str
```

```python
# src/triage/llm.py
import json
import logging
import boto3
from botocore.config import Config
from botocore.exceptions import ClientError
from pydantic import ValidationError
from triage.config import get_settings
from triage.schemas import Ticket, TriageResult

log = logging.getLogger(__name__)
_settings = get_settings()
_bedrock = boto3.client("bedrock-runtime", region_name=_settings.aws_region,
                        config=Config(retries={"mode": "adaptive", "max_attempts": 6}, read_timeout=60))

SYSTEM = "You are a support triage assistant. Respond ONLY by calling the record_triage tool."

TOOL = {"toolSpec": {
    "name": "record_triage",
    "description": "Record the triage analysis of a support ticket.",
    "inputSchema": {"json": TriageResult.model_json_schema()},          # Pydantic → JSON Schema → tool schema
}}

class TriageError(Exception): ...
class TransientTriageError(TriageError): ...        # caller may retry (Lambda will, on raise)

def triage(ticket: Ticket) -> TriageResult:
    prompt = f"Subject: {ticket.subject}\n\nBody:\n{ticket.body}"
    try:
        resp = _bedrock.converse(
            modelId=_settings.model_id,
            system=[{"text": SYSTEM}],
            messages=[{"role": "user", "content": [{"text": prompt}]}],
            toolConfig={"tools": [TOOL], "toolChoice": {"tool": {"name": "record_triage"}}},   # force the structured tool call
            inferenceConfig={"maxTokens": _settings.max_tokens, "temperature": 0},
        )
    except _bedrock.exceptions.ThrottlingException as e:
        raise TransientTriageError("bedrock throttled") from e
    except _bedrock.exceptions.ModelNotReadyException as e:
        raise TransientTriageError("model not ready") from e
    except ClientError as e:                                            # validation, access denied, ... → permanent
        code = e.response["Error"]["Code"]
        raise TriageError(f"bedrock error {code}: {e.response['Error']['Message']}") from e

    for block in resp["output"]["message"]["content"]:
        if "toolUse" in block:
            try:
                return TriageResult.model_validate(block["toolUse"]["input"])   # validate the model's JSON
            except ValidationError as e:
                raise TriageError(f"model returned invalid triage: {e}") from e
    raise TriageError(f"no tool call in response (stopReason={resp['stopReason']})")
```

```python
# src/triage/repository.py
import boto3
from triage.config import get_settings
from triage.schemas import TriageResult

_table = boto3.resource("dynamodb", region_name=get_settings().aws_region).Table(get_settings().table_name)

def save(ticket_id: str, result: TriageResult) -> None:
    _table.put_item(Item={"ticket_id": ticket_id, **result.model_dump()})

def get(ticket_id: str) -> dict | None:
    return _table.get_item(Key={"ticket_id": ticket_id}).get("Item")
```

```python
# src/triage/handler.py
import json
import logging
import boto3
from triage import repository
from triage.llm import TransientTriageError, TriageError, triage
from triage.schemas import Ticket

log = logging.getLogger()
log.setLevel(logging.INFO)
s3 = boto3.client("s3")

def on_s3_event(event, context):
    """Triggered by S3 ObjectCreated. Processes every record; transient errors raise → Lambda retry / DLQ."""
    for record in event["Records"]:
        bucket, key = record["s3"]["bucket"]["name"], record["s3"]["object"]["key"]
        ticket = Ticket.model_validate_json(s3.get_object(Bucket=bucket, Key=key)["Body"].read())
        try:
            result = triage(ticket)
        except TransientTriageError:
            log.warning("transient failure for %s; will retry", ticket.id)
            raise
        except TriageError:
            log.exception("permanent failure for %s", ticket.id)     # log with traceback, skip this record
            continue
        repository.save(ticket.id, result)
        log.info("triaged %s → %s/%s", ticket.id, result.category, result.urgency)

def get_ticket(event, context):
    """Function URL / API Gateway: GET /tickets/{id}"""
    ticket_id = event["pathParameters"]["id"]
    item = repository.get(ticket_id)
    if item is None:
        return {"statusCode": 404, "body": json.dumps({"detail": "not found"})}
    return {"statusCode": 200, "headers": {"content-type": "application/json"}, "body": json.dumps(item, default=str)}
```

```yaml
# template.yaml (AWS SAM) — the infra, abbreviated
Resources:
  TicketsTable:
    Type: AWS::DynamoDB::Table
    Properties: {BillingMode: PAY_PER_REQUEST, AttributeDefinitions: [{AttributeName: ticket_id, AttributeType: S}], KeySchema: [{AttributeName: ticket_id, KeyType: HASH}]}
  TriageFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.13
      Handler: triage.handler.on_s3_event
      CodeUri: .
      Timeout: 120
      Environment: {Variables: {MODEL_ID: us.anthropic.claude-sonnet-4-5-20250929-v1:0, TABLE_NAME: !Ref TicketsTable}}
      Policies:
        - DynamoDBCrudPolicy: {TableName: !Ref TicketsTable}
        - S3ReadPolicy: {BucketName: !Ref TicketsBucket}
        - Statement: [{Effect: Allow, Action: [bedrock:InvokeModel, bedrock:InvokeModelWithResponseStream], Resource: "*"}]
      Events:
        Upload: {Type: S3, Properties: {Bucket: !Ref TicketsBucket, Events: s3:ObjectCreated:*, Filter: {S3Key: {Rules: [{Name: prefix, Value: tickets/}]}}}}
```

### Tests with moto (no AWS account needed)

```python
# tests/test_handler.py
import json, os
import boto3, pytest
from moto import mock_aws

@pytest.fixture(autouse=True)
def aws_env(monkeypatch):
    for k, v in {"AWS_ACCESS_KEY_ID": "test", "AWS_SECRET_ACCESS_KEY": "test", "AWS_DEFAULT_REGION": "us-east-1", "TABLE_NAME": "tickets"}.items():
        monkeypatch.setenv(k, v)

@mock_aws
def test_s3_event_triages_and_stores(monkeypatch):
    s3 = boto3.client("s3"); s3.create_bucket(Bucket="b")
    ddb = boto3.resource("dynamodb")
    ddb.create_table(TableName="tickets", BillingMode="PAY_PER_REQUEST",
                     AttributeDefinitions=[{"AttributeName": "ticket_id", "AttributeType": "S"}],
                     KeySchema=[{"AttributeName": "ticket_id", "KeyType": "HASH"}])
    s3.put_object(Bucket="b", Key="tickets/1.json", Body=json.dumps(
        {"id": "T-1", "customer_email": "a@x.io", "subject": "Charged twice", "body": "I was billed two times."}))

    from triage import handler, repository
    from triage.schemas import TriageResult
    monkeypatch.setattr(handler, "triage", lambda t: TriageResult(category="billing", urgency=4, summary="double charge", suggested_reply="Sorry..."))
    repository._table = ddb.Table("tickets")                 # point the module's table at the mocked one

    handler.on_s3_event({"Records": [{"s3": {"bucket": {"name": "b"}, "object": {"key": "tickets/1.json"}}}]}, None)
    assert repository.get("T-1")["category"] == "billing"
```

Bedrock itself is patched (moto does not run models); the `triage()` function gets its own unit test with a fake client returning a canned `toolUse` block, and one *live* smoke test marked `@pytest.mark.integration` that you run manually with real credentials.

### What it teaches

- The **boto3 patterns in real code**: module-level clients, adaptive retries, `ClientError` classification into transient vs permanent, paginated S3, DynamoDB resource API.
- **Structured output via forced tool use** on Bedrock Converse with a schema generated from Pydantic — the most reliable way to get JSON from any Bedrock model — and validating it before trusting it.
- **Lambda shape**: handler functions, environment-based config, retries/DLQ by raising, cold-start-friendly imports.
- **Testing AWS code offline** with moto + monkeypatch.
- Reading the SAM template: IAM policies for Bedrock (`bedrock:InvokeModel`) are what the exam and real deployments require.

---

## Project 6 — Python AI Service + Java Backend

**Goal:** see the two languages coexist. A Spring Boot application owns customers and orders; it delegates one AI capability — "summarise this customer's recent activity for a support agent" — to a Python FastAPI service that talks to Bedrock.

```text
Java Spring Boot  (customer-service)
        |  REST: POST /internal/summaries  {customer, orders[]}   (JSON, API key, 10 s timeout, retry ×2)
        v
Python FastAPI    (ai-summary-service)
        |  prompt template + Bedrock Converse (+ later: RAG over support docs, tools, evaluation)
        v
Amazon Bedrock / LLM
```

### The Python service

```text
ai-summary-service/
├── pyproject.toml
└── src/ai_summary/
    ├── main.py
    ├── config.py          # AISUMMARY_API_KEY, AISUMMARY_MODEL_ID, AWS region
    ├── schemas.py         # contract shared with Java
    └── summarizer.py
```

```python
# src/ai_summary/schemas.py — this IS the contract; FastAPI publishes it at /openapi.json
from datetime import date
from pydantic import BaseModel, Field

class OrderIn(BaseModel):
    id: str
    placed_on: date
    total_cents: int
    status: str

class SummaryRequest(BaseModel):
    customer_name: str
    plan: str
    orders: list[OrderIn] = Field(max_length=100)
    open_tickets: list[str] = []

class SummaryResponse(BaseModel):
    summary: str
    risk_level: str                 # "low" | "medium" | "high"
    model_id: str
    input_tokens: int
    output_tokens: int
```

```python
# src/ai_summary/summarizer.py
import boto3
from botocore.config import Config
from ai_summary.config import get_settings
from ai_summary.schemas import SummaryRequest, SummaryResponse

SYSTEM = """You write 3-sentence briefings for support agents about a customer. Be factual; no speculation.
Finish with a line 'Risk: low|medium|high' based on cancellations, refunds and open tickets."""

class Summarizer:
    def __init__(self) -> None:
        s = get_settings()
        self.client = boto3.client("bedrock-runtime", region_name=s.aws_region, config=Config(retries={"mode": "adaptive", "max_attempts": 4}, read_timeout=20))
        self.model_id = s.model_id

    def summarize(self, req: SummaryRequest) -> SummaryResponse:
        orders = "\n".join(f"- {o.placed_on} {o.id}: {o.total_cents / 100:.2f} ({o.status})" for o in req.orders) or "- none"
        tickets = "\n".join(f"- {t}" for t in req.open_tickets) or "- none"
        prompt = f"Customer: {req.customer_name} (plan: {req.plan})\nOrders:\n{orders}\nOpen tickets:\n{tickets}"
        resp = self.client.converse(modelId=self.model_id, system=[{"text": SYSTEM}],
                                    messages=[{"role": "user", "content": [{"text": prompt}]}],
                                    inferenceConfig={"maxTokens": 400, "temperature": 0.2})
        text = resp["output"]["message"]["content"][0]["text"]
        risk = next((lvl for lvl in ("high", "medium", "low") if f"risk: {lvl}" in text.lower()), "low")
        return SummaryResponse(summary=text, risk_level=risk, model_id=self.model_id,
                               input_tokens=resp["usage"]["inputTokens"], output_tokens=resp["usage"]["outputTokens"])
```

```python
# src/ai_summary/main.py
from typing import Annotated
from fastapi import Depends, FastAPI, Header, HTTPException, Request
from ai_summary.config import Settings, get_settings
from ai_summary.schemas import SummaryRequest, SummaryResponse
from ai_summary.summarizer import Summarizer

app = FastAPI(title="AI Summary Service", version="1.0.0")
summarizer = Summarizer()

def require_api_key(settings: Annotated[Settings, Depends(get_settings)],
                    x_api_key: Annotated[str | None, Header()] = None) -> None:
    if x_api_key != settings.api_key.get_secret_value():
        raise HTTPException(401, "invalid api key")

@app.post("/internal/summaries", response_model=SummaryResponse, dependencies=[Depends(require_api_key)])
def summaries(req: SummaryRequest) -> SummaryResponse:          # sync: boto3 → threadpool
    return summarizer.summarize(req)

@app.get("/health")
def health(): return {"status": "ok"}
```

### The Java side

```java
// CustomerSummaryClient.java — Spring Boot 3.5, RestClient
@Component
public class CustomerSummaryClient {

    private final RestClient http;

    public CustomerSummaryClient(RestClient.Builder builder,
                                 @Value("${ai.summary.base-url}") String baseUrl,
                                 @Value("${ai.summary.api-key}") String apiKey) {
        this.http = builder.baseUrl(baseUrl)          // timeouts come from spring.http.client.* (see application.yml)
                .defaultHeader("X-API-Key", apiKey)
                .build();
    }

    public record OrderIn(String id, LocalDate placedOn, long totalCents, String status) {}
    public record SummaryRequest(String customerName, String plan, List<OrderIn> orders, List<String> openTickets) {}
    public record SummaryResponse(String summary, String riskLevel, String modelId, int inputTokens, int outputTokens) {}

    @Retry(name = "aiSummary")                     // resilience4j: retry transient 5xx / timeouts
    @CircuitBreaker(name = "aiSummary", fallbackMethod = "fallback")
    public SummaryResponse summarize(SummaryRequest request) {
        return http.post().uri("/internal/summaries")
                .contentType(MediaType.APPLICATION_JSON)
                .body(request)
                .retrieve()
                .body(SummaryResponse.class);
    }

    SummaryResponse fallback(SummaryRequest request, Throwable t) {
        return new SummaryResponse("Summary temporarily unavailable.", "unknown", null, 0, 0);
    }
}
```

```yaml
# application.yml
ai.summary.base-url: http://ai-summary:8000
ai.summary.api-key: ${AI_SUMMARY_API_KEY}
spring.http.client.connect-timeout: 2s
spring.http.client.read-timeout: 15s                     # model calls are slow: budget seconds, not milliseconds
spring.jackson.property-naming-strategy: SNAKE_CASE     # Python side uses snake_case field names
resilience4j.retry.instances.aiSummary: {max-attempts: 2, wait-duration: 500ms}
```

```java
@Service
public class SupportBriefingService {
    private final CustomerRepository customers; private final OrderRepository orders; private final CustomerSummaryClient ai;
    ...
    public String briefing(long customerId) {
        var c = customers.findById(customerId).orElseThrow();
        var recent = orders.findTop20ByCustomerIdOrderByPlacedOnDesc(customerId).stream()
                .map(o -> new OrderIn(o.getId(), o.getPlacedOn(), o.getTotalCents(), o.getStatus().name())).toList();
        return ai.summarize(new SummaryRequest(c.getName(), c.getPlan(), recent, List.of())).summary();
    }
}
```

```yaml
# docker-compose.yml (both services)
services:
  ai-summary:
    build: ./ai-summary-service
    environment: {AISUMMARY_API_KEY: dev-key, AISUMMARY_MODEL_ID: amazon.nova-pro-v1:0, AWS_REGION: us-east-1}
    volumes: ["~/.aws:/root/.aws:ro"]          # local creds; in ECS/EKS use a task role / IRSA instead
    ports: ["8000:8000"]
  customer-service:
    build: ./customer-service
    environment: {AI_SUMMARY_API_KEY: dev-key, AI_SUMMARY_BASE_URL: http://ai-summary:8000}
    ports: ["8080:8080"]
    depends_on: [ai-summary]
```

Generate a typed Java client from the Python service's `/openapi.json` (openapi-generator) if the contract grows; for a single endpoint, records + RestClient are enough.

### Why this architecture — and when not

**Choose the Python service when:**

- The AI logic will grow beyond one prompt: RAG over support docs, tool calls into other systems, model routing, evaluation datasets, prompt experiments in notebooks. All of that is *cheaper to build and iterate in Python*.
- An AI/data team owns it and deploys on its own cadence; the Java team consumes a stable contract.
- It needs Python-only dependencies (local embedding models, classical ML, LlamaIndex loaders) or a different runtime (GPU).
- Multiple consumers (Java, batch jobs, a chatbot) need the same capability.

**Call the model directly from Java (Spring AI / AWS SDK) when:**

- The capability is *one prompt → one answer* with no retrieval or tools. A network hop, a deployment, an API key and a second language are not worth it for `chatClient.prompt().user(...).call()`.
- The team is Java-only and would not be able to debug or evolve the Python service.
- You need the AI result inside a JPA transaction or strict latency budgets where the extra hop is measurable (rare — model latency dominates).

**Operational details that matter in both cases:** timeouts sized for model latency (seconds, not milliseconds); retries only on transient failures (5xx/timeouts/throttles), never on 4xx; a circuit breaker with a graceful fallback; API key or mTLS between services; structured logs with a shared request id (propagate `X-Request-ID` from Java to Python and into the log middleware from Chapter 20); cost/usage metrics (`input_tokens`, `output_tokens`) emitted by the Python side and aggregated centrally.

### What it teaches

- A **contract-first boundary** between Java and Python: Pydantic on one side, records + Jackson on the other, OpenAPI in the middle.
- Where each language is strong: Java holds domain data and transactions; Python holds prompt logic and the AI stack.
- The real trade-off: one more service to run versus the freedom to evolve AI logic in the language the ecosystem speaks. You can now make that call deliberately.

---

# Part VII — Becoming productive

---

# 33. Common Mistakes Java Developers Make in Python

Each mistake: what it looks like, why it happens, and the Pythonic alternative. You will recognise yourself in several of these; that is the point.

### 1. Writing Java syntax in Python

```python
# Java accent
class Main:
    @staticmethod
    def main():
        i = 0
        while (i < len(items)):
            System_out_println(items[i]); i += 1
Main.main()
```

**Why:** muscle memory. **Instead:** module-level code with `if __name__ == "__main__":`, `for item in items:`, `print()`. Read Chapter 13 again; let ruff nag you.

### 2. Creating unnecessary classes

`class StringUtils`, `class PriceCalculator` with one method, `class Config` holding static constants, a `Handler` class for every event type. **Why:** in Java, code must live in a class. **Instead:** functions in modules; constants at module level; a dict of functions or `match` for dispatch. Create a class when you have *state plus behaviour* or need polymorphism. "Would this class have zero fields?" → it is a module.

### 3. Getters and setters everywhere

`def get_name(self): return self._name` × 20 fields. **Why:** Java needs them for evolvability. **Instead:** public attributes; `@property` only when read/write logic appears (Chapter 6). A dataclass with public fields *is* the idiomatic DTO.

### 4. Overusing inheritance

`AbstractBaseRepository → AbstractSqlRepository → CustomerRepository`, mixins for everything, interfaces as ABCs for every dependency. **Why:** it is the Java structuring tool. **Instead:** composition, functions, `Protocol` for contracts; inherit from framework bases (`BaseModel`, `DeclarativeBase`) and from your one `AppError`, and rarely otherwise.

### 5. Treating type hints like Java types

Assuming `def f(x: int)` rejects strings; over-annotating every local; using `Any` to "make the checker shut up"; casting with `int(x)` because the hint says int. **Why:** hints look like declarations. **Instead:** remember they are checked by mypy/pyright, not the interpreter (Chapter 8); validate at boundaries with Pydantic; annotate signatures, let locals infer; treat a checker error like a compile error.

### 6. Misunderstanding mutable objects — the shared default

```python
def add_item(item, items=[]): ...          # one list for all calls
class Cart: items = []                       # one list for all carts
```

**Why:** in Java, `new ArrayList<>()` in a field initialiser runs per instance. **Instead:** `items=None` + `if items is None: items = []`; `field(default_factory=list)`; per-instance state in `__init__` (Chapters 5, 6, 9).

### 7. Confusing references and copies

`b = a` then mutating `b` changes `a`; `copy()` of a nested structure sharing inner lists; sorting a list a caller still holds. **Why:** Java's `final` fields and defensive copying habits are inconsistent, and Python has no `new` to signal allocation. **Instead:** think "every name is a reference to a heap object"; use `list(x)`/`x.copy()`/`copy.deepcopy`; prefer returning new objects (`sorted(x)` over `x.sort()`) at API boundaries; use tuples/frozen dataclasses for values.

### 8. Ignoring virtual environments

`pip install` globally, `sudo pip`, running the system interpreter, "works on my machine". **Why:** Maven made isolation invisible. **Instead:** every project has `.venv` + `pyproject.toml` + `uv.lock`; `uv run` everything (Chapter 2). Never touch the OS Python.

### 9. Global state used carelessly

Module-level mutable dicts as caches/registries mutated from everywhere; `global` inside functions; clients created at import with environment read at import time. **Why:** Java's static fields feel similar but are at least namespaced by class and initialised lazily. **Instead:** module-level *immutable* constants and *singletons created lazily* (`lru_cache`'d factories, lifespan); pass dependencies explicitly; no import-time side effects (Chapter 10).

### 10. Misunderstanding `async`

Adding `async def` to every endpoint "for performance", then calling `requests`/boto3/`time.sleep` inside it; awaiting calls sequentially in a loop when `gather` was the point; mixing sync SQLAlchemy sessions into async code. **Why:** `async` looks like `@Async`/`CompletableFuture`. **Instead:** Chapter 17's decision guide. `async` only with async libraries; `def` endpoints for blocking ones; `gather`/`TaskGroup` for fan-out; `asyncio.to_thread` for the odd blocking call.

### 11. Misunderstanding the GIL

Either "Python threads are useless" (so everything is sequential) or "add threads to make CPU work faster" (no gain). **Why:** it is genuinely different from the JVM. **Instead:** threads for I/O-bound blocking work (they release the GIL), processes/native libraries for CPU work, asyncio for high-concurrency I/O with async libraries; worker processes for web servers.

### 12. Writing overly clever Python

Nested comprehensions, `lambda` chains, `reduce`, walrus everywhere, dunder tricks — as an over-correction after learning "Pythonic". **Why:** the idioms are fun. **Instead:** readability is the standard, not brevity (Chapter 13). If it needs a comment to explain *how*, unroll it.

### 13. Using frameworks for problems that don't need them

Reaching for LangChain to make one API call; Django for a 3-endpoint internal tool; a DI framework for FastAPI; Celery for a job that `BackgroundTasks` or a cron handles. **Why:** Spring conditions you to expect a framework for everything. **Instead:** start with the standard library and the provider SDK; add a framework when you feel the specific pain it solves. Python's strength is that simple things stay simple.

### 14. (Bonus) Other classic slips

- `except:` / `except Exception: pass` swallowing everything — catch specific exceptions (Chapter 11).
- `if x == None` / `if len(xs) == 0` / `== True` — use `is None`, truthiness.
- `x = xs.sort()` — `sort` returns `None`; use `sorted`.
- Naming a module `json.py`, `test.py`, `logging.py` — shadows the stdlib.
- Circular imports from Java-style "one class per file with everything imported at the top" — group related code per module, import lazily inside functions if needed.
- f-strings in logging calls, `print` instead of logging in services.
- Running `python path/to/module.py` inside a package instead of `python -m package.module` (Chapter 10).
- `is` for value comparisons (`x is 5` works by accident for small ints — and fails for 1000).
- Forgetting `self`, forgetting `super().__init__()`, forgetting `()` when calling a method (`if user.is_active:` on a *method* is always truthy).
- Iterating a dict while modifying it; expecting `dict.keys()` to be a list.
- Reusing an exhausted generator.

---

# 34. What a Java Developer Should NOT Learn Yet

Python's surface is huge. Focus is the skill. Use this as your "not now" list.

### Must know — professional Python and AI development

- Execution model, dynamic/strong typing, duck typing, the GIL (Chapters 1, 17)
- uv, venvs, `pyproject.toml`, ruff, a type checker (Chapters 2, 8, 23)
- Core syntax, collections, slicing/unpacking/comprehensions (Chapters 3, 4)
- Functions, closures, `*args/**kwargs`, decorators, context managers, generators (Chapters 5, 14–16)
- The object model: `self`, `__init__`, class vs instance attributes, dunder basics, `@property`, `@dataclass` (Chapters 6, 9)
- Protocol vs ABC vs duck typing (Chapter 7)
- Type hints and Pydantic v2 (Chapters 8, 9)
- Modules, packages, `__main__`, `src/` layout (Chapters 10, 24)
- Exceptions, EAFP (Chapter 11)
- Files, JSON, httpx, pathlib, env vars (Chapter 12)
- asyncio fundamentals and the sync/async decision (Chapter 17)
- pytest: fixtures, parametrize, mocking, TestClient (Chapter 18)
- logging + pydantic-settings (Chapter 19)
- FastAPI: routes, models, Depends, async endpoints, exception handlers (Chapters 20, 21)
- SQLAlchemy 2.0 ORM + Alembic basics (Chapter 22)
- NumPy arrays, pandas DataFrames at reading level (Chapter 25)
- Calling LLMs with a provider SDK and boto3/Bedrock Converse; streaming; structured output; tool use (Chapters 28, 29)
- Embeddings, vector search, RAG mechanics (Project 4)

### Should know — useful ecosystem knowledge

- `match` statements, `enum`, `functools`, `itertools`, `contextvars`
- Poetry and `requirements.txt` well enough to work in other people's repos
- `typing` extras: `TypedDict`, `Literal`, `Annotated`, generics syntax, `override`
- SQLAlchemy async, eager loading strategies, Core for bulk operations
- scikit-learn's estimator API; matplotlib at reading level; Jupyter workflow
- LangChain 1.x concepts (models, prompts, tools, `create_agent`, retrievers) and LlamaIndex's data path
- Transformers `pipeline`, tokenizers, sentence-transformers; PyTorch tensor/module/training-loop vocabulary
- Bedrock Knowledge Bases, Guardrails, Agents from the boto3 side; OpenSearch k-NN; DynamoDB for chat state
- `moto`, `testcontainers`, `respx`, `pytest-asyncio`
- structlog, OpenTelemetry for Python, tracing LLM calls (LangSmith/Langfuse/Phoenix)
- Packaging and publishing a wheel; Docker best practices for uv

### Can learn later — not necessary initially

- Metaclasses, descriptors, `__slots__`, `__getattr__` magic, `__init_subclass__` — framework-author territory
- Free-threaded Python, `concurrent.interpreters`, C extensions, Cython, Rust bindings (PyO3)
- asyncio internals (custom event loops, protocols/transports), `trio`/`anyio` beyond what FastAPI needs
- Django (unless your company runs it), Flask beyond reading
- Advanced SQLAlchemy (custom types, polymorphic mappings, query caching, sharding)
- Advanced pandas (multi-index, window functions), Polars, Dask, Spark
- Training and fine-tuning models, `peft`/LoRA, `accelerate`, distributed training, CUDA
- Deep LangGraph (custom graphs, checkpointers, multi-agent topologies) until you have an agent problem
- Building your own vector database or embedding models
- `t-strings`, `typing` esoterica (`ParamSpec`, `TypeVarTuple`, variance), plugin systems
- Packaging with C extensions, conda ecosystems, manylinux wheels

If a tutorial requires something from the last list to do something from the first, find a different tutorial.

---

# 35. Practical Learning Path

```text
Python fundamentals            (Ch 1–4)     ── 3–4 days
       ↓
Python object model            (Ch 6–7)     ── 2 days
       ↓
Functions / modules / packages (Ch 5, 10)   ── 2 days
       ↓
Type hints                     (Ch 8)       ── 1 day
       ↓
Dataclasses / Pydantic         (Ch 9)       ── 1 day
       ↓
Pythonic programming           (Ch 11–16)   ── 3 days   → Project 1 (CLI)
       ↓
Async / concurrency            (Ch 17)      ── 2 days
       ↓
pytest                         (Ch 18)      ── 1 day
       ↓
FastAPI                        (Ch 19–20)   ── 2 days
       ↓
SQLAlchemy                     (Ch 21–22)   ── 2 days   → Project 2 (REST API)
       ↓
NumPy / pandas basics          (Ch 25–26)   ── 2 days
       ↓
LLM APIs                       (Ch 27–28)   ── 2 days   → Project 3 (Chat API)
       ↓
boto3 / AWS                    (Ch 29)      ── 2 days   → Project 5 (AWS GenAI)
       ↓
GenAI frameworks               (Ch 30–31)   ── 3 days   → Project 4 (RAG)
       ↓
AI projects                    (Ch 32)      ── ongoing  → Project 6 (Java + Python)
```

**What to practise at each stage**

| Stage | Practise this |
|---|---|
| Fundamentals | Rewrite three small Java utilities you know by heart (string parsing, a `Map` aggregation, a file reader) in Python. Use the REPL constantly. Run `dis.dis` once on a function to see bytecode. |
| Object model | Write a class with `__repr__`, `__eq__`, a `@property`, a `@classmethod` constructor. Then delete half of it and replace with a dataclass. Explain `self` to a rubber duck. |
| Functions/modules | Build a package with three modules and a `__main__` entry; run it with `-m`. Write a closure-based "configured function" and a decorator. |
| Type hints | Turn on `mypy --strict` on your package; fix every error. Add a `Protocol` for one dependency and write a fake against it. |
| Pydantic | Model a JSON payload from a real API (a Bedrock Converse response, a GitHub event); validate it; round-trip to JSON; generate `model_json_schema()`. |
| Pythonic | Take Project 1 through ruff with the `UP`, `B`, `SIM`, `C4` rule sets and fix everything it flags. Write a generator pipeline over a large file. |
| Async | Fan out 20 HTTP calls with `asyncio.gather`; then do the same with a `ThreadPoolExecutor`; measure both. Break it deliberately with `time.sleep` in a coroutine and observe. |
| pytest | Write fixtures with `yield`, a parametrised test, one `MagicMock` test and one hand-written fake, one `TestClient` test. |
| FastAPI | Build Project 2. Read `/openapi.json`. Add auth as a dependency. Override it in a test. |
| SQLAlchemy | Add a relationship + eager loading; generate and apply two Alembic migrations; write one Core bulk insert. |
| NumPy/pandas | In a notebook: load a CSV of "LLM calls" (model, tokens, latency), group by model, plot latency; compute cosine similarity of 100 fake embeddings without loops. |
| LLM APIs | Build Project 3 with both providers; add structured output and one tool; stream to `curl -N`. |
| boto3 | Build Project 5; run tests with moto; deploy with SAM to a sandbox account; read CloudWatch logs. |
| Frameworks | Rebuild Project 4's retrieval with LangChain (`PGVector` + retriever) and with LlamaIndex; compare line counts and control. Run a `transformers` pipeline locally once. |
| Projects | Build Project 6 end-to-end with Docker Compose; then decide, for your own company's system, where the model call should live and write down why. |

Total: roughly **five to six focused weeks** part-time. You already know backend engineering; you are learning a dialect and an ecosystem, not a discipline.

---

# 36. Final Knowledge Check

Model answers, kept short. Cover the answer, answer aloud, compare.

**Why doesn't Python require variable declarations like Java?**
Because variables are names bound to objects, not typed slots. The object carries the type; the name is a dictionary entry. There is nothing to declare — the checker (mypy/pyright), not the interpreter, gives you Java-like verification when you add hints.

**What is the difference between a list and a tuple?**
Both are ordered sequences; `list` is mutable and used for homogeneous, variable-length collections; `tuple` is immutable, hashable (so usable as a dict key), and used for fixed heterogeneous groupings and multiple return values. Immutability is shallow.

**What is duck typing?**
Compatibility by behaviour, not declaration: an object is acceptable if it has the attributes/methods used, checked at the moment of use. `Protocol` brings the same idea to the type checker; Java's nominal interfaces require `implements`.

**What is a decorator?**
A callable applied to a function or class at definition time (`f = deco(f)`) that returns a replacement — usually a wrapper closure, sometimes the original after registration, sometimes a modified class. Unlike Java annotations, it *executes*; no container or proxy is needed.

**Why does Python have the GIL?**
CPython manages memory with non-atomic reference counts; a single global lock is the cheap way to keep them (and C extensions) safe across threads. Consequence: one thread runs Python bytecode at a time; threads still help for I/O because the GIL is released while blocking. The free-threaded build (3.13+/3.14) removes it at some single-thread cost.

**When should I use async?**
When the work is I/O-bound *and* the libraries are async-capable *and* concurrency is high (fan-out, streaming, many connections). Not for CPU work, not with blocking libraries, not for a single sequential call — async is throughput under concurrent waiting, not speed.

**What is Pydantic?**
A runtime validation/serialisation library: `BaseModel` classes built from type hints that parse and coerce untrusted data (JSON, env, LLM output) into typed objects, raise structured `ValidationError`s, dump back to dict/JSON, and generate JSON Schema — the Jackson + Bean Validation + schema layer of Python, used by FastAPI, settings, and LLM SDKs.

**What is FastAPI?**
An ASGI web framework that derives routing, validation, serialisation, dependency injection and OpenAPI docs from type-hinted function signatures and Pydantic models; supports sync and async endpoints; runs on uvicorn. Spring Boot's role for JSON/AI APIs, with far less code and no container.

**How does SQLAlchemy compare with Hibernate?**
Both are ORMs with a session/unit-of-work, identity map, change tracking, lazy loading and relationships. SQLAlchemy is layered (Core SQL expressions + ORM), uses explicit `select()` expressions instead of JPQL/derived queries, has explicit transaction boundaries instead of `@Transactional` proxies, and pairs with Alembic for migrations (Flyway's role).

**What is NumPy?**
The n-dimensional typed array library underlying Python's numeric/AI stack: contiguous typed memory, vectorised operations executed in C, broadcasting. Embeddings, model inputs and similarity math are NumPy arrays (or PyTorch tensors, which share the model).

**What is an embedding?**
A dense vector (e.g. 1024 floats) produced by a model such that semantically similar inputs have nearby vectors (high cosine similarity). Stored in a vector store (pgvector, OpenSearch) and compared with a query's embedding to retrieve relevant chunks.

**How does RAG work?**
Offline: load documents → chunk → embed → store vectors with text. Online: embed the question → retrieve the top-k similar chunks → build a prompt with those chunks as context → call the LLM → return the answer (with sources). It grounds the model in your data without training.

**How does boto3 call AWS?**
`boto3.client("service")` builds a client from the service's JSON API model; each method takes keyword arguments (dicts), signs an HTTPS request with credentials from the standard chain (env, profile, role), sends it, and returns the response as a dict. Bedrock is just another service: `converse`, `converse_stream`, `invoke_model`.

**Why is Python so common in AI?**
The ecosystem (NumPy → PyTorch → Transformers → provider SDKs → RAG/agent frameworks) is Python-first; heavy computation runs in C/CUDA so interpreter speed rarely matters; notebooks and dynamic typing make experimentation fast; research and vendors ship Python examples first. It is the control plane of AI, not the engine.

**Extra — quick self-test**

- What does `if __name__ == "__main__":` protect, and why does it matter for tests?
- Why are `models.py` (SQLAlchemy) and `schemas.py` (Pydantic) different things?
- What happens if you call a blocking boto3 method inside `async def`?
- Why is `except Exception: pass` worse in Python than an empty catch in Java? (Hint: what else does it hide — `KeyError` from a typo, for instance.)
- What are the four ways Python expresses an "interface", from lightest to heaviest?
- What is `uv.lock` for and why is `requirements.txt` not a replacement for `pyproject.toml`?

---

# 37. Final Goal — What Do I Now Know About Python?

> *"If I am already a Java backend developer, what do I now know about Python?"*

You know **how Python thinks**: names bound to heap objects instead of typed slots; a runtime that resolves everything by name at the moment of use; a class that is code that ran; a function that is a value; a module that is a script that ran once and became a singleton. You know why that design produces duck typing, decorators, dataclasses, `self`, and no `private`, and why it makes frameworks like FastAPI and Pydantic possible with almost no machinery.

You know **how Python actually works**: source → bytecode → the CPython VM; reference counting plus a cycle collector; the GIL as the price of that memory model; C/Rust/CUDA libraries doing the heavy work underneath a slow but expressive interpreter — and therefore why Python is the language of AI even though the JVM is faster.

You can **read normal Python comfortably** — comprehensions, unpacking, generators, `with`, `@decorators`, `async`/`await`, nested dicts from JSON and boto3, `match` statements, type hints and `Protocol`s — and you can spot Java-accented Python and fix it.

You can **write Python applications**: a packaged CLI, a FastAPI service with Pydantic models, a service/repository split, PostgreSQL through SQLAlchemy and Alembic, configuration from the environment, logging, and pytest suites with fixtures, fakes and Testcontainers. You know how to structure a project, why `src/` exists, why `if __name__ == "__main__"` exists, and how uv, `pyproject.toml` and `uv.lock` replace Maven.

You **understand Python concurrency** well enough to choose correctly: threads for blocking I/O, processes for CPU, asyncio for high-concurrency async I/O — and you know what the GIL does and does not prevent.

You can **use Python for AI**: NumPy arrays and pandas frames at reading level; scikit-learn's estimator API; calling Claude, GPT and Bedrock models with the provider SDKs and boto3 — system prompts, multi-turn history, streaming, structured output, tool use, error handling and retries; embeddings and cosine similarity; RAG end-to-end on pgvector; the shape of LangChain, LlamaIndex, Transformers and PyTorch code so that none of it looks alien.

You can **read Python AWS/GenAI code**: `boto3.client("bedrock-runtime").converse(...)`, `invoke_model` for embeddings, Knowledge Bases, S3/DynamoDB/Lambda idioms, `ClientError` handling — and map each of them to the AWS SDK for Java you already use. That is the Python you will meet in the AIP-C01 material and in real AWS projects.

And you know **how Python and Java coexist**: a Java/Spring backend owning the domain and a Python FastAPI service owning the AI logic, connected by an OpenAPI contract, with clear criteria for when the model call belongs in Java instead.

Most importantly:

> **Python is no longer a different world.** It is a different dialect of ideas you already hold — objects, modules, services, tests, HTTP, SQL, concurrency — with a few genuinely different design choices (dynamic typing, duck typing, the GIL, functions as values) that you now understand well enough to exploit rather than fight. When you open a Python file in an AWS, AI, ML or GenAI project, you will read it as a backend developer reads code: knowing what it does, why it is shaped that way, and how you would write it yourself.

Now go build the projects.

---

*End of guide.*
