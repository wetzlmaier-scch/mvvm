# `UserManagement/Design/DesignAddUserViewModel.cs`

### Purpose
`DesignAddUserViewModel` exists so the "add user" form has plausible content to render in the WPF XAML designer. It subclasses the runtime `AddUserViewModel` and pre-fills `FirstName` with `"Sample"` and `LastName` with `"User"` in its constructor. Nothing about it participates in the running application.

### Role in the Codebase
It is one of three design-time stubs in `UserManagement/Design/` (alongside `DesignShowUsersViewModel` and `DesignUserViewModel`), each mirroring a real view model from `UserManagement/ViewModel/` with demo data. The designer instantiates the view through a `d:DesignInstance` reference to `DesignUserViewModel`, which in turn constructs this class:

```csharp
AddUserViewModel = new DesignAddUserViewModel();
```

Because the only thing it adds is two demo strings, removing it would only lose sample content in the designer — the application's runtime behaviour, driven by the real `AddUserViewModel`, is unaffected.

### Relationships
- **Base class:** `AddUserViewModel` (`UserManagement/ViewModel/AddUserViewModel.cs`), a `ValidatableViewModelBase<IUser>` whose `[ObservableProperty]` source generators produce the `FirstName` and `LastName` properties this class sets. It inherits all of that view model's behaviour — validation hooks and the `SaveUserCommand` — but none of that logic runs in the designer, where no MEF imports (`UserService`, `LoggingService`, `MessageService`) are provided.
- **Sole caller:** `DesignUserViewModel` (`UserManagement/Design/DesignUserViewModel.cs`), the design-time root that builds the whole designer view hierarchy.

### Behavior & Details
The entire class is its parameterless constructor, which assigns the two demo names to the inherited observable properties. Setting them also fires the inherited `OnFirstNameChanged`/`OnLastNameChanged` hooks, which call `ValidateProperty` — harmless at design time, since validation results are never acted on there. The values are display-only placeholders with no runtime meaning.

Note the class is declared `partial`, but no other part for it exists in the repository; the modifier has no effect.