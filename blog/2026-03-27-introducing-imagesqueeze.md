---
slug: introducing-imagesqueeze
title: "Introducing ImageSqueeze: A Crash-Safe Android Image Compression Library"
authors: [kevin]
tags: [android, kotlin]
---

Handling image compression on Android shouldn't crash your app. That's why I built **ImageSqueeze** — a robust, crash-safe image compression library built in Kotlin, designed as a modern and more resilient alternative to [Zelory Compressor](https://github.com/zetbaitsu/Compressor).

<!-- truncate -->

## 🤔 Why I Built This

If you've worked on Android apps that deal with user photos — profile pictures, document uploads, chat attachments — you've probably used Zelory Compressor at some point. It works great in most cases, but in production environments with millions of users, edge cases start to surface:

- **`FileNotFoundException`** — source file got deleted or is inaccessible (Scoped Storage quirks)
- **`FileAlreadyExistsException`** — destination file cannot be overwritten
- **`ENOSPC`** — device storage is full
- **`OutOfMemoryError`** — image too large for the available heap
- **`BitmapFactory.decodeFile` returns null** — corrupt or unsupported image format

In Zelory, these scenarios result in **unhandled exceptions that crash your app**. I kept running into these issues in production, so I decided to build a library that handles all of these gracefully.

## ✨ What Makes ImageSqueeze Different

### Crash-Safe by Design

Every possible failure is captured and returned as a typed `SqueezeResult.Error` — **no unhandled exceptions, ever**. You decide how to handle each error:

```kotlin
when (result) {
    is SqueezeResult.Success -> {
        uploadToServer(result.file)
    }
    is SqueezeResult.Error -> {
        when (result.errorType) {
            SqueezeError.FILE_NOT_FOUND -> showRetakePhotoDialog()
            SqueezeError.NO_DISK_SPACE  -> showClearStoragePrompt()
            SqueezeError.OUT_OF_MEMORY  -> showLowerResolutionHint()
            SqueezeError.DECODE_FAILED  -> showUnsupportedFormatMessage()
            else -> {
                logToAnalytics(result.exception)
                showGenericError(result.message)
            }
        }
    }
}
```

### Clean Kotlin DSL

Configuration feels natural with the DSL builder:

```kotlin
val result = ImageSqueeze.compress(context, sourceFile) {
    resolution(1280, 720)       // Max width x height
    quality(85)                 // Starting JPEG quality (1–100)
    size(500_000L)              // Target max file size (500 KB)
    format(Bitmap.CompressFormat.WEBP) // Output format
}
```

### Kotlin Extension Functions

For an even more concise API:

```kotlin
import vinz.android.imagesqueeze.extensions.squeeze

val result = sourceFile.squeeze(context) {
    resolution(1024, 1024)
    quality(80)
    size(1_000_000L) // 1 MB
}
```

## 📦 Installation

ImageSqueeze is published on **Maven Central** — no custom repository configuration needed:

```kotlin title="build.gradle.kts"
dependencies {
    implementation("io.github.kevinmf1:imagesqueeze:1.1.0")
}
```

Or via **JitPack**:

```kotlin title="build.gradle.kts"
// In settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        maven { url = uri("https://jitpack.io") }
    }
}

// In app/build.gradle.kts
dependencies {
    implementation("com.github.kevinmf1:ImageSqueeze:v1.1.0")
}
```

## 🚀 Quick Start

### Basic Coroutine Usage

```kotlin
lifecycleScope.launch {
    val result = ImageSqueeze.compress(context, sourceFile)

    when (result) {
        is SqueezeResult.Success -> {
            val compressedFile = result.file
            // Upload, display, or save
        }
        is SqueezeResult.Error -> {
            Log.e("Compress", result.message)
        }
    }
}
```

### Custom Destination & Thread

```kotlin
val destination = File(getExternalFilesDir(null), "compressed_photo.jpg")

val result = ImageSqueeze.compress(
    context = context,
    source = sourceFile,
    destination = destination,
    dispatcher = Dispatchers.Default
) {
    quality(75)
}
```

### Jetpack Compose Support

ImageSqueeze works natively with Compose — no extra wrappers needed:

```kotlin
@Composable
fun CompressImageScreen(originalFile: File) {
    val context = LocalContext.current
    val coroutineScope = rememberCoroutineScope()
    var compressedResult by remember { mutableStateOf<SqueezeResult?>(null) }

    Button(onClick = {
        coroutineScope.launch {
            compressedResult = originalFile.squeeze(context) {
                resolution(1024, 1024)
                quality(80)
            }
        }
    }) {
        Text("Compress Now")
    }

    when (val res = compressedResult) {
        is SqueezeResult.Success -> Text("Saved ${res.file.length()} bytes!")
        is SqueezeResult.Error   -> Text("Error: ${res.message}")
        null                     -> Text("Waiting to compress...")
    }
}
```

### Synchronous API

For background threads you manage yourself:

```kotlin
val result = ImageSqueeze.compressSync(context, sourceFile) {
    quality(80)
}
```

## ⚙️ Configuration Options

| Option | Type | Default | Description |
|---|---|---|---|
| `resolution(w, h)` | `Int, Int` | `612 × 816` | Maximum output dimensions |
| `quality(q)` | `Int` | `80` | Starting JPEG quality (1–100) |
| `size(bytes)` | `Long` | `1,000,000` | Target max file size in bytes |
| `format(fmt)` | `CompressFormat` | `JPEG` | Output format (`JPEG`, `WEBP`, `PNG`) |
| `isForDisplay` | `Boolean` | `false` | Optimize for display (set `true` for previews) |
| `minQuality` | `Int` | `10` | Minimum quality threshold |

### Full Configuration Example

```kotlin
val result = ImageSqueeze.compress(context, sourceFile) {
    resolution(1920, 1080)
    quality(90)
    size(2_000_000L)        // 2 MB
    format(Bitmap.CompressFormat.WEBP)
    minQuality = 20         // Don't go below quality 20
}
```

## 🔬 How Compression Works

ImageSqueeze follows a battle-tested pipeline with safety checks at every step:

1. **Validate** — Check source file exists, is readable, and is non-empty
2. **Disk space guard** — Verify at least 10 MB of free storage
3. **Copy to cache** — Work on a safe copy to prevent source corruption
4. **Decode with `inSampleSize`** — Downsample using power-of-2 scaling to prevent OOM
5. **EXIF rotation** — Read orientation tag and rotate bitmap accordingly
6. **Iterative quality reduction** — Encode at starting quality, then step down by 10 until file size constraint is met or `minQuality` is reached
7. **Safe write** — Atomic write with temp file fallback
8. **Cleanup** — Delete working files from cache

## 🏗️ Architecture

```
imagesqueeze/
├── ImageSqueeze.kt         # Public API entry point
├── CompressionConfig.kt    # DSL configuration class
├── SqueezeResult.kt        # Sealed result wrapper (Success / Error)
├── SqueezeError.kt         # Error type enum
├── core/
│   └── CompressorCore.kt   # Internal compression engine
├── utils/
│   ├── FileUtil.kt         # File I/O, disk space checks
│   └── ImageUtil.kt        # Bitmap decoding, EXIF rotation
└── extensions/
    └── FileExt.kt          # Kotlin File extension functions
```

## 🧪 Testing

ImageSqueeze ships with **71 tests** covering configuration, validation, error handling, the compression pipeline, and the public API surface — all passing with 0 failures.

```bash
# Unit tests (JVM — no device required)
./gradlew :imagesqueeze:testDebugUnitTest

# Instrumented tests (requires device / emulator)
./gradlew :imagesqueeze:connectedDebugAndroidTest
```

## 📋 Requirements

- **Min SDK**: 21 (Android 5.0 Lollipop)
- **Kotlin**: 1.9+
- **Coroutines**: `kotlinx-coroutines-android`

## Links

- 📦 [GitHub Repository](https://github.com/kevinmf1/ImageSqueeze)
- 📖 [Maven Central](https://central.sonatype.com/artifact/io.github.kevinmf1/imagesqueeze)

---

If you're dealing with image compression on Android and tired of unexpected crashes in production, give ImageSqueeze a try. Feel free to ⭐ the repo, open issues, or submit PRs!
