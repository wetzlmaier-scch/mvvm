### Documentation

**Path:** `ConsoleApp1/Program.cs`

### Purpose

The entry point of the console application — `Main` builds a MEF (Managed Extensibility Framework) dependency-injection container from the assemblies in the application directory, pulls in the user-management view models, and runs a small demo: it saves the same user 100 times, then loads all active users and prints each one to the console. It exists to exercise the `UserManagement` view models end-to-end without a GUI, in the same role that `WinFormsApp1/Program.cs` and `WpfApp1/App.xaml.cs` play for their respective hosts.

### Role in the Codebase

`ConsoleApp1` is one of three thin application hosts in the repository (console, WinForms, WPF). The real behaviour lives in the shared layer — `UserManagement/ViewModel`, `Mvvm`, `BusinessLayer`, and the `DataAccessLayer*` projects — which this program consumes. Removing this file only removes the console demo; the other hosts and all shared code are unaffected.

### Relationships

- `Common/Composition/CompositionHelper.cs` — `CompositionHelper.CreateCompositionContainer(ignoreAssemblies)` scans the directory containing `Common`'s assembly for `.dll`/`.exe` files (per `AssemblyHelper.Extensions`), loads each one, and builds a `CompositionContainer` over an aggregate of `AssemblyCatalog`s. Every non-trivial load failure is swallowed with a `Console.WriteLine` of the message, so a missing dependency manifests later as a missing export rather than at startup.
- `UserManagement/ViewModel/IUserViewModel.cs` — the contract the program resolves. Implemented by `UserViewModel`, which is `[Export]`ed as both `IUserViewModel` and `IMainViewModel` and has `[Import]`ed `IAddUserViewModel` and `IShowUsersViewModel` members.

No other project references `ConsoleApp1` or its `Program` class; it is a leaf.

### Behavior & Details

`Main` is `async` and runs under a `using` block, so the composition container is disposed when the method returns. The sequence is:

1. **Build the container.** `CompositionHelper.CreateCompositionContainer(["ConsoleApp1.exe"])` loads every `.dll` and `.exe` next to `Common`'s assembly into an `AggregateCatalog`, skipping the file named `ConsoleApp1.exe` (case-insensitive), and adds the executing and entry assemblies if not already cataloged. Because the ignore check compares file names, this excludes whichever assembly happens to be named `ConsoleApp1.exe` from the MEF scan — the executing/entry assemblies are cataloged separately just below.
2. **Resolve the view model.** `container.GetExportedValues<IUserViewModel>().SingleOrDefault()` returns the `UserViewModel` export. `SingleOrDefault()` means a *second* `IUserViewModel` export would throw on the ambiguity. The following `if (userViewModel == null)` guard is a dead check: `SingleOrDefault()` never returns `null` here — it returns the single element or throws — so "IUserViewModel not found." can in practice only surface as an unhandled `InvalidOperationException` from `SingleOrDefault()`.
3. **Insert 100 users.** The loop writes the literal strings `"Firstname"` / `"Lastname"` into `AddUserViewModel.FirstName`/`LastName` and awaits `SaveUserAsync()` each iteration. Each save: validates via `UserValidator`, calls `UserService.AddUser` with a `UserDto`, broadcasts a `UserChangedMessage` over `WeakReferenceMessenger`, and clears the fields — so `ShowUsersViewModel`'s messenger handler appends a `UserModel` for every single save. A `BusinessLayerException` is rethrown and propagates out of `Main`.
4. **Load and print.** `OnLoadedAsync()` fetches active users from `UserService` and appends them to `ShowUsersViewModel.Users`. Then `Main` iterates that collection and writes one `Name: {FirstName} {LastName}` line per entry. Because `Users` was already populated by the per-save `UserChangedMessage` in step 3, `OnLoadedAsync` *re-adds* the same users, so the output contains duplicates: roughly 2 × 100+ names (or more, if any of the 100 adds are rejected as duplicates by `UserService`).

### Rough edges

- The "IUserViewModel not found." throw is unreachable: `SingleOrDefault()` throws on zero matches, so the null branch never executes.
- The `ConsoleApp1.exe` entry in the ignore list excludes whatever assembly is actually named `ConsoleApp1.exe` from the MEF catalog; with a typical build name (`ConsoleApp1.dll`) the ignore string matches nothing. The comment in the code calls the value invalid, but its net effect depends on the deployed file name rather than on any deliberate exclusion.
- `Main` ignores `args` entirely, and the 100 saved users are hard-coded literals — the program is a fixed smoke/demo run, not a parameterizable one.