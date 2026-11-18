### Purpose

`Mvvm/IMainViewModel.cs` defines `IMainViewModel`, the contract for the view model bound to the application's main window. Ordinary view models only expose data; this interface marks a view model as the main window's and adds the window-level operations the host can perform against it: reading the window icon, closing the window, and shutting the whole application down with an exit code.

The interface itself contains no logic. Its value is as a lookup key and a capability marker: code that wants to drive the main window depends on this contract rather than on any concrete view model.

### Role in the Codebase

It is one of the shared abstractions in the `Mvvm` project (alongside `IViewModel`, `IMessageService`, `ViewModelBase`, etc.), a small framework layer that the concrete host projects (`WpfApp1`, `WinFormsApp1`) and feature projects (`UserManagement`) build on. Because the interface names no specific host, the same contract works under both WPF and WinForms hosts.

If it were removed, `MainViewModelBase` (`Mvvm/MainViewModelBase.cs`) would have nothing to implement, and the `DefaultModuleController` classes in both host projects would have no type to import for the main view model.

### Relationships

- **Extends `IViewModel`** (`Mvvm/IViewModel.cs`). `IViewModel` itself extends `INotifyPropertyChanged` and declares no members, so inheriting it simply marks a type as a change-notifying view model; everything here (`Icon`, `CloseWindow`, `Shutdown`) is new.
- **Implemented via `MainViewModelBase`** (`Mvvm/MainViewModelBase.cs`), an abstract class that provides the default implementations of all three members. App-specific view models inherit from `MainViewModelBase` rather than implementing the interface directly.
- **Resolved through MEF.** A view model advertises itself as the main one with `[Export(typeof(IMainViewModel))]` — e.g. `UserManagement/ViewModel/UserViewModel.cs` does this in addition to its `IUserViewModel` export. The host's `DefaultModuleController` (`WpfApp1/Controller/DefaultModuleController.cs` and `WinFormsApp1/Controller/DefaultModuleController.cs`) receives an `IMainViewModel` through its `[ImportingConstructor]` and sets it as the view's data context. The WPF controller also calls `Shutdown()` when the main window closes.

### Behavior & Details

Three members, each describing a window-level operation:

- **`ImageSource? Icon { get; }`** — the icon to display on the main window. Nullable by design: `MainViewModelBase` populates it from the executing assembly's icon, but only if loading succeeds, so a missing icon leaves it `null`.
- **`CloseWindow()`** — closes the application's main window. In the base implementation this is `Application.Current?.MainWindow?.Close()`; because closing the main window tears down the WPF application, this is effectively "quit the UI". It returns without doing anything in design mode or if no application context exists.
- **`Shutdown(int exitCode)`** — shuts down the whole application, passing `exitCode` to `Application.Current?.Shutdown(exitCode)` so the process exits with the caller's code. Like `CloseWindow`, it is a no-op in design mode.

The concrete behaviour of all three members lives in `MainViewModelBase`, which hooks the main window's `Closing` event in its constructor to give view models a chance to cancel close (via `OnClosing`) and to run `OnClosed()` after. A change to how these members should behave belongs in `MainViewModelBase`, not in this interface.

One caveat: a host is not obliged to call these members. The WinForms `DefaultModuleController`, for instance, imports `IMainViewModel` but never invokes `CloseWindow()` or `Shutdown()`, so their contract is only exercised where a host actually uses them.