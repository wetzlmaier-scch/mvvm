# `Mvvm/IMainViewModel.cs`

### Purpose

This file defines `IMainViewModel`, the contract for the view model bound to the application's main window. A plain view model only exposes data; this interface additionally marks a view model as the one the application shell treats as "the main window" — it can display a window icon and it can control window and application lifetime (`CloseWindow()`, `Shutdown(int exitCode)`). It exists so the host controllers can be written against an abstraction instead of a concrete feature's view model.

### Role in the Codebase

`IMainViewModel` is one of the MVVM contracts in the shared `Mvvm` project. It is deliberately thin — three members only — because the behaviour it declares is implemented once, in the abstract base class `MainViewModelBase` (`Mvvm/MainViewModelBase.cs`). Concrete feature view models inherit from `MainViewModelBase` and advertise themselves as the main view model by adding `[Export(typeof(IMainViewModel))]`, which the MEF container resolves when a host builds its `DefaultModuleController`.

The interface is what lets any module act as the application shell. If it were removed, the controllers in both host apps (`WpfApp1` and `WinFormsApp1`) could no longer receive the main view model through constructor injection, and the `[Export(typeof(IMainViewModel))]` attribute on `UserViewModel` would have nothing to export against.

### Relationships

- **`IViewModel`** (`Mvvm/IViewModel.cs`) — the parent interface. It extends `INotifyPropertyChanged` and declares no members of its own, so `IMainViewModel` simply inherits the change-notification contract.
- **`MainViewModelBase`** (`Mvvm/MainViewModelBase.cs`) — the only class in the codebase that implements `IMainViewModel` directly (as an abstract base). It provides the concrete behaviour for all three members.
- **`UserViewModel`** (`UserManagement/ViewModel/UserViewModel.cs`) — the concrete implementation: it extends `MainViewModelBase` and carries `[Export(typeof(IMainViewModel))]` alongside `[Export(typeof(IUserViewModel))]`.
- **`DefaultModuleController`** in both `WpfApp1/Controller/` and `WinFormsApp1/Controller/` — the callers. Each has an MEF `[ImportingConstructor]` that takes an `IMainViewModel`, stores it in a `protected MainViewModel` property, and assigns it to the main view as its `DataContext`.

### Behavior & Details

The three members:

- **`Icon`** (`ImageSource?`) — the main window's icon. `MainViewModelBase`'s constructor loads it from the application's executable icon via `AssemblyHelper.LoadApplicationIcon()` and converts it to a `BitmapSource`, skipping this in design mode.
- **`CloseWindow()`** — closes the application's main window. In `MainViewModelBase` this calls `Application.Current?.MainWindow?.Close()` (WPF `System.Windows.Application`), which raises the `Closing` event that the base class also hooks to call `OnClosing()`/`OnClosed()`.
- **`Shutdown(int exitCode)`** — shuts the whole application down with a specific exit code. In `MainViewModelBase` this calls `Application.Current?.Shutdown(exitCode)` and does nothing in design mode.

One caveat worth knowing before you rely on these methods: the host controllers never actually call `CloseWindow()` or `Shutdown(int)`. Both `DefaultModuleController` classes call `MainView.Close()` directly in their `Shutdown()` method, so the interface's window-lifecycle members are only reachable from code inside the view model itself (for example, a view model that decides on its own to close the window or exit with a code). The interface documents the capability; it does not guarantee the host will invoke it.

Note also that although this file lives in the framework-agnostic `Mvvm` namespace, it imports `System.Windows.Media` (for `ImageSource`), so `MainViewModelBase`'s implementations are WPF-oriented.