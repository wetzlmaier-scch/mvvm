### Purpose

`UserForm.resx` (`WinFormsApp1/View/UserForm.resx`) is the resource file paired with the `UserForm` designer-generated code. It is an `.resx` XML file, not a source file — it holds the declarative, designer-managed state for the form's controls, and the designer tooling generates and rewrites it; it should not be edited by hand.

### Role in the codebase

It sits alongside `UserForm.cs` and `UserForm.Designer.cs` in `WinFormsApp1/View`. The `.Designer.cs` file (the "partial" half of `UserForm`) calls `InitializeComponent()`, which loads this resource at design and compile time to reconstruct control metadata such as `ToolTips`, `ErrorProvider` layout, and other designer-serialised properties. Removing it would break `InitializeComponent()` regeneration and the designer's ability to round-trip the form.

### Relationships

- Consumed by `UserForm.Designer.cs`, which calls `InitializeComponent()`; that method reads this file's entries (e.g. the `errorProvider.TrayLocation` metadata) to reconstruct the form's component state.
- The single `<data>`-less `<metadata name="errorProvider.TrayLocation">` entry describes where the form's `errorProvider` component sits in the designer's component tray (position `17, 17`). `UserForm.cs` uses that `errorProvider` at runtime to show validation errors from `AddUserViewModel` on `txtFirstName` and `txtLastName`.

### Behavior & Details

- The bulk of the file is the standard Microsoft ResX header: schema, version (`2.0`), and reader/writer class references (`System.Resources.ResXResourceReader` / `ResXResourceWriter` from `System.Windows.Forms` 4.0). This is boilerplate present in every `.resx` file.
- No `<data>` entries are defined, so this form has no string, image, or object resources — only designer metadata.
- The sole non-boilerplate entry is:
  ```xml
  <metadata name="errorProvider.TrayLocation" type="System.Drawing.Point, System.Drawing, ...">
    <value>17, 17</value>
  </metadata>
  ```
  This records the `errorProvider` component's position in the designer's component tray, so the form opens with the component in a known place the next time the designer loads it.