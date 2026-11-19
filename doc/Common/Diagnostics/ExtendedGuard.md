### Purpose

`Common/Diagnostics/ExtendedGuard.cs` defines `ExtendedGuard`, a small static helper for argument preconditions on collection membership. It exists to give callers a one-liner for the two assertions the standard `Guard` class (from the MVVM toolkit) doesn't cover: "this item must be in this collection" and "this item must not be in this collection". If the assertion fails, the method throws `ArgumentException`.

### Role in the Codebase

`ExtendedGuard` is the only file in `Common/Diagnostics/`. It is a leaf utility: it references nothing inside the `Common` project, and it is referenced by exactly one place, `Mvvm/ViewModelBase.cs`. Removing it would force `ViewModelBase.RegisterChildViewModel` and `UnregisterChildViewModel` to inline their duplicate- and missing-child checks.

### Relationships

- No imports beyond `System`/`System.Collections` (implicit usings); no dependencies on other `Common` types.
- The sole consumer is `Mvvm/ViewModelBase.cs`:
  - `RegisterChildViewModel` calls `ExtendedGuard.NotContains(_childViewModels, viewModel, nameof(viewModel))` before adding, so registering the same child view model twice throws.
  - `UnregisterChildViewModel` calls `ExtendedGuard.Contains(_childViewModels, viewModel, nameof(viewModel))` before removing, so unregistering a view model that was never registered throws.
- `ViewModelBase` also uses the toolkit's standard `Guard` for null checks; `ExtendedGuard` supplements rather than replaces it.

### Behavior & Details

Two generic methods, both throwing `ArgumentException` (with `collection` as the parameter name) on failure:

- `Contains<T>(ICollection<T> collection, T item, string name)` — throws if `collection.Contains(item)` is `false`. Used as a precondition that an item exists before removal.
- `NotContains<T>(ICollection<T> collection, T item, string name)` — throws if the item is already present. Used as a precondition against duplicates before addition.

Both take a `name` parameter intended to name the argument in the error message, but neither interpolates it into the exception text (the messages are the literal strings `"Collection does not contain '{name}'"` and `"Collection already contains '{name}'"`).

### Rough edges

The `name` parameter is accepted but never used in either exception message, so the thrown `ArgumentException`s do not identify which argument failed. The literal `{name}` placeholder is misleading, implying interpolation that does not happen.