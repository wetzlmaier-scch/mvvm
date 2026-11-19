### Purpose

The `Common/Converter` directory contains diagnostic utilities for WPF data binding. Its sole component, `DebugConverter`, allows developers to inspect binding values at runtime by logging details and pausing execution. It lives in the `Common` assembly to keep diagnostic logic separate from application-specific code, making it reusable across views.

### Software Components

This directory contains a single source file:

*   `DebugConverter.cs`: Implements `System.Windows.Data.IValueConverter`. It acts as a pass-through converter that does not modify data. Instead, its `Convert` method logs the value, target type, and parameter to the debug output and triggers a `Debugger.Break()`. The `ConvertBack` method is a simple pass-through with no side effects.

### Relationships

`DebugConverter` is a leaf component with no internal dependencies. It depends on the WPF framework interface `IValueConverter`.

Externally, `WpfApp1/View/UserView.xaml` declares `DebugConverter` as a resource. However, the only binding referencing it is currently commented out, meaning the class is dormant in the active runtime. No other files in the repository reference this converter.

```mermaid
flowchart TD
    subgraph "Common/Converter"
        DC["DebugConverter.cs"]
    end

    subgraph "External Dependencies"
        IVC["IValueConverter (WPF)"]
        UV["UserView.xaml"]
    end

    DC -.->|implements| IVC
    UV -.->|references (commented out)| DC
```

## Contents
- [DebugConverter.cs](Common/Converter/DebugConverter.md)