# ConsoleApp1/Program.cs

The `Program` class — the `Main` entry point of the ConsoleApp1 console application.

### Purpose

This is the executable's entry point for a headless (no UI) client of the `UserManagement` module. It exists to demonstrate and exercise the full MEF wiring end to end: build a composition container from the assemblies in the output directory, resolve the user-management view model, create 100 users through `AddUserViewModel.SaveUserAsync()`, then load the active-user list through `ShowUsersViewModel.OnLoadedAsync()` and print every user's name to the console. The only output of the whole program is the one `Console.WriteLine` per user at the end.

### Role in the Codebase

`ConsoleApp1` is one of three host applications in the solution (alongside `WinFormsApp1` and `WpfApp1`). The console host is the simplest of the three: it contains no views or views' code at all, just this entry point, `ConsoleApp1.csproj`, and `App.config`. It is the composition root for this host — the place where the container is created and the first export is resolved. The other two hosts do the same kind of wiring for their own UIs (for instance `WinFormsApp1/View/UserForm.cs` resolves `IUserViewModel` for data binding). Remove this program and the repository loses its command-line way to run the user-management feature and verify the MEF composition at all; nothing else depends on the `Program` type itself.

The project reference set is `Common` (composition/reflection helpers), `Mvvm.Console`, and `UserManagement` (the feature's view models and their exports).

### Relationships

- `Common.Composition.CompositionHelper` (Common/Composition/CompositionHelper.cs) — builds the MEF container. `Program` calls the `CreateCompositionContainer(string[] ignoreAssemblies, ...)` overload. That method scans the directory of `Common`'s own assembly for `*.dll`/`*.exe` files (per `Common.Reflection.AssemblyHelper.Extensions`), skips any file whose name is in `ignoreAssemblies` (case-insensitive), loads each surviving file with `Assembly.LoadFrom` into an `AggregateCatalog`, then also adds the executing assembly and the entry assembly, and finally composes the container with itself exported. Note it silently `Console.WriteLine`s and skips any assembly it fails to load.
- `UserManagement.ViewModel.IUserViewModel` (UserManagement/ViewModel/IUserViewModel.cs) — the contract the program resolves. The concrete class `UserViewModel` (UserManagement/ViewModel/UserViewModel.cs) carries `[Export(typeof(IUserViewModel))]` and `[Export(typeof(IMainViewModel))]`, and its two child properties (`AddUserViewModel`, `ShowUsersViewModel`) are MEF `[Import]`ed, so resolving `IUserViewModel` transitively composes the rest of the feature graph (`AddUserViewModel`, `ShowUsersViewModel`, `IUserService`, `ILoggingService`, `IMessageService`).
- `AddUserViewModel` / `ShowUsersViewModel` (UserManagement/ViewModel/) — the two children the program drives. `AddUserViewModel.SaveUserAsync()` validates, calls `UserService.AddUser(...)`, broadcasts a `UserChangedMessage` via `WeakReferenceMessenger`, and clears its own fields; `ShowUsersViewModel.OnLoadedAsync()` calls `UserService.GetActiveUsers()` and fills its `Users` collection, which also listens for `UserChangedMessage`.
- Configuration: `App.config` supplies the `ConnectionString` (`Server=localhost;Database=Users;...`), which the data-access side of the feature consumes at runtime. The program itself never reads configuration.

### Behavior & Details

Startup, in order:

1. **Build the container** inside a `using` block, calling `CompositionHelper.CreateCompositionContainer(["ConsoleApp1.exe"])`. Passing `["ConsoleApp1.exe"]` tells the helper to skip the program's own executable file when scanning the output directory for assemblies.
2. **Resolve the feature root**: `container.GetExportedValues<IUserViewModel>().SingleOrDefault()`. A single `IUserViewModel` export is expected; `SingleOrDefault()` implies a second export would itself throw.
3. **Create 100 users**: for each of 100 iterations, set `FirstName = "Firstname"` and `LastName = "Lastname"` on `AddUserViewModel`, then `await SaveUserAsync()`. So all 100 records written are identical.
4. **Refresh the list**: `await ShowUsersViewModel.OnLoadedAsync()` loads active users from the service.
5. **Print** one line per user: `Name: {FirstName} {LastName}`.

Notable details:

- No error handling around the 100 saves. `SaveUserAsync` rethrows `BusinessLayerException` (after catching it) and otherwise lets validation-failure paths early-return; an exception from any one save aborts the whole program.
- The console's stdout also carries diagnostics from inside `CompositionHelper` (one `Console.WriteLine` per assembly that fails to load), intermixed before the user list is printed.
- The container (and with it the database-backed services) is disposed when the `using` block exits, i.e. immediately after printing.

Rough edges:

- The inline comment says `"ConsoleApp1.exe" is not a valid .net assembly`, but that is not the mechanism: the helper's `ignoreAssemblies` list is a name-based skip filter, so the entry point executable is simply not loaded as a MEF catalog (which is also why the helper later adds the entry assembly separately).
- The `if (userViewModel == null)` throw is unreachable: `SingleOrDefault()` already throws for zero matches, so the `GetExportedValues<...>().SingleOrDefault()` result is either an instance or a thrown exception, never null.
- `SaveUserAsync()` clears `FirstName`/`LastName` after each save, which the program overwrites on every loop iteration — harmless here, but it means the program's loop is doing redundant work the view model would otherwise do.