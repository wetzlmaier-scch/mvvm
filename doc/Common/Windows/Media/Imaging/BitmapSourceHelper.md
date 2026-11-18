# `Common/Windows/Media/Imaging/BitmapSourceHelper.cs`

### Purpose

`BitmapSourceHelper` is a small static utility class that converts images between GDI+ (`System.Drawing.Image` / `Bitmap`) and WPF's `BitmapSource` type system, and loads images from file streams. It exists because WPF UI elements bind to `BitmapSource`, while Win32/GDI+ APIs (window icon extraction, file dialogs, interop with unmanaged code) produce `Bitmap` objects — this class is the bridge between the two.

### Role in the Codebase

This file is the only source file in the `Common/Windows/Media/Imaging` directory. It is a leaf-level helper with no dependencies on other custom code in the repository — it relies solely on BCL and WPF imaging APIs. The class is stateless and thread-safe in the trivial sense (no mutable static state).

The only caller in the repository is `Mvvm/MainViewModelBase`, which uses `LoadBitmap(Bitmap)` in its constructor to convert the application's executable icon (obtained via `AssemblyHelper.LoadApplicationIcon()`) into a `BitmapSource` and assign it to the window's `Icon` property. If this class were removed, that icon-loading path would need to be rewritten inline.

### Relationships

- **`System.Drawing`** — provides `Image` and `Bitmap` (GDI+ types).
- **`System.Windows.Interop.Imaging`** — provides `CreateBitmapSourceFromHBitmap`, the API that converts a GDI handle into a WPF `BitmapSource`.
- **`System.Windows.Media.Imaging`** — provides `BitmapSource`, `BitmapImage`, `BitmapFrame`, and `BmpBitmapEncoder` (WPF imaging types).
- **Caller**: `Mvvm/MainViewModelBase` (via `using Common.Windows.Media.Imaging`).

### Behavior & Details

The class exposes five methods, all `public static`:

**`LoadImage(Image? source)`**
Null-pass-through. If `source` is null, returns null. Otherwise wraps the `Image` in a new `Bitmap` (GDI+ requires this to call `GetHbitmap`) and delegates to `LoadBitmap(Bitmap)`.

**`LoadBitmap(Bitmap? source)`**
Null-pass-through. If non-null, calls `source.GetHbitmap()` to obtain a GDI handle, then passes it to `Imaging.CreateBitmapSourceFromHBitmap` to produce a WPF `BitmapSource`. The `Int32Rect.Empty` argument means "use the full bitmap," and `FromEmptyOptions()` means "default pixel format and color context."

**`SaveBitmap(BitmapSource? source)`**
The reverse direction. Null-pass-through. Encodes the `BitmapSource` as a BMP into a `MemoryStream` via `BmpBitmapEncoder`, then reads that stream back into a GDI+ `Bitmap`. Returns a new `Bitmap` wrapping the decoded one.

**`LoadBitmap(string filename)`**
Opens the file for reading and delegates to the `Stream` overload. Note: this overload returns a non-nullable `BitmapSource` and will throw (`FileNotFoundException`, etc.) if the file is missing or unreadable — it does not do null-checks the way the `Bitmap` overloads do.

**`LoadBitmap(Stream stream)`**
Creates a `BitmapImage`, sets `CacheOption` to `OnLoad` (so the entire image is decoded into memory immediately and the stream can be closed), assigns the stream, and calls `BeginInit`/`EndInit` to trigger the decode. Returns the loaded `BitmapImage` (which is a `BitmapSource`).

### Rough Edges

`LoadBitmap(Bitmap)` calls `GetHbitmap()` but never calls `DeleteObject` on the resulting HBITMAP. Each call therefore leaks one native GDI handle for the lifetime of the process. In the current codebase this path is hit once at startup for the window icon, so the leak is bounded — but any future caller that invokes this method repeatedly (e.g. in a loop or per user action) would accumulate handles until the process runs out of GDI objects.

`SaveBitmap` allocates a full in-memory BMP round-trip and returns `new Bitmap(bitmap)` — an unnecessary copy of the `Bitmap` just constructed from the stream. The extra allocation is harmless but the returned `Bitmap` is a GDI+ object the caller must dispose.

`LoadBitmap(string filename)` declares a non-nullable return type, so a missing file surfaces as an exception rather than a null — inconsistent with the other `LoadBitmap` overloads that return null on null input.