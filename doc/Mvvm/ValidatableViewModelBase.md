### Purpose

`Mvvm/ValidatableViewModelBase.cs` defines `ValidatableViewModelBase<T>`, an abstract base class for ViewModels whose properties are validated with [FluentValidation](https://fluentvalidation.net/) and whose error state is surfaced to the UI through the standard WPF/WinForms `INotifyDataErrorInfo` contract. It exists so that a ViewModel doesn't have to hand-roll error bookkeeping: the class holds the current validation errors, fires `ErrorsChanged` when they change, and lets derived ViewModels call `validator.Validate(...)`-backed helpers directly.

### Role in the Codebase

It lives in the `Mvvm` project, which holds the framework-level ViewModel plumbing (`ViewModelBase`, this class) shared by the UI front-ends in `Mvvm.Wpf`, `Mvvm.Winforms`, and `Mvvm.Console`. A concrete ViewModel that wants property validation inherits from it and passes in a FluentValidation validator via the primary constructor — for example `AddUserViewModel : ValidatableViewModelBase<IUser>(new UserValidator())` in `UserManagement/ViewModel/AddUserViewModel.cs`.

The comment at the top of the class notes that this is an example base class, and that for projects that already use CommunityToolkit.Mvvm's own validation, `ObservableValidator` can be used instead.

### Relationships

- **Base class:** `ViewModelBase` (defined in `Mvvm/ViewModelBase.cs`), which itself derives from CommunityToolkit.Mvvm's `ObservableObject` and implements `IViewModel`. Inherited from `ViewModelBase`: `INotifyPropertyChanged` plumbing, design-mode detection, and child-ViewModel registration. Inherited from `ObservableObject`: the `ObservableObject` API. This class adds the `INotifyDataErrorInfo` interface, which is not implemented by `ViewModelBase`.
- **External dependency:** `FluentValidation.IValidator<T>` and its `Validate`/`ValidationResult` types.
- **Callers:** any ViewModel that wants `INotifyDataErrorInfo`-driven validation. In this repo, `AddUserViewModel` is the current example.

### Behavior & Details

The class is generic over the object type `T` being validated and receives a single `IValidator<T>` through its primary constructor. Derived classes pass in their concrete validator, e.g. `new UserValidator()` for `IUser`.

State is a single dictionary: `Dictionary<string, List<string>> _errors`, keyed by property name, holding the list of error messages for that property.

`INotifyDataErrorInfo` is implemented as follows:

- `GetErrors(propertyName)` — returns the list of messages for that property, or an empty enumerable. A `null` property name (the contract's "give me all errors" call) also returns an empty enumerable; see Rough edges.
- `HasErrors` — true as long as `_errors` is non-empty, i.e. as long as any property currently has at least one error.
- `ErrorsChanged` — the standard event; raise it after mutating `_errors` so the UI re-queries `GetErrors`/`HasErrors`.
- `OnErrorsChanged(DataErrorsChangedEventArgs)` — protected helper that wraps the event invocation so derived classes don't have to null-check.

The two validation entry points are:

- `ValidateAllProperties(T obj)` — clears `_errors` and runs `validator.Validate(obj)` over every rule. For each failed property, it appends the message into `_errors[propertyName]` and raises `ErrorsChanged` for that property. Because it starts by clearing, it's the right helper to call before a "save" action: it gives a fresh, whole-object result.
- `ValidateProperty(T obj, string propertyName)` — runs `validator.Validate(obj, options => options.IncludeProperties(propertyName))` so only rules for that property execute. If valid, the property's entry is removed from `_errors`; if not, the entry is replaced with the fresh list of messages. `ErrorsChanged` is raised for the property either way.

A typical derived ViewModel calls `ValidateProperty(this, nameof(Prop))` from an `OnPropChanged` hook and `ValidateAllProperties(this)` before a save, then checks `HasErrors` to decide whether to proceed — exactly what `AddUserViewModel` does in `SaveUserAsync`.

### Rough edges

- `GetErrors(null)` returns an empty enumerable. The `INotifyDataErrorInfo` contract documents the `null` argument as "return errors for all properties"; the `HasErrors` property exists as the substitute, and WPF's data-error template binding uses `GetErrors` per property, so this is workable, but consumers who call `GetErrors(null)` expecting every error will come back empty-handed.
- `ValidateProperty` only re-evaluates rules attached to the one property passed in. A FluentValidation rule that compares two properties (e.g. "confirm password must match password") won't be picked up here even when the other property changes — only `ValidateAllProperties` will exercise it. Callers have to know which rules cross properties and call `ValidateAllProperties` for those cases.