# TestProject1/MSTestSettings.cs

## Purpose

This file is MSTest's **module-level settings file**. Its single line,

```csharp
[assembly: Parallelize(Scope = ExecutionScope.MethodLevel)]
```

tells the MSTest runner (MSTest 4.2.1, per `TestProject1.csproj`) to run the test **methods** in this assembly concurrently, using a separate execution context per method.

That is the only configuration this assembly carries: no explicit worker count, no `MaxConcurrency` — the runner defaults apply.

## Role in the Codebase

This is not a source file with logic; it is build-time test-runner configuration for the `TestProject1` unit test project. MSTest looks for a file named `MSTestSettings.cs` in the test assembly and applies any `[assembly: ...]` attributes it finds there, so the file name is what makes it work — it cannot be arbitrarily renamed or merged into `UserViewModelTest.cs` (an assembly attribute could technically live anywhere in the assembly, but the conventional, discoverable location is this file). If it were removed, all tests in the project would fall back to MSTest's default: sequential execution, no parallelization.

## Relationships

- **Consumed by the MSTest framework**, not by application code. No class in the project imports or references anything from this file; it compiles into assembly-level attributes that the test runner reads at discovery/execution time.
- **Scope of effect: the `UserViewModelTest` class** in `UserViewModelTest.cs` — the only test class in the project. That class is annotated `[DoNotParallelize]`, which opts its methods out of the parallelization enabled here. So in practice, as the project stands today, the setting has **no observable effect**: every test in the assembly is excluded from parallel execution.

## Behavior & Details

- `ExecutionScope.MethodLevel` means each `[TestMethod]` gets its own isolated context and may run in parallel with methods from other classes. A coarser scope, `ClassLevel`, would run methods within a class serially and classes in parallel; the finer method-level scope is the most aggressive option.
- `[DoNotParallelize]` at class level (as `UserViewModelTest` uses) is the supported escape hatch, and is appropriate here: the test class relies on MSTest's per-test `TestInitialize`/`TestCleanup` cycle and shares static, mutable infrastructure — `WeakReferenceMessenger.Default` (a `CommunityToolkit.Mvvm` static singleton) — which concurrent tests could interleave on. So opting out is deliberate and correct, but it does make the assembly-level `Parallelize` setting currently inert.
- A caveat if you ever add more test classes: the `[Parallelize]` setting applies assembly-wide from the moment any class does **not** carry `[DoNotParallelize]`. New tests that touch shared statics (messenger registries, caches, file system) will silently start running concurrently, and any shared-state assumption will surface as flaky tests rather than a compile error.

### Rough edges

- As currently written, the file's only directive is fully neutralized by `[DoNotParallelize]` on the sole test class — the two files contradict each other, and it is unclear which is intended to be the default policy (parallel where possible, or sequential by default).