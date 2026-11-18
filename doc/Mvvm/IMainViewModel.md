# Mvvm/IMainViewModel.cs

### Purpose

`IMainViewModel` is the contract for the main window's view model. It marks a view model as the one bound to the application's main window, and — beyond ordinary view-model behaviour — gives controllers the ability to close that window or shut the whole application down with an exit code. It exists so that host-specific code (module controllers in the WPF and WinForms host apps) can drive the application's lifecycle through an interface rather than a concrete view model class.

### Role in the Codebase

This interface is a small piece of the shared `Mvvm` namespace — the cross-platform view-model layer consumed by both host applications. It defines the role, not the behaviour: the only implementation is `MainViewModelBase` (in `Mvvm/MainViewModelBase.cs`), an abstract class that app-specific main view models derive from. Removing it would break every module controller in the host apps, which all depend on this type (see below).

### Relationships

- **Extends `IViewModel`** (defined in `Mvvm/IViewModel.cs`). `IViewModel` itself extends `INotifyPropertyChanged` and declares no members, so inheriting it simply marks this type as a change-notifying view model. `IMainViewModel` adds the window-lifecycle surface on top.
- **Implemented by `MainViewModelBase`** (`Mvvm/MainViewModelBase.cs`), which itself extends `ViewModelBase` (in `Mvvm/ViewModelBase.cs`). The interface is the point through which consumers see those implementations.
- **Consumed by the module controllers** in the host applications, which are composed via MEF:
  - `WinFormsApp1/Controller/DefaultModuleController.cs` receives `IMainViewModel` through its `[ImportingConstructor]`, stores it in `MainViewModel`, and assigns it as `MainView.DataContext`.
  - `WpfApp1/Controller/DefaultModuleController.cs` does the same.
  - App-specific view models advertise themselves as implementations with `[Export(typeof(IMainViewModel))]` (e.g. `UserManagement/ViewModel/UserViewModel.cs`), which is what the MEF composition matches against.

### Behavior & Details

The interface declares one property and two methods:

- `ImageSource? Icon` — the window icon as WPF imaging. `MainViewModelBase` fills it in from the application's executable icon (via `AssemblyHelper.LoadApplicationIcon` and `BitmapSourceHelper.LoadBitmap`); it stays `null` if no icon loads, hence the nullable type.
- `CloseWindow()` — closes the main window. In `MainViewModelBase` this calls `Application.Current?.MainWindow?.Close()` and is a no-op in design mode. Note this goes through the window's `Closing` event, so a view model can still cancel the close (the base class routes it through `OnClosing`).
- `Shutdown(int exitCode)` — shuts down the whole application, propagating `exitCode`. In `MainViewModelBase` this calls `Application.Current?.Shutdown(exitCode)`.

One caveat for anyone relying on this contract: both methods depend on `Application.Current` at call time (WPF's `Application`, specifically). The interface does not say how the current `Application` is reached, and the base implementation's behaviour — and its no-op in design mode — is an implementation detail callers shouldn't need to know but do in practice.