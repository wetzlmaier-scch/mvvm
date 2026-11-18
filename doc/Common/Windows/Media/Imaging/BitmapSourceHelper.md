# `Common/Windows/Media/Imaging/BitmapSourceHelper.cs`

### Purpose

`BitmapSourceHelper` is a small static class that converts images between GDI+ (`System.Drawing.Image` / `Bitmap`) and WPF's `BitmapSource` type system. WPF UI elements bind to `BitmapSource`; Win32/GDI+ icon and image code produces `Bitmap`. This class is the glue in both directions, so callers don't have to know the interop plumbing (`GetHbitmap`, `Imaging.CreateBitmapSourceFromHBitmap`, `BmpBitmapEncoder`) themselves.

### Role in the Codebase

It is a shared helper under `Common/Windows/`, the WPF-side of a cross-cutting `Common` namespace. It holds no state — every method is a pure conversion — and it is the only file in its directory. Callers that need to feed a `Bitmap`-based image into WPF, or round-trip a WPF image back to GDI+, go through it instead of repeating the conversion code.

### Relationships

- **Consumed by `Mvvm/MainViewModelBase`** — its constructor loads the application icon via `AssemblyHelper.LoadApplicationIcon()`, converts it with `BitmapSourceHelper.LoadBitmap(icon.ToBitmap())`, and assigns the result to the window's `Icon` property. That is the caller that makes the class's purpose concrete: getting a non-WPF image onto a WPF window.
- **System APIs it leans on** (all from the BCL / WPF, not the repo):
  - `System.Windows.Interop.Imaging.CreateBitmapSourceFromHBitmap` — the actual GDI+ → WPF bridge used by `LoadBitmap(Bitmap)`.
  - `BmpBitmapEncoder` / `BitmapFrame` — used by `SaveBitmap` to serialize a `BitmapSource` to PNG… specifically to BMP, in a memory stream.

### Behavior & Details

Four conversion entry points, all null-tolerant (a null input yields a null result except where noted):

- **`LoadImage(Image? source)`** — takes any GDI+ `Image` and wraps it in a `new Bitmap(source)` before delegating to `LoadBitmap(Bitmap)`. Use this when you only have an `Image` (e.g. from an icon handle) and don't need the `Bitmap` itself.

- **`LoadBitmap(Bitmap? source)`** — the core GDI+ → WPF conversion. It extracts the bitmap's HBITMAP handle via `GetHbitmap()` and hands it to `CreateBitmapSourceFromHBitmap` with an empty `Int32Rect` and `FromEmptyOptions()`. One caveat: `GetHbitmap()` returns a GDI handle that is never freed here (no corresponding `DeleteObject`), so callers that convert many images through this method should be aware each call leaks one HBITMAP until process exit.

- **`SaveBitmap(BitmapSource? source)`** — the reverse direction: encodes the `BitmapSource` as a BMP in a `MemoryStream`, rebuilds a GDI+ `Bitmap` from the stream, and wraps it once more in a fresh `Bitmap`. The double wrap discards the intermediate and guarantees an owned `Bitmap` rather than one still tied to the stream. The encoder is a `BmpBitmapEncoder`, so the intermediate representation is always BMP regardless of the source's native format.

- **`LoadBitmap(string filename)` / `LoadBitmap(Stream stream)`** — load a file directly into a WPF `BitmapSource`. These go through a different path than the `Bitmap` overloads: they construct a `BitmapImage` with `BeginInit`/`EndInit` and set `CacheOption = BitmapCacheOption.OnLoad`. That flag is important — it forces the full image data to be decoded and cached in memory during `EndInit`, so the returned `BitmapImage` no longer depends on the stream (or file) staying open. Note the file-based overload opens the stream with `File.OpenRead` inside a `using` block, so the file handle is released by the time the method returns, which is only safe because of the `OnLoad` cache option.

Two small asymmetries are worth knowing: the `Image`/`Bitmap` overloads are nullable-annotated and return `BitmapSource?`, while the `string`/`Stream` overloads declare a non-nullable `BitmapSource` and will throw (`FileNotFoundException` / `ArgumentException`) on bad input instead of returning null.

### Rough edges

- `LoadBitmap(Bitmap)` never releases the HBITMAP obtained from `GetHbitmap()` — a native handle leak per call (see above).
- `SaveBitmap` always routes through BMP encoding even when the caller might want a lossless format like PNG, so quality/format of the round-tripped image is fixed regardless of the `BitmapSource` it started from.