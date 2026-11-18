# Common/Diagnostics/ExtendedGuard.cs

### Purpose

`ExtendedGuard` is a small static helper that asserts a collection does or does not contain a given item, throwing an `ArgumentException` when the assertion fails. It complements the preconditions in the third-party `CommunityToolkit.Diagnostics.Guard` class, which handles null/whitespace checks but offers no collection-containment checks.

### Role in the Codebase

It sits under `Common/Diagnostics`, the project's shared guard/argument-checking helpers, and is consumed from the Mvvm layer. It has no other dependencies. Removing it would break the child-ViewModel bookkeeping in `ViewModelBase` (below), which relies on it to detect double-registration and unregistration of unregistered view models.

### Relationships

The only in-repo caller is `Mvvm/ViewModelBase.cs`:

- `ViewModelBase.RegisterChildViewModel` calls `ExtendedGuard.NotContains(_childViewModels, viewModel, nameof(viewModel))` before adding a child, so a view model cannot be registered twice.
- `ViewModelBase.UnregisterChildViewModel` calls `ExtendedGuard.Contains(...)` before removing a child, so you cannot unregister a view model that was never registered.

Both call sites combine it with `Guard.IsNotNull(viewModel)` from CommunityToolkit.Diagnostics.

### Behavior & Details

Two methods, both on the static class:

- `Contains<T>(ICollection<T> collection, T item, string name)` — throws `ArgumentException` if `collection` does not contain `item`.
- `NotContains<T>(ICollection<T> collection, T item, string name)` — throws `ArgumentException` if it does.

The `name` parameter is meant to identify the offending item in the error message for the caller's convenience. Both containment checks use `ICollection<T>.Contains`, so equality is whatever the item type's `Equals` does (for view models, reference equality by default).

### Rough edges

The thrown messages hard-code a literal `'{name}'` instead of interpolating the `name` argument (e.g. `"... does not contain '{name}'"`), so the parameter is accepted but never actually appears in the exception message. As written, the message gives no hint about which collection or item failed.