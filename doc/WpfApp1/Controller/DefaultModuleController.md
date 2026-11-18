# `WpfApp1/Controller/DefaultModuleController.cs`

## Purpose

`DefaultModuleController` is the WPF application's lifecycle controller. It is a single, minimal implementation of `IModuleController` (defined in `Mvvm/IModuleController.cs`) that exists so the WPF host has a module to load. Its whole job is to take the main view and main view model handed to it by MEF, bind them together, and map the three lifecycle callbacks (`Initialize`, `Run`, `Shutdown`) to showing and closing the main window.

## Role in the codebase

The WPF host is `App` (`WpfApp1/App.xaml.cs`), which is an empty subclass of `MvvmApplication` (`Mvvm.Wpf/MvvmApplication.cs`). `MvvmApplication.OnStartup` builds a MEF `CompositionContainer`, resolves every exported `IModuleController`, calls `Initialize()` on all of them, then `Run()` on all of them, and finally `Shutdown()` on all of them (in reverse order) in `OnExit`. This class is the only `IModuleController` export in `WpfApp1` — if it were removed, `MvvmApplication` would find no modules, show a "No module found." error box, and shut down with an error exit code.

## Relationships

- **`IModuleController`** (`Mvvm/IModuleController.cs`) — the three-method lifecycle contract this class implements: `Initialize()`, `Run()`, `Shutdown()`.
- **`IMainView<TDialogResult, TDimension>`** (`Mvvm/IMainView.cs`) — extends `IDialogView<TDialogResult>`, which declares `Show()` and `Close()`. This class receives it as `IMainView<bool?, double>` and calls those two methods. The concrete export in this project is `UserView` (`WpfApp1/View/UserView.xaml.cs`), a `Window` exported as `IMainView<bool?, double>`.
- **`IMainViewModel`** (`Mvvm/IMainViewModel.cs`) — extends `IViewModel`; this class receives it, holds a reference, and assigns it as the view's `DataContext` in the constructor. Its concrete implementation is injected by MEF.
- **MEF (`System.ComponentModel.Composition`)** — the class is discovered via `[Export(typeof(IModuleController))]` and constructed via its `[ImportingConstructor]`-marked constructor.

The caller is `MvvmApplication`: it never references this class by name, only through the `IModuleController` interface resolved from the container.

## Behavior & Details

- **Constructor** — The `[ImportingConstructor]` tells MEF to inject `IMainView<bool?, double>` and `IMainViewModel` here. Beyond storing them in the two `protected` properties, the constructor wires `MainView.DataContext = mainViewModel`. That binding is the only setup the module does; nothing happens in `Initialize()`.

- **`Initialize()`** — Empty by design; the constructor already did all the work.

- **`Run()`** — Calls `MainView.Show()`, which displays the main window.

- **`Shutdown()`** — Calls `MainView.Close()`. In practice the host reaches `Shutdown()` from `MvvmApplication.OnExit`, which fires when the application is exiting (typically when the main window closes). So `Close()` here is a second close on an already-closing/closed window — harmless, but the real exit is driven by the user closing the window, not by this method.

- All three lifecycle methods are `virtual`, so a subclass could extend the WPF app's startup/shutdown behavior by overriding them, while keeping the MEF export of the base type.

- The `protected` `MainView` and `MainViewModel` properties are there for exactly that: a derived controller can access the wired view and view model.

## Rough edges

- `Shutdown()` calls `MainView.Close()`, but in this host the window is already closing when `OnExit` runs, so the call is effectively a no-op — the module never initiates shutdown on its own.

- The generic parameters on the view (`IMainView<bool?, double>`: dialog result, dimension type) are fixed here and must match the export in `UserView.xaml.cs`; a mismatch between the two files means MEF cannot satisfy the constructor and the container will fail to build this module.