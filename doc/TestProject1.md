### Purpose

`TestProject1` is the unit test project for the user management view-models. It isolates the presentation logic of `AddUserViewModel` and `ShowUsersViewModel` from the rest of the application, allowing you to verify validation rules, error handling, and data population without needing a live database or UI.

The directory serves as a single, focused test suite. It contains only what is necessary to test the view-model layer: a configuration file for the test runner and the test class itself. Its primary goal is to provide fast feedback on changes to the user management UI logic.

### Software Components

The directory contains one executable component:

*   **`UserViewModelTest`**: Defined in [UserViewModelTest.cs](TestProject1/UserViewModelTest.md), this class holds the unit tests for `AddUserViewModel` and `ShowUsersViewModel`. It uses Moq to mock `IUserService` and `IMessageService`, ensuring the tests run in isolation.

The directory also contains one configuration file:

*   **`MSTestSettings.cs`**: This file configures the MSTest v2 runner for the entire assembly. It is not a software component in the traditional sense but defines how the tests are executed.

### Relationships

The relationship within this directory is straightforward: `MSTestSettings.cs` defines the execution environment, and `UserViewModelTest.cs` contains the logic that runs within it.

`MSTestSettings.cs` enables method-level parallelization for the whole assembly. However, `UserViewModelTest.cs` explicitly opts out of this via the `[DoNotParallelize]` attribute. This is necessary because the tests rely on `WeakReferenceMessenger.Default`, a global static instance. If tests ran in parallel, they would interfere with each other's messenger subscriptions. Therefore, the parallelization setting currently has no practical effect on this project, as `UserViewModelTest` is the only test class.

```mermaid
flowchart TD
    subgraph "TestProject1"
        Settings[MSTestSettings.cs]
        Tests[UserViewModelTest.cs]
    end
    
    Settings -->|Enables assembly-level parallelization| Tests
    Tests -->|Opt-out via [DoNotParallelize]| Settings
```

### Rough edges

The assembly-level parallelization in `MSTestSettings.cs` is currently ineffective because the only test class, `UserViewModelTest`, opts out. This creates a misleading configuration: new developers might assume tests run in parallel and fail to protect against shared static state when adding new test classes. Additionally, `UserViewModelTest`'s `Teardown` does not clean up messenger subscriptions for the root `UserViewModel`, leading to accumulated dead registrations in the global messenger during test runs.

## Contents
- [MSTestSettings.cs](TestProject1/MSTestSettings.md)
- [UserViewModelTest.cs](TestProject1/UserViewModelTest.md)