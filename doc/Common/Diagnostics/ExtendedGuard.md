# Common/Diagnostics/ExtendedGuard.cs

### Purpose

`ExtendedGuard` is a small static helper for argument preconditions: it asserts that a collection does or does not contain a given item, and throws `ArgumentException` when the assertion fails. It fills a gap in the third-party guard library already used by this codebase (CommutityToolkit.Mvvm's `Guard`, which validates nulls and such but not collection membership). The two methods exist so that add/remove operations can fail loudly instead of silently producing duplicates or no-ops.

### Role in the Codebase

It is the sole file in `Common/Diagnostics/` and a leaf utility in the `Common` project: it has no dependencies on other project code, only on `System.Collections.Generic`. Nothing depends on it at compile time except its one current consumer, `Mvvm/ViewModelBase.cs`; the class would be dead code if that call site were removed, but nothing else in `Common` references it.

### Relationships

- `Mvvm/ViewModelBase.cs` is the only caller in the repository:
  - `ViewModelBase.RegisterChildViewModel` calls `ExtendedGuard.NotContains(_childViewModels, viewModel, nameof(viewModel))` before adding a child view model, so registering the same child twice throws.
  - `ViewModelBase.UnregisterChildViewModel` calls `ExtendedGuard.Contains(_childViewModels, viewModel, nameof(viewModel))` before removing one, so unregistering a view model that was never registered throws.

Note that `ViewModelBase` also uses the standard `Guard` class (from the MVVM toolkit) for null checks; `ExtendedGuard` supplements it rather than replacing it.

### Behavior & Details

Two generic methods, both taking `ICollection<T> collection`, the item to test, and a `string name` intended to identify the argument in error messages:

- `Contains<T>(collection, item, name)` — throws `ArgumentException` if the item is **not** in the collection.
- `NotContains<T>(collection, item, name)` — throws `ArgumentException` if the item **is** in the collection.

Membership is delegated to `ICollection<T>.Contains`, so behaviour matches the collection's own equality semantics (reference equality for `ViewModelBase` instances, unless a type overrides `Equals`).

Rough edge: the `name` parameter is never used. Both exception messages hard-code the literal text `"'{name}'"` (e.g. `"Collection already contains '{name}'"`), so the thrown message is identical regardless of what `ViewModelBase` passes. The placeholder was likely meant to be interpolated as `$"..." {name} ..."` — as written, the message does not identify which argument failed, and the parameter is dead weight.