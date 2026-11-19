# Mvvm/ValidatableViewModelBase.cs

### Purpose

`ValidatableViewModelBase<T>` is an abstract base class that gives a ViewModel FluentValidation-backed validation with errors surfaced through the standard `INotifyDataErrorInfo` interface. It exists so that ViewModels validating form data against business rules don't each have to hand-roll an error dictionary and the `ErrorsChanged` plumbing; a derived ViewModel supplies an `IValidator<T>` and gets per-property and all-property validation, plus `HasErrors`, for free.

The class comment notes this is an example/pattern base class: for ViewModels that don't need FluentValidation, `CommunityToolkit.Mvvm`'s built-in `ObservableValidator` is the alternative.

### Role in the Codebase

It is a mid-level node in the ViewModel base-class hierarchy in the `Mvvm` project, alongside `ViewModelBase`, `DisposableViewModelBase`, and `MainViewModelBase`. It extends `ViewModelBase` (defined in `Mvvm/ViewModelBase.cs`, which extends `CommunityToolkit.Mvvm`'s `ObservableObject` and implements `IViewModel`) and adds the validation layer on top. It is framework plumbing, not business logic: all the actual rules live in FluentValidation validators such as `UserValidator` (`Dto/Validation/UserValidator.cs`).

If it were removed, ViewModels deriving from it (currently only `AddUserViewModel`) would lose their `INotifyDataErrorInfo` implementation and the `ValidateProperty`/`ValidateAllProperties` helpers, breaking XAML data-binding validation.

### Relationships

- **`ViewModelBase`** (`Mvvm/ViewModelBase.cs`) — the base class; provides the `ObservableObject` change-notification machinery, design-mode detection, and child-ViewModel registration. `ValidatableViewModelBase` adds nothing to those and only layers validation on top.
- **FluentValidation's `IValidator<T>`** (external library) — taken as a primary constructor parameter and held for the lifetime of the instance. All validation is delegated to it.
- **`INotifyDataErrorInfo`** (`System.ComponentModel`) — implemented here. This is the interface WPF (and WinForms) binding engines use for per-field error display; XAML bindings opt in with `ValidatesOnNotifyDataErrors=True` (as done in the user-add view).
- **`AddUserViewModel`** (`UserManagement/ViewModel/AddUserViewModel.cs`) — the known caller. It derives from `ValidatableViewModelBase<IUser>(new UserValidator())`, passes `this` to `ValidateProperty(this, nameof(FirstName))` in its `On*Changed` handlers, and to `ValidateAllProperties(this)` before saving; it also reads `HasErrors` to decide whether to block the save.

### Behavior & Details

The class holds a single piece of state: `_errors`, a `Dictionary<string, List<string>>` mapping property name to error messages.

- **`GetErrors(propertyName)`** — implements `INotifyDataErrorInfo`. Returns an empty sequence for a `null` property name (the interface's "give me all errors" query) or for a property with no errors; otherwise the list of messages for that property.
- **`HasErrors`** — true when any property currently has at least one error.
- **`OnErrorsChanged(DataErrorsChangedEventArgs)`** — protected; raises the public `ErrorsChanged` event. Callers must use this rather than invoking the event directly so the notification flow stays consistent.

**`ValidateAllProperties(T obj)`**

1. Clears `_errors` entirely.
2. Runs `validator.Validate(obj)` against the whole object.
3. For each failing rule, appends the message under `error.PropertyName` and fires `ErrorsChanged` for that property.

Because it clears first, this is a full revalidation: properties that now pass lose their errors, but note that no `ErrorsChanged` is raised for properties that *cleared* — the UI only re-evaluates the properties that still have errors. This is fine in practice because `AddUserViewModel` calls it once before saving and then checks `HasErrors` rather than relying on the bindings to update.

**`ValidateProperty(T obj, string propertyName)`**

Runs `validator.Validate(obj, options => options.IncludeProperties(propertyName))` — only the rules for that one property. If valid, removes the property from `_errors`; otherwise replaces its entry wholesale with the fresh list of messages. Either way it raises `ErrorsChanged` for that property, so the binding updates live as the user types (that is how `AddUserViewModel`'s `OnFirstNameChanged`/`OnLastNameChanged` handlers work).

### Rough edges

- `ValidateAllProperties` clears `_errors` before re-adding entries, so properties that became valid get no `ErrorsChanged` notification, and `HasErrors` briefly reflects a mixed state between the clear and the repopulation. A caller mid-validation (e.g. reading `HasErrors` from another thread) could observe an inconsistent snapshot; in the current single-threaded WPF usage this is not exercised.
- The two validate methods have asymmetric error bookkeeping: `ValidateAllProperties` appends to existing lists after a clear, `ValidateProperty` replaces. That is correct given `ValidateAllProperties` clears first, but the two styles are easy to mix up when extending the class.