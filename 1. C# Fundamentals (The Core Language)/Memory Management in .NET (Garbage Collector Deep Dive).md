---
created: 2026-02-19
topic_type: #fundamental
status: #seed
source_link: https://metanit.com/sharp/
---
# Lesson 4: Memory Management in .NET (Garbage Collector Deep Dive)

## 1. Executive Summary
In .NET, most memory management is **automatic**. You allocate objects (usually on the managed heap), and the **Garbage Collector (GC)** later reclaims memory that your program can no longer reach.

This automation is a massive productivity and safety win, but it is not magic. A senior engineer understands:

- **Allocations are cheap, but not free.**
- **GC work is the “bill” you pay later for allocations.**
- The main goal is not “avoid the GC”.
	The goal is **reduce unnecessary allocations** and **avoid pathological object lifetimes** that cause pauses or memory growth.
- **GC does not manage everything.**
	Things like file handles, sockets, and unmanaged memory still need deterministic cleanup.

If you understand:
- stack vs heap,
- reachability,
- generations (Gen0/1/2),
- LOH,
- finalizers vs `IDisposable`,
- and common allocation traps,

you can write .NET code that is fast, stable under load, and predictable in production.

---

## 2. Technical Deep Dive

### 2.1 Managed vs unmanaged resources (what GC does and does not do)
.NET apps deal with two broad categories of resources:

#### Managed memory (GC-managed)
- Objects allocated on the managed heap (most `class` instances, arrays, strings, etc.)
- Lifetime is handled by GC based on **reachability**
- Example: `new List<int>()`, `new string(...)`, `new byte[1024]`

#### Unmanaged resources (NOT automatically reclaimed in a timely way)
These live outside the managed heap:
- OS file handles
- sockets
- database connections
- window handles
- unmanaged memory allocated via native calls

Important truth:
- The GC can reclaim managed objects.
- The GC cannot reliably “close your file” at the moment you need it closed.

Senior rule:
- **Use deterministic cleanup (`IDisposable`) for scarce or OS-level resources.**

---

### 2.2 Allocation: what happens when you write `new`
When you write:
```
var list = new List<int>();
```

You are requesting:
- a managed heap allocation for the `List<int>` object
- possibly more allocations later (for the internal array) as the list grows

Allocation in .NET is generally fast because:
- it often means “bump a pointer” in a region of memory

But the cost shows up later:
- more allocations → more GC cycles
- more long-lived objects → more Gen2 work

---

### 2.3 Reachability: how GC decides what to collect
GC uses the concept of **reachability**:
- An object is “alive” if it can be reached from a **GC root**
- If it is not reachable, it is eligible to be collected

Common GC roots:
- Local variables and parameters on thread stacks (references)
- Static fields
- CPU registers holding references
- Handles tracked by the runtime

Example:
```
var buffer = new byte[10_000_000];
```

Senior insight:
- Repeatedly allocating large buffers can create memory pressure.
- Production systems often use pooling strategies for large buffers.

---

### 2.6 Stop-the-world pauses (what causes latency spikes)
GC often requires pausing managed threads briefly to safely analyze references and reclaim memory. This can show up as:
- small pauses (often not noticeable)
- occasional longer pauses under heavy allocation + high heap size

What makes pauses worse:
- large heaps (especially lots of Gen2 objects)
- high allocation rates under load
- fragmentation and LOH pressure
- too many objects surviving collections (promotion)

Senior rule:
- If you care about latency, you care about **allocation rate** and **object lifetime**.

---

### 2.7 Finalizers vs `IDisposable` (deterministic cleanup)
#### `IDisposable` (what you should use)
Use `IDisposable` to free unmanaged resources immediately:
```
public sealed class NativeBuffer : IDisposable

{

private IntPtr _ptr;

private bool _disposed;

public NativeBuffer(int size)

{

_ptr = System.Runtime.InteropServices.Marshal.AllocHGlobal(size);

}

~NativeBuffer()

{

Dispose(false);

}

public void Dispose()

{

Dispose(true);

GC.SuppressFinalize(this);

}

private void Dispose(bool disposing)

{

if (_disposed) return;

if (_ptr != [IntPtr.Zero](http://IntPtr.Zero))

{

System.Runtime.InteropServices.Marshal.FreeHGlobal(_ptr);

_ptr = [IntPtr.Zero](http://IntPtr.Zero);

}

_disposed = true;

}

}
```

Senior insight:
- Most application types do not need finalizers.
- Prefer `SafeHandle`-based APIs and `using` for resource cleanup.

---

### 2.8 Common allocation traps (and the senior fixes)
#### Trap 1: accidental allocations in loops
```
for (int i = 0; i < 1_000_000; i++)

{

var s = i.ToString(); // allocates a new string each iteration

}
```

Senior fix:
- Avoid repeated string creation in hot paths.
- Use structured logging, batching, or buffers where appropriate.

#### Trap 2: hidden allocations from closures
```
var numbers = Enumerable.Range(1, 1000);

var result = numbers.Where(n => n % 2 == 0).ToList();
```

Senior fix:
- LINQ is fine in non-hot paths.
- In hot paths, prefer loops to reduce allocations and overhead.

#### Trap 3: keeping references longer than needed
```
static List<byte[]> Cache = new();

void AddToCache()

{

Cache.Add(new byte[1024 * 1024]); // long-lived memory by design

}
```

Senior fix:
- Design caches with bounds, eviction, and observability.
- Prefer `IMemoryCache` or a bounded structure.

---

## 3. Implementation Example

### Standard Usage: demonstrate allocations and object lifetime
```
using System;

using System.Collections.Generic;

static void PrintMemory(string title)

{

var bytes = GC.GetTotalMemory(forceFullCollection: false);

Console.WriteLine($"{title}: {bytes:N0} bytes");

}

PrintMemory("Start");

for (int i = 0; i < 200_000; i++)

{

var list = new List<int>(100);

list.Add(i);

}

PrintMemory("After allocations");

// Force a full collection for demonstration only.

// Do NOT do this in real production code.

GC.Collect();

GC.WaitForPendingFinalizers();

GC.Collect();

PrintMemory("After forced GC");
```

Senior note:
- `GC.Collect()` is almost always a bad idea in real apps.
- It is used here only to make the example observable.

### Advanced/Pragmatic Usage: pooling to reduce GC pressure (conceptual)
For large buffers, pooling can reduce allocations:
```
using System;

using System.Buffers;

var pool = ArrayPool<byte>.Shared;

byte[] buffer = [pool.Rent](http://pool.Rent)(1024 * 1024);

try

{

// Use the buffer

buffer[0] = 123;

}

finally

{

pool.Return(buffer, clearArray: true);

}
```

Senior insight:
- Pooling is powerful, but it adds complexity.
- Use it when profiling shows allocation/GC is a real bottleneck.

---

## 4. Connections (Mental Map)
- **Parent Topic:** [[C# Fundamentals]]
- **Related Patterns:** [[Stack vs Heap]], [[Value vs Reference Types]], [[IDisposable and using]], [[LOH]], [[Performance: Allocations and GC]], [[Object Lifetime and Caching]]

---

## 5. Practical Assessment

### Practice Task
Write a console app that:
1. Allocates many short-lived objects in a loop (like lists or strings).
2. Prints `GC.GetTotalMemory(false)` before and after.
3. Forces a GC (for learning only) and prints memory again.
4. Repeat with a pooling strategy (`ArrayPool<byte>`).

In your notes, answer:
- Which version allocates more?
- Which version keeps memory stable under repeated runs?
- What is the complexity cost of pooling?

### Source Code Review
On source.dot.net, search and skim:
- `GC` APIs to see what is public vs internal.
- `ArrayPool<T>` to understand why pooling exists.
- `List<T>` growth logic to see how capacity increases cause new array allocations.

Goal:
- Identify where allocations happen and how the BCL tries to balance safety, speed, and simplicity.