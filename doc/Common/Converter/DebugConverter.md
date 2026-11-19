# `Common/Converter/DebugConverter.cs`

### Purpose

`DebugConverter` is a WPF value converter that exists purely to help developers inspect data-binding traffic at runtime. When a binding is routed through it, every `Convert` call writes the value, target type, and parameter to the debug output via `Debug.WriteLine` and then calls `Debugger.Break()` so execution pauses while the values are visible. The value itself passes through unchanged — the converter alters no data, it only observes and interrupts.

### Role in the Codebase

It is the only file in the `Common/Converter` directory, which exists to hold diagnostic tools for WPF data binding. Because it lives in the `Common` assembly (not the `WpfApp1` app assembly), any view in the application can declare it as an XAML resource via the `clr-namespace:Common.Converter;assembly=Common` mapping.

Removing it would break nothing in the current runtime behaviour: `WpfApp1/View/UserView.xaml` declares it as the resource `MyDebugConverter`, but the only binding that uses it (`{Binding LastName, Converter={StaticResource MyDebugConverter}}`) is commented out. The class is effectively dormant until someone re-enables or adds such a binding.

### Relationships

- **Implements `System.Windows.Data.IValueConverter`** — a WPF (.NET framework/WPF) interface, not defined in this repository. The two members `Convert` and `ConvertBack` are the interface's required methods.
- **Consumed in `WpfApp1/View/UserView.xaml`** — declared as `<converter:DebugConverter x:Key="MyDebugConverter" />` in the window resources, but referenced only by a commented-out `TextBox`. No active code path currently calls it.

No other project files reference the class.

### Behavior & Details

- **`Convert(object? value, Type targetType, object? parameter, CultureInfo culture)`** — logs `"Convert: Value '{0}', TargetType '{1}', Parameter '{2}'"` to the debug trace, then immediately calls `Debugger.Break()`. It returns `value` unmodified. The break means that while a debugger is attached, the app halts on every binding update that flows through this converter — by design, since the point is manual inspection. When no debugger is attached, `Debugger.Break()` is a no-op and only the `Debug.WriteLine` output is produced.
- **`ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)`** — a pass-through that returns `value` unchanged, with no logging and no break. A converter bound in `TwoWay` or `OneWayToSource` mode would therefore only interrupt on the source-to-target direction.

Both members are safe to call repeatedly; there is no state, caching, or side effect other than the debug output and the debugger break.

### Rough edges

- The `Debugger.Break()` in `Convert` means re-enabling the commented-out `UserView` binding in a release/debug build with a debugger attached will halt the app on every keystroke of that `TextBox` (it uses `UpdateSourceTrigger=PropertyChanged`-style live bindings elsewhere in the same form), making it usable only for a one-off inspection, not extended debugging.