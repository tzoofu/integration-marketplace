# ZXing QR code generation

- **category**: other
- **provider**: ZXing
- **reusable**: yes — a trivial drop-in library any repo needing client-side QR generation (e.g. an invite/share flow) could reuse the same way.
- **docs**: https://github.com/zxing/zxing

## Overview
Client-side library (no external network calls) that encodes text into a QR code bitmap — used here to turn a group invite code into a scannable QR image, entirely on-device.

## Playbook

### Prerequisites
- ZXing `core` library on the classpath (`com.google.zxing:core`) — no server, API key, or network dependency; encoding happens entirely on-device.
- Any UI layer capable of rendering a `Bitmap`/`ImageBitmap` (the source uses Jetpack Compose `Image`, but the encoding step itself is UI-framework-agnostic).

### Setup steps
1. Add the ZXing `core` dependency (Gradle version catalog or direct coordinate) — no `zxing-android-embedded` or camera/scanning module needed if you only generate codes, not scan them.
2. Write a small pure function that takes the text to encode + a pixel size, calls `QRCodeWriter().encode(text, BarcodeFormat.QR_CODE, sizePx, sizePx)` to get a `BitMatrix`, then walks every pixel of the matrix into a `Bitmap` (black/white per bit).
3. Wrap that pure function in a memoized UI component keyed by the input text, so the (synchronous, CPU-bound) encode only re-runs when the encoded content actually changes, not on every recomposition/render.
4. Render the resulting bitmap at whatever display size is needed — encode at a fixed higher resolution (e.g. 512px) once and let the UI layer scale down, rather than re-encoding at the exact display size every time.

### Core pattern
```kotlin
// QrCode.kt — generic ZXing QR bitmap generator + Compose wrapper
private fun generateQrBitmap(text: String, sizePx: Int): Bitmap {
    val matrix = QRCodeWriter().encode(text, BarcodeFormat.QR_CODE, sizePx, sizePx)
    val bitmap = Bitmap.createBitmap(sizePx, sizePx, Bitmap.Config.RGB_565)
    for (x in 0 until sizePx) {
        for (y in 0 until sizePx) {
            bitmap.setPixel(x, y, if (matrix.get(x, y)) android.graphics.Color.BLACK else android.graphics.Color.WHITE)
        }
    }
    return bitmap
}

@Composable
fun QrCodeImage(text: String, modifier: Modifier = Modifier, size: Dp = 180.dp) {
    // Keyed on `text` — encoding only re-runs when the underlying content
    // actually changes, not on every recomposition.
    val bitmap = remember(text) { generateQrBitmap(text, sizePx = 512) }
    Image(bitmap = bitmap.asImageBitmap(), contentDescription = null, modifier = modifier.size(size))
}
```

```ts
// Equivalent shape for a web/TS stack (e.g. using the `qrcode` npm package
// instead of ZXing directly) — same "pure encode function, memoized by input" pattern.
import QRCode from "qrcode";

export async function generateQrDataUrl(text: string, sizePx = 512): Promise<string> {
  return QRCode.toDataURL(text, { width: sizePx, margin: 1 });
}
```

### Env vars
none — purely client-side/on-device encoding library, no external network calls or configuration.

### Gotchas
- Encode once at a fixed, higher-than-display resolution (e.g. 512px) and scale down in the UI layer, rather than re-encoding at whatever pixel size the component happens to render at — re-encoding is CPU work you don't need to repeat.
- Memoize/cache the encode by its input text (`remember(text)` in Compose, a `useMemo`/cache keyed by text in React) — `QRCodeWriter().encode(...)` is synchronous CPU work and will visibly jank a render loop if re-run on every frame/recomposition instead of only when the encoded text changes.
- `Bitmap.Config.RGB_565` (no alpha channel) is sufficient and cheaper than `ARGB_8888` for a plain black/white QR code — don't reach for a heavier bitmap config unless you need transparency.
- This only covers QR *generation*; scanning/decoding QR codes needs a different ZXing entry point (and typically camera permissions) — don't assume the same dependency covers both directions.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter generates a QR code image encoding a
shareable invite code, so it can be scanned to join a group.
