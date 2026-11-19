# UserManagement/ViewModel/IUserViewModel.cs

### Purpose

`IUserViewModel` is the top-level view-model contract for the UserManagement feature. It exists so host code can ask the MEF composition container for "the user management feature as a whole" without knowing which concrete class fulfils it. The interface declares exactly two read-only properties — `IAddUserViewModel AddUserViewModel` and `IShowUsersViewModel ShowUsersViewModel` — making the contract a facade over the two child screens of the feature: adding a user and listing users.

### Role in the Codebase

It is the entry-point contract of the `UserManagement` module, sitting alongside the concrete `UserViewModel` (same folder) that implements it. The composition root resolves the feature by this interface: `ConsoleApp1/Program.cs` builds a MEF container and calls `container.GetExportedValues<IUserViewModel>().SingleOrDefault()`, throwing `IUserViewModel not found.` if nothing is exported. In the WinForms host, `WinFormsApp1/View/UserForm.cs` overrides the `DataContext` setter and casts the received value to `IUserViewModel` to wire up its data bindings. Remove this interface and both of those resolution paths break — the feature would no longer be addressable by name from the outside.

### Relationships

- **Implementor:** `UserViewModel` (`UserManagement/ViewModel/UserViewModel.cs`) implements the interface and registers itself with `[Export(typeof(IUserViewModel))]` (it is additionally exported as `IMainViewModel` so the shell can resolve it as the main window's view model). Its two properties are MEF `[Import]`s, which the interface exposes as read-only gets.
- **Member types:** `IAddUserViewModel` and `IShowUsersViewModel`, both defined in the same folder. `IUserViewModel` only exposes them for retrieval; the actual capability contracts (e.g. `SaveUserAsync()`, `Users`) live on those two interfaces.
- **Callers:** anything that needs the feature entry point by contract — the console composition root and the WinForms `UserForm`, which reads `AddUserViewModel` and `ShowUsersViewModel` to bind text boxes and the user list.
- The `using System.ComponentModel.Composition;` import in this file is unused; it appears in the sibling files where `[Export]`/`[Import]` are actually applied.

### Behavior & Details

The interface contains no methods, events, or logic — only the two properties. Two conventions are worth knowing:

- **Read-only on the contract, writable underneath.** The interface declares get-only properties; the implementor `UserViewModel` has `{ get; set; }` so MEF can inject the child view models. Consumers should treat both properties as immutable references to already-composed objects, never replace them.
- **Single export expected.** `Program.cs` uses `SingleOrDefault()` when resolving, so the design assumes exactly one `IUserViewModel` export exists in the container. A second class carrying `[Export(typeof(IUserViewModel))]` would make that call throw an `InvalidOperationException` at startup rather than silently picking one.

### Rough edges

The `using System.ComponentModel.Composition;` directive is dead in this file — no MEF attribute is applied here.