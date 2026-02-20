---
created: 2026-02-19
topic_type: #fundamental
status: #seed
source_link: https://metanit.com/sharp/
---
# Lesson 2: The .NET CLI and Project Anatomy (From `dotnet new` to a Build Artifact)

## 1. Executive Summary
In professional .NET development, you should understand two things deeply:

1. **The .NET CLI (`dotnet`) is the “front door”** to the platform.
	It can create projects, restore dependencies, build, run, test, and publish your app.
2. **A .NET project is defined by a `.csproj` file**, not by the IDE.
	Visual Studio is a powerful interface, but the build is ultimately driven by MSBuild reading your `.csproj`.

If you understand:
- how `dotnet` maps to MSBuild,
- how a `.csproj` describes inputs and dependencies,
- and what happens during restore/build/run,

then you can debug build issues, set up CI/CD cleanly, and work confidently across machines.

---

## 2. Technical Deep Dive

### Core Mechanics

### 2.1 What is the .NET CLI (`dotnet`)?
The **.NET CLI** is the command-line tool that ships with the **.NET SDK**. It is not just a “runner”. It is a unified interface for:

- Creating projects and solutions
- Restoring NuGet packages
- Building with MSBuild
- Running apps
- Testing
- Packing and publishing

A key professional mindset:
- **The IDE is optional**
- **The CLI is universal**
- **CI/CD pipelines almost always use CLI commands**

#### The most important commands (conceptual map)
- `dotnet new`  
	Create a project from a template.
- `dotnet restore`  
	Resolve and download NuGet dependencies.
- `dotnet build`  
	Compile code into an assembly (`.dll`/`.exe`).
- `dotnet run`  
	Build (if needed) and execute.
- `dotnet test`  
	Build and run test projects.
- `dotnet publish`  
	Produce deployable output (often with trimming, single-file options, runtime selection).

---

### 2.2 Introducing project/solution structure
In .NET you will usually see one of these structures:

#### A) Single project repo (small apps, learning)

* MyApp/

* MyApp.csproj

* Program.cs
#### B) Solution with multiple projects (professional default)

* MyCompany.Product.sln
* src/
	* Product.Api/
		* Product.Api.csproj
	* Product.Core/
		* Product.Core.csproj
* tests/
	* Product.Tests/
		* Product.Tests.csproj

**What a solution (`.sln`) is:**
- A *container* that groups projects.
- It is mostly for organization and IDE experience.
- The real build definition lives in each **`.csproj`**.

**Professional rule:**
- Solutions help humans and tools navigate.
- Projects define compilation, dependencies, and outputs.

---

### 2.3 Project anatomy: what is in a `.csproj`?
A **`.csproj`** is an MSBuild project file (XML) that describes:
- Target framework (what .NET your app targets)
- Output type (Exe or Library)
- References to other projects
- NuGet packages
- Build settings and compiler options

#### Example: modern SDK-style `.csproj`
```csharp
<Project Sdk="[Microsoft.NET](http://Microsoft.NET).Sdk">

	<PropertyGroup>
		<OutputType>Exe</OutputType>
		<TargetFramework>net8.0</TargetFramework>
		<ImplicitUsings>enable</ImplicitUsings>
		<Nullable>enable</Nullable>
	</PropertyGroup>

</Project>
```


---

### 2.4 SDK-style projects (why they matter)
“SDK-style” means your project file uses:
- `<Project Sdk="...">`

This brought major improvements:
- Far less boilerplate
- Sensible defaults (like automatically including `*.cs` files)
- Easier multi-targeting (build against multiple target frameworks)
- Better compatibility with CLI workflows and modern tooling

#### Example: multi-targeting (library that supports multiple runtimes)
```csharp
<Project Sdk="[Microsoft.NET](http://Microsoft.NET).Sdk">

	<PropertyGroup>
		<TargetFrameworks>net8.0;netstandard2.0</TargetFrameworks>
		<Nullable>enable</Nullable>
	</PropertyGroup>
	
</Project>
```

Senior perspective:
- Multi-targeting is a product decision, not a gimmick.
- It increases compatibility but increases maintenance and testing cost.

---

### 2.5 `Program.cs`: the entry point (and why it changed)
`Program.cs` is commonly where execution starts.

In modern C# templates, you often see **top-level statements**:
```csharp
Console.WriteLine("Hello, world!");
```

What is happening under the hood:
- The compiler generates a `Program` class and a `Main` method for you.
- This reduces ceremony but does not remove the concept of an entry point.

#### When you might prefer an explicit `Main`
In bigger apps, especially when you want clearer control over:
- exit codes
- argument parsing
- structured startup and exception handling

Example:
```csharp
using System;

	public static class Program
	{
		public static int Main(string[] args)
		{
		try
		{
			Console.WriteLine("Starting...");
			return 0;
		}
		catch (Exception ex)
		{
			Console.Error.WriteLine(ex);
			return 1;
		}
	}
}
```

Senior insight:
- Top-level statements are fine.
- The real professionalism comes from disciplined structure, logging, and predictable startup behavior.

---

### 2.6 Restore, Build, Run: what actually happens

#### `dotnet restore`

What it does:
- Reads your `.csproj`
- Resolves NuGet dependencies (including transitive dependencies)
- Downloads packages into a global cache
- Produces a lock/asset file used by the build (commonly under `obj/`)

Pitfalls:
- “It builds on my machine” often means different package versions or different feeds.
- Private NuGet feeds require proper authentication in CI.

#### `dotnet build`

What it does:
- Compiles C# → IL
- Writes outputs to:
	- `bin/<Configuration>/<TargetFramework>/`
- Generates intermediate build files in:
	- `obj/`

Outputs you should recognize:
- `MyApp.dll` (your assembly)
- `MyApp.deps.json` (dependency graph for runtime)
- `MyApp.runtimeconfig.json` (runtime settings, target runtime framework)

Senior insight:
- Understanding `bin/` vs `obj/` helps you debug “stale build” and incremental build issues.

#### `dotnet run`

What it does:
- Builds if necessary (Debug by default)
- Executes your app using the selected target framework
- Uses the runtime on your machine unless you published self-contained

A common professional habit:
- Use `dotnet run -- --your-app-args` to separate dotnet args from app args.

Example:
```powershell
dotnet run -- --port 5001
```

---

## 3. Implementation Example

### Standard Usage: create + inspect a project

```powershell
dotnet new console -n Lesson2Demo
cd Lesson2Demo
```

View the project file:
```powershell
type Lesson2Demo.csproj
```

Build and run:
```powershell
dotnet build
dotnet run
```
### Advanced/Pragmatic Usage: reproducible builds and clean outputs
Clean and rebuild from scratch:
```powershell
dotnet clean
dotnet build -c Release
```
Publish (deployment artifact):
```powershell
dotnet publish -c Release
```
Why seniors publish locally sometimes:
- To inspect actual deploy output
- To verify trimming/single-file/runtime settings
- To catch missing config files early

---

## 4. Connections (Mental Map)
- **Parent Topic:** [[C# Fundamentals]]
- **Related Patterns:** [[NuGet Dependency Management]], [[MSBuild Basics]], [[Target Frameworks]], [[Debug vs Release Builds]], [[Deployment with dotnet publish]]

---

## 5. Practical Assessment

### Practice Task
1. Create a console app with `dotnet new console`.
2. Edit `Program.cs` to print:
	- `Environment.Version`
	- the command-line args length
3. Run:
	- `dotnet run`
	- `dotnet build -c Release`
	- `dotnet publish -c Release`
4. Compare outputs in:
	- `bin/Debug/...`
	- `bin/Release/...`
	- `bin/Release/.../publish/`

Write 5 bullet points explaining what changed and why.

### Source Code Review
On source.dot.net, when you are ready, look for:
- How the runtime starts a managed app (high-level hosting concepts)
- How `Console` and basic IO are implemented
- How `Environment` provides runtime information

(Do not try to memorize internals. The goal is to learn how the platform is layered.)
