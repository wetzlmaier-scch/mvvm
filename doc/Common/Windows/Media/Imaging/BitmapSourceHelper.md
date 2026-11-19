### Purpose

`Common/Windows/Media/Imaging/BitmapSourceHelper.cs`

`BitmapSourceHelper` is a static utility class that converts images between two different .NET imaging systems — GDI+ (`System.Drawing.Image` / `Bitmap`) and WPF's `BitmapSource` — and loads images from a file path. It exists because GDI+ and WPF each have their own bitmap types, and you cannot assign a `System.Drawing.Bitmap` directly to a WPF property (such as `Window.Icon`) without an intermediate conversion step.

The class is stateless: every method is `static` and holds no fields. It is a leaf component in the `Common` project with no dependencies on other custom code.

### Role in the Codebase

The class sits in `Common/Windows/Media/Imaging/` as the sole file in that directory. Its job is to be the single place that knows how to cross the GDI+/WPF imaging boundary, so that callers in the application don't have to deal with `CreateBitmapSourceFromHBitmap`, encoder frames, or stream bookkeeping themselves.

Its only caller in this repository is `Mvvm/MainViewModelBase`, which calls `BitmapSourceHelper.LoadBitmap(icon.ToBitmap())` in its constructor to convert the application icon (loaded from the assembly's manifest resources via `AssemblyHelper`) into a `BitmapSource` suitable for `Window.Icon`. Removing the class would break that icon-loading path and any future code that needs to convert GDI+ images for WPF display.

### Relationships

**No internal dependencies.** The class does not reference any other custom code in the repository. All its dependencies are framework types:

- `System.Drawing` — `Image`, `Bitmap`, `GetHbitmap()`
- `System.Windows.Interop` — `Imaging.CreateBitmapSourceFromHBitmap`
- `System.Windows.Media.Imaging` — `BitmapSource`, `BitmapImage`, `BitmapEncoder`, `BmpBitmapEncoder`, `BitmapFrame`
- `System.IO` — `MemoryStream`, `File.OpenRead`

**Caller:** `Mvvm/MainViewModelBase` (specifically its constructor) calls `LoadBitmap(Bitmap)` to produce the window icon.

### Behavior & Details

**`LoadImage(Image? source)`**
Null-passthrough: returns `null` if the argument is `null`. Otherwise wraps the `Image` in a `new Bitmap(source)` and delegates to `LoadBitmap(Bitmap)`. The new `Bitmap` is the managed handle around the image data; it is not explicitly disposed here, so the caller retains ownership.

**`LoadBitmap(Bitmap? source)`**
The core GDI+ → WPF conversion. Uses `source.GetHbitmap()` to obtain a native GDI handle, then calls `System.Windows.Interop.Imaging.CreateBitmapSourceFromHBitmap` to produce a WPF `BitmapSource` that references that handle. Note: the HBITMAP is a native resource. `CreateBitmapSourceFromHBitmap` does not take ownership of it in the sense of freeing it when the `BitmapSource` is garbage-collected, so the GDI handle leaks for the lifetime of the process. For small, infrequent calls (like loading one application icon at startup) this is bounded and harmless, but repeated calls in a loop would accumulate native GDI objects.

**`SaveBitmap(BitmapSource? source)`**
The reverse direction: WPF → GDI+. Encodes the `BitmapSource` into a BMP byte sequence via `BmpBitmapEncoder` in a `MemoryStream`, then constructs a `System.Drawing.Bitmap` from that stream. The extra `new Bitmap(bitmap)` copy is a shallow clone that detaches the bitmap from the stream, since the `Bitmap(Stream)` constructor keeps the stream alive. The result is a self-contained `Bitmap`.

**`LoadBitmap(string filename)`**
Convenience overload for loading a WPF image directly from disk. Opens the file, delegates to the stream overload, and returns the resulting `BitmapImage`. The `File.OpenRead` stream is disposed by the `using` statement after the `BitmapImage` is constructed. Because `CacheOption.OnLoad` is set (see below), the image data is fully read into memory before the stream closes, so this is safe.

**`LoadBitmap(Stream stream)`**
The underlying WPF image decoder. Creates a `BitmapImage`, calls `BeginInit`/`EndInit`, and sets `CacheOption = BitmapCacheOption.OnLoad` so the pixel data is decoded eagerly from the stream rather than lazily on first render. The caller is responsible for keeping the stream open only long enough for `EndInit`; with `OnLoad` the data is fully consumed by that point.

### Rough Edges

- `LoadBitmap(Bitmap)` calls `GetHbitmap()` without a corresponding `DeleteObject`. Each call leaks one native GDI handle until process exit. This is acceptable for the single icon-load at startup but would be a problem if the method were used repeatedly.