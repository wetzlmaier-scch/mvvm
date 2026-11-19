# `ConsoleApp1/Program.cs`

### Purpose

This is the entry point of the console host (`ConsoleApp1`, an `Exe` per its `.csproj`). It exists to prove that the application can be run headless: it builds the MEF (MEF) dependency container, resolves the root `UserViewModel`, creates 100 users through the add-user view model, then loads and prints the user list through the show-users view model. It is the executable counterpart of the WPF and WinForms hosts in the solution — same composition mechanism, same view models, no UI.

### Role in the Codebase

`Program.cs` is the entire program of `ConsoleApp1`: a single `Program` class with one `static async Task Main(string[] args)`. It plays the composition-root role for this host, just as the WPF/WinForms hosts do for theirs. Removing it (or its `Main`) makes `ConsoleApp1` non-buildable as an executable; nothing else in the repository calls into it.

### Relationships

- `Common.Composition.CompositionHelper` (`Common/Composition/CompositionHelper.cs`) — builds the `CompositionContainer`.
- `UserManagement.ViewModel.IUserViewModel` (`UserManagement/ViewModel/IUserViewModel.cs`) — the contract it resolves. The sole concrete export is `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`), which `[Export(typeof(IUserViewModel))]`s itself and imports `AddUserViewModel` and `ShowUsersViewModel` into it.
- `IAddUserViewModel` / `IShowUsersViewModel` and their implementations `AddUserViewModel`, `ShowUsersViewModel` (`UserManagement/ViewModel/`) — the view models actually driven from `Main`. `AddUserViewModel.SaveUserAsync()` calls `UserService.AddUser(...)` and broadcasts a `UserChangedMessage` via `WeakReferenceMessenger`; `ShowUsersViewModel.OnLoadedAsync()` calls `UserService.GetActiveUsers()` and fills its `Users` collection.
- The `IUserService` and `ILoggingService` implementations are not resolved by name here — they are discovered by the container from the assemblies in the output directory. Their concrete classes were not inspected; the `ConnectionString` used at runtime comes from `ConsoleApp1/App.config` (`Server=localhost;Database=Users;...`), but how the service reads it is defined outside this file.

### Behavior & Details

`Main` runs a short sequence, all inside a `using` block that disposes the MEF container at the end:

1. `CompositionHelper.CreateCompositionContainer(["ConsoleApp1.exe"])` scans the directory of the *executing* assembly (`Common.dll`) for loadable assembly files, skipping any whose lower-cased name is in the ignore list. Unloadable files are skipped silently — `CompositionHelper` catches each load exception and prints its message to the console — so a partially failed assembly scan does not stop startup.
2. `container.GetExportedValues<IUserViewModel>().SingleOrDefault()` resolves the root view model; if it is `null`, the program throws `Exception("IUserViewModel not found.")`. Note `SingleOrDefault()` also throws if more than one `IUserViewModel` export ever exists — today there is exactly one (`UserViewModel`).
3. A loop of 100 iterations sets `AddUserViewModel.FirstName = "Firstname"` / `LastName = "Lastname"` and awaits `SaveUserAsync()`. Each save goes through validation, persists a `UserDto` via `IUserService`, broadcasts the `UserChangedMessage` that `ShowUsersViewModel`'s messenger subscription uses to append to `Users`, and clears the fields — so all 100 records have the identical name.
4. `ShowUsersViewModel.OnLoadedAsync()` fetches active users from the service and adds them to `Users` (on top of any already added via the messages).
5. The program prints each user as `Name: {FirstName} {LastName}`.

Edge cases worth knowing: validation failure inside `SaveUserAsync` only reports "User data is invalid!" via the message service and returns without throwing, so the 100-iteration loop cannot detect it; `BusinessLayerException`s are rethrown and will crash `Main`. The `IUserService` is shared by both view models, so the exact final output depends on whether the persistence layer treats the 100 duplicate saves as 100 rows or fewer.

### Rough edges

- `CreateCompositionContainer(["ConsoleApp1.exe"])` treats its first argument as a list of assemblies to *ignore*, not a module to load — the comment in the code ("the ConsoleApp1.exe is not a valid .net assembly") is the only explanation, and since `ConsoleApp1` is already picked up by the directory scan as the entry assembly, the ignore list is effectively redundant here.
- The 100 users are indistinguishable from one another (same first/last name every iteration), so the printed output cannot show per-iteration state.