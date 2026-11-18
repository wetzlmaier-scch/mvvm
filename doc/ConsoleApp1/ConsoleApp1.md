# ConsoleApp1.csproj

`ConsoleApp1/ConsoleApp1.csproj` — MSBuild project file for the `ConsoleApp1` project.

### Purpose
Build file for `ConsoleApp1`, the console host of the demo solution. Declares that `Program.cs` compiles into an executable targeting `net10.0-windows`, with implicit usings and nullable reference types enabled.

### Role in the codebase
Alongside `Program.cs` and `App.config`, this is what makes `ConsoleApp1` a buildable project in `Demo.slnx`. The solution contains three app hosts — `ConsoleApp1`, `WinFormsApp1`, `WpfApp1` — and this file's job is to be the console one: it wires the shared `UserManagement` module into a plain terminal app rather than a UI. Without it, `Program.cs` would not build.

### Relationships
The `<ProjectReference>` entries define the runtime composition this host depends on:

- `Common/Common.csproj` — provides `Common.Composition.CompositionHelper`, the MEF composition bootstrap that `Program.cs` calls to build its container from `ConsoleApp1.exe`.
- `Mvvm.Console/Mvvm.Console.csproj` — the console view/adapter layer of the demo's MVVM framework; its parts are discovered by MEF during composition.
- `UserManagement/UserManagement.csproj` — the shared user-management module whose `IUserViewModel` export the program resolves and drives.

The `net10.0-windows` target makes the project Windows-only at the framework level (the console code itself doesn't visibly use Windows APIs).

### Behavior & Details
No logic lives in a `.csproj`; its behavior is the build configuration:

- `OutputType=Exe` — produces `ConsoleApp1.exe`, which `Program.cs` deliberately feeds back to the composition container as its MEF assembly directory.
- Transitive references flow in automatically through the three project references, which is how the program reaches the full dependency graph (data access, entities, logging) without listing them here.

### Rough edges
`Program.cs` passes `ConsoleApp1.exe` to the composition container, but that is not a valid .NET assembly (the assembly is `ConsoleApp1.dll`); the code comment admits it. The project file is unaffected, but the host's MEF composition is fragile around this value.