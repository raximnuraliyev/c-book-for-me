---
created: 2026-02-19
topic_type: #fundamental
status: #seed
source_link: https://metanit.com/sharp/
---
# Lesson 3: Types, Variables, and Memory (Stack vs Heap)

## 1. Executive Summary
This lesson answers two questions that control most “why did this happen?” moments in C#:

1. **What is a variable, really?**
	A variable is not “the object”. A variable is a *typed storage location* that holds either:
	- the value itself, or
	- a reference to a value stored elsewhere.

2. **Where does data live at runtime?**
	- The **stack** stores *call frames* and short-lived local data.
	- The **heap** stores *objects with dynamic lifetime* (most reference-type instances).
	- The **GC (Garbage Collector)** reclaims heap objects that are no longer reachable.

If you understand the difference between **value types** and **reference types**, and how that maps to **stack/heap**, you will understand:
- why copying sometimes duplicates data and sometimes does not,
- why mutation can “leak” through references,
- why allocations and GC matter for performance,
- why boxing/unboxing exists.

---

## 2. Technical Deep Dive

### 2.1 Types and variables (what C# guarantees)
C# is **statically typed**:
- Every variable has a **type** at compile time.
- The compiler uses that type to enforce correctness and generate efficient code.

A **variable** is a name bound to a storage location.
- The type defines:
	- how many bytes are needed (conceptually),
	- how to interpret those bytes,
	- what operations are allowed.

Example:
```csharp
int age = 21;
```
- `age` is a variable
- `int` is the type
- `21` is the value stored in `age`

Now compare:
```csharp
var name = "Ajax";
```
- `var` does **not** mean “dynamic”.
- `var` means “infer the static type from the right side”.
- Here the static type is `string`.

Senior insight:
- Strong typing is not just “rules”.
- It enables better refactoring, better tooling, fewer runtime surprises, and often better performance.

---

### 2.2 Value types vs reference types (the most important split)
In C#, types fall into two categories:

#### Value types (`struct`, primitives like `int`, `double`, `bool`)
- The variable typically **contains the value directly**.
- Assignment usually **copies the value**.
- Value types are great for small, immutable data and performance-sensitive code.

Examples:
- `int`
- `double`
- `bool`
- `DateTime` (struct)
- custom `struct`

#### Reference types (`class`, arrays, `string`, `List<T>`)
- The variable contains a **reference** (think: pointer-like) to an object.
- Assignment usually **copies the reference**, not the object.
- Multiple variables can point to the same object.

Examples:
- `string` (special case, but still reference type)
- `object`
- arrays like `int[]`
- `List<int>`
- custom `class`

---

### 2.3 Stack vs heap (a practical mental model)
This model is not perfect in every low-level detail, but it is correct enough for professional reasoning.

#### Stack (call stack)
The stack is:
- a structured memory region that stores **stack frames**
- each frame corresponds to a function call
- it usually contains:
	- return address
	- parameters (conceptually)
	- local variables (especially value-type locals)

Key properties:
- Very fast allocation (just moving a pointer).
- Automatic cleanup when a method returns.
- Lifetime is tied to scope / call chain.

#### Heap (managed heap)
The heap is:
- where **objects** are allocated (most reference-type instances).
- managed by the runtime.
- cleaned up by the **GC**, not by “when the method ends”.

Key properties:
- Allocation is still fast, but not “free”.
- Cleanup is non-deterministic (GC runs when needed).
- Many heap allocations increase GC pressure and can affect latency.

---

### 2.4 Concrete examples (copying, mutation, and why it surprises people)

#### Example A: value type assignment copies data
```csharp
int a = 10;

int b = a; // copy the value

b = 20;

Console.WriteLine(a); // 10

Console.WriteLine(b); // 20
```
Reason:
- `int` is a value type.
- `b = a` duplicates the value (10).

#### Example B: reference type assignment copies the reference
```csharp
var list1 = new List<int> { 1, 2, 3 };

var list2 = list1; // copy the reference (both point to the same list)

list2.Add(4);

Console.WriteLine(list1.Count); // 4

Console.WriteLine(list2.Count); // 4
```
Reason:
- `List<int>` is a reference type.
- `list2 = list1` means both variables refer to the same heap object.
- Mutating through one variable is visible through the other.

Senior insight:
- Many “bugs” are actually *unintended aliasing* (two references to the same object).

---

### 2.5 `string` special case (reference type with value-like behavior)
`string` is a reference type, but it is:
- **immutable** (cannot be changed after creation)
- heavily optimized by the runtime (interning may occur)

Example:
```csharp
string s1 = "hi";

string s2 = s1;

s2 = s2.ToUpper();

Console.WriteLine(s1); // hi

Console.WriteLine(s2); // HI
```

This looks like value behavior, but it is actually:
- `ToUpper()` creates a new string object.
- `s2` is updated to reference the new string.
- `s1` still references the original string.

Senior insight:
- Immutability is one of the strongest tools for correctness in large systems.

---

### 2.6 Boxing and unboxing (why value types can become heap allocations)
**Boxing** happens when a value type is treated as an `object` (or an interface it implements).
The runtime allocates a new object on the heap to hold that value.
```csharp
int x = 123;

object boxed = x; // boxing (heap allocation)

int y = (int)boxed; // unboxing (copy back to value type)
```

Why you should care:
- Boxing introduces allocations.
- Allocations can increase GC work.
- In high-frequency code (logging, collections, serialization), boxing can matter.

Senior pitfall:
- Using non-generic collections (like `ArrayList`) or APIs taking `object` can cause hidden boxing.
- Prefer generics: `List<int>` instead of `List<object>` when appropriate.

---

### 2.7 What determines lifetime? (reachability)
A heap object lives as long as it is **reachable** from “GC roots” (like:
- stack references in running threads,
- static fields,
- handles maintained by the runtime).

Example:
```csharp
List<int> Make()

{

var list = new List<int> { 1, 2, 3 };

return list; // the reference escapes the method, so the object can outlive the stack frame

}
```
Even though the method returns, the list is still reachable via the returned reference.

Senior insight:
- Heap lifetime is not “scope-based”.
- It is “reachability-based”.

---

## 3. Implementation Example

### Standard Usage: show value vs reference behavior
```csharp
using System;

using System.Collections.Generic;

Console.WriteLine("Value type copy:");

int a = 10;

int b = a;

b = 20;

Console.WriteLine($"a = {a}, b = {b}");

Console.WriteLine();

Console.WriteLine("Reference type copy:");

var list1 = new List<int> { 1, 2, 3 };

var list2 = list1;

list2.Add(4);

Console.WriteLine($"list1.Count = {list1.Count}, list2.Count = {list2.Count}");
```

### Advanced/Pragmatic Usage: avoid allocations with structs (carefully)
A common performance technique is using structs to represent small data without heap allocations, but only when the struct is:
- small
- immutable (or treated immutably)
- not frequently copied in large graphs
```csharp
using System;

public readonly struct Point2D

{

public int X { get; }

public int Y { get; }

public Point2D(int x, int y)

{

X = x;

Y = y;

}

public override string ToString() => $"({X}, {Y})";

}

var p1 = new Point2D(10, 20);

var p2 = p1; // copy the value

Console.WriteLine(p1);

Console.WriteLine(p2);
```

Senior warning:
- Large structs copied repeatedly can be slower than classes.
- “Struct for performance” is only correct when you understand copying costs and usage patterns.

---

## 4. Connections (Mental Map)
- **Parent Topic:** [[C# Fundamentals]]
- **Related Patterns:** [[Immutability]], [[Garbage Collection]], [[Boxing and Unboxing]], [[Generics]], [[Performance: Allocations and GC]]

---

## 5. Practical Assessment

### Practice Task
Write a small console app that:
1. Creates a `struct` and a `class` version of the same concept (for example, `Money` or `Point`).
2. Assigns each to a second variable and mutates what is mutable.
3. Prints outputs to prove:
	- value copy behavior
	- reference aliasing behavior

Then answer in your notes:
- Which version is safer by default?
- Which version allocates on the heap?
- In what scenario would you choose each?

### Source Code Review
On source.dot.net, search and skim:
- `System.String` to confirm immutability patterns.
- `System.Collections.Generic.List<T>` to see internal array growth and why resizing allocates.
- `System.Object` to understand why boxing targets `object`.

Goal:
- Identify where allocations happen and why the library is designed that way.
