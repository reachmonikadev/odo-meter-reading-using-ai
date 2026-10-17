# odometer_scanner

[![pub package](https://img.shields.io/pub/v/odometer_scanner.svg)](https://pub.dev/packages/odometer_scanner)

Flutter module: capture an odometer photo → on-device blur check → Gemini extraction → business validation → typed `OdometerReading`.

## How it works

1. **Capture**: the user takes a photo or picks one from the gallery.
2. **On-device check**: blurry photos are rejected right away, without a network call.
3. **AI extraction**: Gemini reads the odometer value and unit, and flags screens, edits, glare or a missing odometer.
4. **Validation**: your business rules run (previous reading, max reading, min confidence).
5. **Result**: the user sees the verified reading and confirms it, or gets a clear reason with how-to-fix hints and retakes the photo.

| Verified (mechanical) | Verified (digital) | Rejected (photo of a screen) | Rejected (edited image) |
|:-:|:-:|:-:|:-:|
| <img src="https://raw.githubusercontent.com/reachmonikadev/odo-meter-reading-using-ai/main/packages/odometer_scanner/screenshots/verified_mechanical.jpg" width="200" alt="Verified mechanical odometer"> | <img src="https://raw.githubusercontent.com/reachmonikadev/odo-meter-reading-using-ai/main/packages/odometer_scanner/screenshots/verified_digital.jpg" width="200" alt="Verified digital odometer"> | <img src="https://raw.githubusercontent.com/reachmonikadev/odo-meter-reading-using-ai/main/packages/odometer_scanner/screenshots/rejected_screen_photo.jpg" width="200" alt="Rejected photo of a screen"> | <img src="https://raw.githubusercontent.com/reachmonikadev/odo-meter-reading-using-ai/main/packages/odometer_scanner/screenshots/rejected_stock_image.jpg" width="200" alt="Rejected edited image"> |

## Install

```bash
flutter pub add odometer_scanner
```

Get a Gemini API key at https://aistudio.google.com/apikey.

## Platform setup

Camera access goes through [`image_picker`](https://pub.dev/packages/image_picker).

**iOS** — add to `ios/Runner/Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Used to photograph your odometer.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Used to pick an odometer photo.</string>
```

**Android** — no extra setup required.

## Usage

```dart
final reading = await OdometerScanner.scan(
  context,
  config: OdometerScannerConfig(
    geminiApiKey: const String.fromEnvironment('GEMINI_API_KEY'),
    previousReading: 42000, // optional: new reading must be >= this
  ),
);
if (reading != null) print('${reading.value} ${reading.unit.symbol}');
```

Other ways to use it:

| Need | API |
|---|---|
| Push the full-screen flow | `OdometerScanner.scan(context, config: …)` |
| Embed in your own route / tabs | `OdometerScannerPage(config: …, onConfirmed: …)` |
| Your own UI, our logic | `BlocProvider(create: (_) => deps.createCubit(), child: YourView())` |
| Headless, no UI | `OdometerScannerDependencies.createAnalyzer(config)(capturedImage)` |
| Custom camera / backend proxy | `OdometerScannerDependencies(captureService: …, analyzeOdometerImage: …)` |
| Extra business rule | Implement `ReadingValidationRule`, pass to `OdometerReadingValidator(rules: […])` |

## Config

| Field | Default | Purpose |
|---|---|---|
| `geminiApiKey` | — | Google AI Studio key |
| `geminiModel` | `gemini-flash-latest` | Any vision-capable Gemini model id |
| `previousReading` | `null` | Rejects readings that go backwards |
| `maxReading` | `9999999` | Sanity upper bound |
| `minConfidence` | `0.75` | Minimum AI confidence |
| `blurThreshold` | `80` | Laplacian-variance threshold for the local check (`0` disables) |
| `maxImageWidth` / `imageQuality` | `1600` / `85` | Downscale before upload |

## What gets rejected

Blurry photo (on-device *and* AI), glare / too dark, not an odometer, trip meter only, digits cut off or obstructed, low AI confidence, reading above max, reading lower than the previous one, and photos of a screen or edited images. Every failure includes a title, message and "how to fix" hints shown to the user.

## Architecture (Clean + Cubit)

```
lib/src
├── core/          Result<T>, sealed OdometerFailure hierarchy, config
├── domain/        entities, repository + service interfaces,
│                  validation rules (strategy), AnalyzeOdometerImage use case
├── data/          Gemini REST data source (structured JSON output), DTO,
│                  repository impl, image_picker service, Laplacian blur checker
├── presentation/  OdometerScannerCubit (sealed states), page, widgets
└── di/            OdometerScannerDependencies (composition root)
```

- Domain depends on nothing; data and presentation depend on domain interfaces (DIP).
- Expected errors are values (`Result`), not exceptions; the repository maps transport errors to domain failures.
- Gemini is called over plain HTTP, so there's no SDK lock-in and it's easy to point at a backend proxy.

## Security note

`--dart-define` keys still ship inside the app binary. For production, route Gemini calls through your backend (or Firebase AI Logic with App Check) by supplying a custom `OdometerRepository`.

## Example

See [`example/lib/main.dart`](example/lib/main.dart).

## License

MIT — see [LICENSE](LICENSE).
