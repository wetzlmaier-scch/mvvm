# `Mvvm/ValidatableViewModelBase.cs`

`Mvvm/ValidatableViewModelBase.cs` defines `ValidatableViewModelBase<T>`, an abstract base class for ViewModels that need FluentValidation-backed validation with errors surfaced through WPF/WinForms data-binding validation.

### Purpose

`ViewModelBase` provides observable state but no validation story. This class adds that: it implements `INotifyDataErrorInfo` so the UI can display per-field error text and disable actions, while the actual rules live in an external `FluentValidation.IValidator<T>` rather than in the ViewModel. The contract is: the concrete ViewModel is (or exposes) the model object `T` that the validator knows how to validate, and the base class keeps the error state and raises change notifications. The comment in the file notes that if you don't need FluentValidation, `CommunityToolkit.Mvvm`'s `ObservableValidator` is the alternative.

### Role in the Codebase

It is one of the base-class family in the `Mvvm` project (alongside `ViewModelBase`, `DisposableViewModelBase`, `MainViewModelBase`) — framework-level plumbing consumed by the UI front-ends. It is a mixin of the "validation" concern: a concrete ViewModel picks it in when it needs validation and passes its validator in the primary constructor. The one current consumer is `AddUserViewModel` in `UserManagement/ViewModel/AddUserViewModel.cs`, declared as `ValidatableViewModelBase<IUser>(new UserValidator())` — it validates itself (`ValidateProperty(this, nameof(FirstName))`) because it implements the `IUser` properties the `UserValidator` rules read. Removing this class would break `AddUserViewModel`'s error reporting; its `HasErrors` check gates `SaveUserAsync`.

### Relationships

- Extends `ViewModelBase` (same folder), which extends CommunityToolkit.Mvvm's `ObservableObject` and implements `IViewModel` — the inheritance brings `INotifyPropertyChanged` state-change plumbing for free.
- Implements `System.ComponentModel.INotifyDataErrorInfo` (`GetErrors`, `HasErrors`, `ErrorsChanged`), the contract WPF/WinForms `DataTemplate` validation binds to.
- Depends on `FluentValidation`'s `IValidator<T>` (external library); the concrete validator is supplied by the caller, e.g. `Dto/Validation/UserValidator.cs`.
- Callers: concrete ViewModels like `AddUserViewModel` invoke `ValidateProperty` from generated `On{Prop}Changed` hooks and `ValidateAllProperties` before committing (its `SaveUserAsync` calls it and checks `HasErrors`).

### Behavior & Details

- The constructor takes an `IValidator<T> validator` (primary constructor) and holds it for the lifetime of the ViewModel.
- Error state is `_errors`, a `Dictionary<string, List<string>>` mapping property name to a list of error messages.

`ValidateProperty(T obj, string propertyName)` — single-field validation, the path used for per-keystroke feedback. It runs the validator with `IncludeProperties(propertyName)`, so only that property's rules execute. If valid, the entry is removed from `_errors`; if invalid, the entry is replaced wholesale with the fresh messages. Either way it raises `ErrorsChanged` for just that property, so the UI re-queries `GetErrors` for one field.

`ValidateAllProperties(T obj)` — full validation. It clears `_errors` first, then walks every rule failure, appending messages per property and raising `ErrorsChanged` once per failing property. Note the ordering: because the dictionary is cleared before any event fires, a consumer polling `HasErrors` mid-method sees transient emptiness; in practice the caller (`AddUserViewModel.SaveUserAsync`) reads `HasErrors` only after the method returns.

`GetErrors` / `HasErrors` / `ErrorsChanged` — the `INotifyDataErrorInfo` surface. `GetErrors(null)` (the "whole object" query) always returns empty: errors are only ever tracked per property, so a UI asking for aggregate errors via `null` sees none even when fields are invalid. `HasErrors` is simply `_errors.Count != 0`, and it is the flag the ViewModel's save command actually uses.

`OnErrorsChanged` is the protected helper the base itself calls; a subclass could also use it to raise events for other reasons, but no current subclass does.

Rough edges

- `GetErrors` returning empty for a `null` `propertyName` means `INotifyDataErrorInfo`'s whole-object query is a no-op; correctness of `HasErrors` depends on the caller having invoked one of the two validate methods (both do, in `AddUserViewModel`), and there is no automatic re-validation on `PropertyChanged`.
- `ValidateAllProperties` fires `ErrorsChanged` per failing property after clearing the dictionary, so subscribers see a brief window where no errors exist.