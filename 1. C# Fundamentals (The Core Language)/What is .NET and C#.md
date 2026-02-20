---
created: 2026-02-19
topic_type: #fundamental
status: #seed
source_link: https://metanit.com/sharp/
---
# Introduction to C# and .NET

## 1. Executive Summary
C# is the programming language you write. .NET is the platform that makes your C# code *run* reliably and efficiently on real machines.

If you keep one mental model:
- **C# = syntax + rules + compiler**
- **.NET = runtime + libraries + tooling + app models**
- **Your C# code → compiled to IL → executed by .NET (JIT/AOT) → native machine code**

This lesson explains what each piece is, how it works under the hood, how .NET evolved, what versions mean in practice, and how to install everything and run your first program professionally.

---

## 2. Technical Deep Dive

### 2.1 What is C#
**C#** is a modern, strongly-typed language created for building:
- Web APIs and backend services (ASP.NET Core)
- Desktop apps (WPF/WinForms)
- Mobile and cross-platform apps (.NET MAUI)
- Games (Unity uses C#)
- Tools, automation, and cloud workloads

#### Core Mechanics (how C# actually becomes a running program)
**C# source code** (`.cs`) does not run directly. The flow is:

1. **You write C#**
2. **Roslyn compiler** compiles it into an **assembly**
	- Output: `.dll` or `.exe`
	- Contains:
		- **IL (Intermediate Language)**: CPU-independent instructions
		- **Metadata**: descriptions of types, methods, attributes, references
3. **.NET runtime loads the assembly**
4. **JIT (Just-In-Time) compilation** typically compiles IL to **native code** for your current CPU
5. The program executes as machine code

This is why C# can be portable (IL is portable) while still being fast (native code is produced).

#### The “managed code” idea
C# is usually **managed**:
- Memory allocation is tracked by the runtime
- Objects on the heap are reclaimed by the **GC (Garbage Collector)**
- You are protected from many memory-corruption bugs common in unmanaged languages

You can still do unsafe/interoperability code when needed, but the default is safe and productive.

#### Type system: value types vs reference types (why seniors care)
C# has two big categories:

- **Value types** (`struct`, e.g., `int`, `double`, `DateTime`)
	- Usually stored “inline” (often on the stack or inside another object)
	- Copy-by-value semantics (assignment copies the value)
	- Can avoid heap allocations when designed well

- **Reference types** (`class`, e.g., `string`, `List<T>`)
	- Instances typically live on the managed heap
	- Variables store references (pointers) to objects
	- GC must later reclaim them

This directly impacts:
- allocations
- GC pressure
- performance in hot paths
- correctness (mutability and shared state)

#### Senior Insights & Pitfalls (real-world mistakes and how to think)
- **Pitfall: “C# performance = language features”**
	- Usually performance is driven by:
		- allocation patterns
		- boxing/unboxing
		- async overhead
		- string + LINQ usage in loops
		- collections sizing and copying

- **Pitfall: overusing LINQ in critical loops**
	- LINQ is great for clarity, but can allocate enumerators/closures and add overhead.
	- Senior approach: use LINQ for readability in non-hot paths, write explicit loops in hot paths.

- **Pitfall: ignoring nullability**
	- Modern C# uses nullable reference types to prevent `NullReferenceException`.
	- Senior approach: enable nullability and treat warnings as design feedback.

---

### 2.2 What is .NET
**.NET** is the full platform that builds and runs C# applications.

At a high level, .NET includes:
- **Runtime** (CoreCLR): executes code, JIT, GC, threading, exceptions
- **Base Class Library (BCL)**: `System.*` namespaces (collections, IO, HTTP, JSON, etc.)
- **SDK + CLI** (`dotnet`): create/build/test/publish tools
- **App frameworks**: ASP.NET Core, worker services, desktop, MAUI

#### Core Mechanics (runtime responsibilities)
When your app runs, the .NET runtime handles:
- **Loading assemblies**
- **Type checks and metadata inspection**
- **JIT compilation**
- **Garbage collection**
- **Threading and ThreadPool**
- **Async/await scheduling**
- **Interop** with native libraries (P/Invoke)

#### JIT vs AOT (what it means practically)
- **JIT** compiles methods at runtime.
	- Pros: runtime can optimize for the actual machine
	- Cons: startup cost (usually small but real)
- **AOT** compiles ahead of time.
	- Pros: faster startup, more predictable deployment in some cases
	- Cons: constraints around reflection/dynamic loading, build complexity

Senior view: choose based on production needs (startup vs flexibility vs deployment constraints).

#### Senior Insights & Pitfalls
- **“.NET” is not just a library. It is the execution environment.**
	- Your debugging, profiling, memory behavior, and concurrency model all depend on runtime behavior.
- **Deployment strategy is architecture**
	- Framework-dependent: smaller, depends on installed runtime
	- Self-contained: ships runtime with app, larger, more predictable

---

### 2.3 History of .NET (why it evolved)
You do not need every date. You need the *reasons* behind the changes.

#### Core Mechanics (high-signal evolution)
1. **.NET Framework era (Windows-first)**
	- Tight integration with Windows.
	- Powerful desktop and enterprise server tech.
	- Limitation: not designed as cross-platform.

2. **.NET Core era (cross-platform reset)**
	- Goal: run on Windows, Linux, macOS.
	- More performance focus, modular design.
	- Unified CLI workflow: `dotnet new/build/run/test/publish`.

3. **Unified .NET (5+)**
	- Microsoft simplified branding and roadmap:
		- No split between “Framework vs Core” for new development.
		- “.NET” is the main line going forward.

#### Senior Insights
- Many enterprises still maintain legacy .NET Framework apps.
- Migration is mostly blocked by:
	- third-party dependencies
	- hosting model changes
	- Windows-only technologies

---

### 2.4 Versions of .NET (how to interpret version numbers)
Think in terms of:
- **Target Framework**: what your app is built for (e.g., `net8.0`)
- **Runtime installed**: what your machine has available to run the app
- **Support model**: LTS vs non-LTS

#### What you should know
- **.NET Framework**: legacy line, Windows-only, maintenance mode.
- **.NET Core (1.0–3.1)**: the bridge to modern .NET.
- **.NET 5+**: the modern unified platform.

#### LTS vs STS mindset (production-grade rule)
- **LTS**: best default for real products and teams.
- **STS**: good when you want features faster and can upgrade often.

Senior rule: *use LTS unless you have a strong reason not to.*

---

### 2.5 Major Components of .NET (the “map”)
A clean mental map:

1. **C# Compiler (Roslyn)**
- Turns `.cs` into IL + metadata.

2. **.NET Runtime (CoreCLR)**
- JIT/AOT, GC, threading, exception handling, assembly loading.

3. **BCL (Base Class Library)**
- Core types and APIs: `string`, `List<T>`, `Task`, `HttpClient`, IO, JSON, etc.

4. **SDK + CLI (`dotnet`)**
- Project creation, building, running, testing, publishing.

5. **Project System (MSBuild)**
- Resolves references, runs build steps, produces artifacts.

6. **NuGet**
- Package manager for third-party and internal libraries.

7. **Application Models**
- Console apps
- ASP.NET Core web apps and APIs
- Worker services
- Desktop apps
- Mobile/cross-platform apps

#### Senior Insights & Pitfalls
- Most “mysterious bugs” come from misunderstanding one layer:
	- runtime version mismatch
	- package version conflicts
	- trimming/reflection constraints
	- async/threadpool behavior under load

---

## 3. Implementation Example

### 3.1 Installing .NET (SDK) correctly
You want the **.NET SDK**, not only the runtime.

#### Verify installation
After install, open a terminal:

```powershell
dotnet --info

dotnet --version
```

You should see:
- OS info
- installed SDKs
- installed runtimes

If `dotnet` is not recognized, it is a PATH issue or install issue.
#### Senior Notes (common setup mistakes)
- Installing only the runtime means you cannot build projects.
- Multiple SDK versions can coexist. That is normal.
- Teams often pin a version using `global.json` so everyone builds with the same SDK.

Example `global.json`:
```json
{

"sdk": {

"version": "8.0.200",

"rollForward": "latestFeature"

}

}
```

---

### 3.2 Hello World in 5 minutes
#### Create a new console app

```powershell
dotnet new console -n HelloDotNet

cd HelloDotNet

dotnet run
```
#### What is created (important files)
- `HelloDotNet.csproj` (project file)
- `Program.cs` (entry code)
#### Minimal C# program (top-level statements)

```csharp
Console.WriteLine("Hello, .NET!");
```
#### What actually happens when you run it
- Build step compiles C# → IL assembly
- Runtime loads assembly
- JIT compiles relevant methods
- Your app runs as native code

---

### 3.3 Installing an IDE: Visual Studio (Windows)
For learning and professional work, **Visual Studio** is a strong default on Windows.

#### What to install (workloads concept)
Visual Studio installs features via **Workloads**. Choose based on your goal:
- **.NET desktop development** (for WinForms/WPF)
- **ASP.NET and web development** (for web APIs)
- Optional: **Azure development** (if you deploy to Azure)

#### After installation (sanity checks)
- Create a new “Console App”
- Run it
- Confirm it uses the expected .NET version

#### Senior Notes
- Visual Studio includes MSBuild tooling and excellent debugging.
- For backend work, also learn the CLI (`dotnet`) so your workflow matches CI/CD.

---

## 4. Connections (Mental Map)
- **Parent Topic:** [[C# Fundamentals]]
- **Related Patterns:** [[CLR and IL]], [[JIT Compilation]], [[Garbage Collection]], [[NuGet Versioning]], [[.NET Deployment: Framework-dependent vs Self-contained]]

---

## 5. Practical Assessment

### Practice Task
1. Install .NET SDK.
2. Run `dotnet --info` and paste the output into your notes.
3. Create a console app with `dotnet new console`.
4. Modify it to print:
	- .NET version
	- OS description
	- process architecture

Hint (use these APIs):
- `Environment.Version`
- `System.Runtime.InteropServices.RuntimeInformation`

### Source Code Review
When reviewing .NET internals on source.dot.net for the first time, search and skim:
- How the runtime represents types and loads metadata (conceptual)
- How core types are written to avoid allocations
- Why common APIs have allocation-friendly patterns (`TryParse`, spans, pooling)
