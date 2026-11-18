# `Common/Converter/DebugConverter.cs`

### Purpose

`DebugConverter` is a WPF value converter that exists purely for debugging data bindings. When a binding is routed through it, every `Convert` call writes the value, target type, and parameter to the debug output stream and then calls `Debugger.Break()`, dropping the developer into the debugger mid-binding. The value itself passes through unchanged, so the UI keeps working while you inspect state.

### Role in the Codebase

It is the only component in the `Common/Converter` directory, which exists to hold diagnostic tools for WPF data binding. The class implements `System.Windows.Data.IValueConverter` (a .NET WPF interface, not defined in this repo), which is why it can be instantiated directly in XAML markup. Nothing in the application depends on it at runtime in a meaningful way — removing it would only break XAML that references the `MyDebugConverter` resource key, which today is commented out.

### Relationships

- **`WpfApp1/View/UserView.xaml`** declares it as a resource — `<converter:DebugConverter x:Key="MyDebugConverter" />` via the `clr-namespace:Common.Converter;assembly=Common` mapping — and contains one commented-out binding that uses it: a `TextBox` bound to `LastName` with `Converter={StaticResource MyDebugConverter}`.
- Callers are WPF views that reference it as a static resource in a binding's `Converter` property. It lives in the `Common` assembly, so it's shared infrastructure rather than app-specific code.

### Behavior & Details

- **`Convert`** — logs via `Debug.WriteLine` the incoming `value`, `targetType`, and `parameter`, then calls `Debugger.Break()`, then returns `value` unchanged. So the converter is a pure passthrough with a breakpoint as a side effect.
- **`ConvertBack`** — returns `value` unchanged, with no logging or break. Writes from the UI back to the data source are silent; only source-to-target conversions interrupt.

### Rough edges

- `Debugger.Break()` in `Convert` means a single binding that updates on every keystroke (e.g. `UpdateSourceTrigger=PropertyChanged` in the reverse direction is fine, but any two-way or frequent source refresh) will halt the process repeatedly while attached. It is clearly meant to be attached momentarily and removed, not left in a binding.
- The `ConvertBack` path is not instrumented, so if you are debugging why a property write isn't taking effect, this converter will not help.