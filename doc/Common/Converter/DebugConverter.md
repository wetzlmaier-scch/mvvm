### Purpose

`DebugConverter` (`Common/Converter/DebugConverter.cs`) is a WPF `IValueConverter` used for debugging data bindings. When you attach it to a binding, every time the binding asks for a converted value, it logs the values in play and drops you into the debugger — so you can watch what's flowing through a binding you suspect is broken.

### Role in the Codebase

It is the only converter in the `Common/Converter` folder. It exists purely as a diagnostic aid: you temporarily point a binding at it (via a resource key), observe the binding traffic in the debugger, then remove it. It has no production behavior — both directions simply pass the value through unchanged.

### Relationships

- It implements `System.Windows.Data.IValueConverter` (a WPF/WindowsBase type), so it can be instantiated as a XAML resource and referenced from bindings.
- It is consumed from WPF views: `WpfApp1/View/UserView.xaml` declares it as `<converter:DebugConverter x:Key="MyDebugConverter" />` and uses it in a binding, though the current usage on a `TextBox` is commented out. Callers are therefore XAML files that reference it by a static resource key.
- It uses `System.Diagnostics.Debug` for the log output.

### Behavior & Details

- `Convert(value, targetType, parameter, culture)`: writes a line to the debug output via `Debug.WriteLine` showing the bound value, target type, and parameter, then calls `Debugger.Break()`, then returns `value` unchanged. This is the side effect that makes the converter useful: if you don't have a debugger attached, the break is a no-op and the converter is just a pass-through.
- `ConvertBack(value, targetType, parameter, culture)`: returns the value as-is with no logging or breaking. So the converter only triggers when the binding pushes a value *to* the view, not when it reads back from the view.

Because the class is `public` with a parameterless constructor, it can be instantiated in XAML without any special registration.