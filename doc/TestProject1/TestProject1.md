### Purpose

**TestProject1.csproj** — `TestProject1/TestProject1.csproj`

This is the MSBuild project file for the solution's unit test project. It declares a .NET 10 (Windows) test assembly built on MSTest, with Moq for mocking, and wires the project up to the two production layers under test: `BusinessLayer` and `UserManagement`.

### Role in the Codebase

It is not a build or deployment tool — it simply defines how the two C# files in `TestProject1/` (`UserViewModelTest.cs` and `MSTestSettings.cs`) are compiled into a test project. Those tests exercise `UserManagement.ViewModel.UserViewModel` (and its `ShowUsersViewModel` / `AddUserViewModel` children), using mocks of `IUserService` and `IMessageService`.

If the file is removed, the solution has no test project: the `UserViewModelTest` and `MSTestSettings` sources would compile nowhere.

### Relationships

- **`Microsoft.NET.Test.Sdk` 18.4.0** — the test host; required for MSTest to be discoverable by `dotnet test`.
- **`MSTest` 4.2.1** — the test framework, supplying the `[TestClass]`, `[TestMethod]`, `[DataRow]`, `[TestInitialize]`, `[TestCleanup]`, `Assert`, and assembly-level `Parallelize` attributes used by the test sources.
- **`Moq` 4.20.72** — mocking library; `Mock<IUserService>` and `Mock<IMessageService>` in `UserViewModelTest.cs` depend on it. Note the test file also uses `CommunityToolkit.Mvvm` (`WeakReferenceMessenger`) — it reaches it transitively through the `UserManagement` project reference, not a direct package here.
- **Project references** — `..\BusinessLayer\BusinessLayer.csproj` (provides `IUserService` and `BusinessLayerException`, the type the tests verify) and `..\UserManagement\UserManagement.csproj` (provides `UserViewModel`, `ShowUsersViewModel`, `AddUserViewModel`, and `IMessageService`).
- **Global using** — `<Using Include="Microsoft.VisualStudio.TestTools.UnitTesting" />` adds the MSTest namespace to every source file in the project, so the test classes don't need explicit `using` directives for `TestClassAttribute` etc.
- **Test discovery** — `MSTestSettings.cs` carries `[assembly: Parallelize(...)]`, which only takes effect because this project references MSTest.

### Behavior & Details

- `TargetFramework` is `net10.0-windows`, so the tests build and run only on Windows with the .NET 10 SDK.
- `Nullable` is enabled and `LangVersion` is `latest`; the test sources already use the null-forgiving operator (`= null!`) accordingly.
- The `<Using Include="...">` item is the only non-standard element: it is the MSBuild mechanism that makes `Microsoft.VisualStudio.TestTools.UnitTesting` implicitly available project-wide, so `MSTestSettings.cs` (a one-line file with no `using`) still resolves the `Parallelize` assembly attribute.

### Rough edges

The assembly-level `[assembly: Parallelize(Scope = ExecutionScope.MethodLevel)]` in `MSTestSettings.cs` and the class-level `[DoNotParallelize]` in `UserViewModelTest.cs` are mutually exclusive at class scope — the class attribute wins for this class, so the assembly attribute currently has nothing to apply to. It would only matter if a future test class didn't opt out.