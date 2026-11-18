# `Mvvm/ViewModelBase.cs`

### Purpose

This file defines `ViewModelBase`, the abstract root of the application's view model hierarchy in the `Mvvm` namespace. It exists to give every view model three shared capabilities: WPF data-binding change notification (inherited from `ObservableObject`), a static `IsInDesignMode` flag so logic can be skipped while the app runs inside the Visual Studio designer, and a parent/child view model registration mechanism with paired `OnClosing`/`OnClosed` hooks that propagate through the hierarchy.

It implements `IViewModel` (defined in `Mvvm/IViewModel.cs`, which is simply `INotifyPropertyChanged` with no additional members), so any `ViewModelBase` is usable where the interface is expected.

### Role in the Codebase

`ViewModelBase` is the foundational base class of the `Mvvm` layer. Three abstract classes build directly on it:

- `DisposableViewModelBase` — adds `IDisposable` semantics (a `Disposables` collection, `CheckIfDisposed`, `Disposed` event).
- `MainViewModelBase` — represents the view model of the main window and is the place that actually wires the child lifecycle into the window: it subscribes to `MainWindow.Closing` in its constructor and, in its `OnClosing(CancelEventArgs)` handler, calls the no-argument `OnClosing()` to ask the child tree whether the window may close (`e.Cancel = true` if any child says no), then calls `OnClosed()`.
- `ValidatableViewModelBase<T>` — adds validation via `INotifyDataErrorInfo`.

Concrete view models such as `ShowUsersViewModel` also derive directly from `ViewModelBase`. Removing this class would break the entire `Mvvm` layer, since everything above in the hierarchy inherits from it.

### Relationships

- **`CommunityToolkit.Mvvm.ObservableObject`** (external library): the base class; supplies `INotifyPropertyChanged` and the `SetProperty`/`[ObservableProperty]` machinery that concrete view models use. `ViewModelBase` inherits it but adds no properties of its own.
- **`CommunityToolkit.Diagnostics.Guard`** (external library): `RegisterChildViewModel`/`UnregisterChildViewModel` use `Guard.IsNotNull`.
- **`Common.Diagnostics.ExtendedGuard`** (`Common/Diagnostics/ExtendedGuard.cs`): project-local guard helpers. `ExtendedGuard.NotContains` rejects registering a child that is already registered; `ExtendedGuard.Contains` rejects unregistering one that is not. Both throw `ArgumentException` on failure. Note these helpers format the message with `'{name}'` but never interpolate the actual argument name, so the name passed in is not included in the message.
- **`System.Windows`** (`DesignerProperties`, `DependencyPropertyDescriptor`, `FrameworkElement`): used by `IsInDesignModeStatic` to read the designer-mode property at runtime.
- **Downstream callers:** `MainViewModelBase` is the only place that invokes `OnClosing()`/`OnClosed()`, so the child-tree propagation only runs when a `MainViewModelBase` is present in the hierarchy.

### Behavior & Details

**`IsInDesignModeStatic`** — A lazily initialized static flag. On first read it reads the default value of WPF's `DesignerProperties.IsInDesignModeProperty` via `DependencyPropertyDescriptor`. Since that property can report `false` in some designer scenarios, it also checks whether the current process name starts with `devenv` (the Visual Studio host process) and forces `true` in that case. The result is cached in `_isInDesignMode` for the lifetime of the process; the instance property `IsInDesignMode` is just a convenience pass-through to the static.

**Child view model registration**

- `RegisterChildViewModel(viewModel)` adds a child to `_childViewModels`. Guards: non-null and not already registered (duplicates throw `ArgumentException`).
- `UnregisterChildViewModel(viewModel)` removes a child. Guards: non-null and currently registered (unregistering an unknown child throws).

**`OnClosing()` / `OnClosed()`** — Both are `protected virtual` and are *not* invoked by `ViewModelBase` itself:

- `OnClosing()` aggregates the answers of all children: it returns `true` only if every child's `OnClosing()` returns `true` (`Aggregate(true, ... & ...)`). The default implementation therefore returns `true` (no children → closable). A subclass overrides this to veto closing.
- `OnClosed()` is a notification that the view model has been closed; the default implementation recursively calls `OnClosed()` on each child. A subclass overrides it for cleanup.

The propagation is depth-first and happens in the order children were registered.

**Non-obvious control flow:** the child hooks only fire because `MainViewModelBase`'s constructor subscribes to `MainWindow.Closing` and translates it into `OnClosing()` → (if allowed) `OnClosed()`. For view models under a plain `ViewModelBase` (e.g. `ShowUsersViewModel`) that are not in the `MainViewModelBase` subtree, nothing calls these methods unless a concrete class does so explicitly.

### Rough edges

- `ExtendedGuard.Contains`/`NotContains` take a `name` parameter that is never used in the exception message, so the `ArgumentException`s thrown by the register/unregister guards do not identify which argument failed.
- The `devenv` process-name check only catches the Visual Studio process; other host processes that run the app in a design-like state (e.g. Blazor/XAML tooling, tests launched from a VS host) would still see `IsInDesignModeStatic` as `false`.