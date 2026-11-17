# AMVerge iOS: Architecture and Implementation Plan

This document describes the technical architecture for the native iOS (iPhone and iPad) version of AMVerge. It is built on a pure native Apple stack using Swift, SwiftUI, AVFoundation, FFmpegKit (XCFramework), and SwiftData storage, mirroring the architecture established for the Android version.

## 1. Overview

Desktop AMVerge relies on external Python (`AMVerge-CLI`) and standalone `ffmpeg` child processes via `std::process::Command`. 

On iOS, Apple’s sandbox strictly prohibits `fork()` and `exec()`. Spawning command-line child processes is rejected by the iOS kernel and App Store review guidelines. To achieve smooth 60 to 120 FPS ProMotion performance and compliant execution, the iOS app is designed as a 100% native Swift application without WebView wrappers.

### Core Stack
* **UI Framework**: Swift 6 and SwiftUI (declarative native UI with ProMotion 120Hz support)
* **Media Playback**: AVFoundation (`AVPlayer` and `AVPlayerLayer`)
* **Instant Trimming & Muxing**: `FFmpegKit.xcframework` (C/C++ libraries compiled for `arm64-apple-ios` and iOS Simulator)
* **Thumbnail Generation**: `AVAssetImageGenerator` (hardware-accelerated frame extraction)
* **Scene Detection**: `AVAssetReader` keyframe sync scanner and CoreML (TransNetV2)
* **Local Data Storage**: SwiftData (modern SQLite persistence) and iOS App Sandbox Container

## 2. Architecture Diagram

```
+-------------------------------------------------------------+
|                     UI LAYER (SwiftUI)                      |
|  - LazyVGrid for 2-column video scene browsing              |
|  - DragGesture thumbnail scrubbing across clip cards        |
|  - Sheet & presentationDetents for export & scenepacks      |
|  - SensoryFeedback (haptics) on clip multi-selection        |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|                 DATA LAYER (SwiftData ORM)                  |
|  - Episodes, scene cut points, and scenepack models         |
|  - Timestamp to thumbnail file path mappings                |
|  - Security-scoped bookmark resolution for Files app video  |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|              MEDIA ENGINE (AVFoundation + C++)              |
|  - FFmpegKit (C/C++): Instant stream-copy cuts (no re-encode)|
|  - AVAssetImageGenerator: Hardware-accelerated thumbnails   |
|  - AVPlayer: In-app video preview with scrubbing tolerance  |
|  - AVAssetReader: Fast keyframe sync sample packet parser   |
+-------------------------------------------------------------+
```

## 3. Video Processing and Cutting Pipeline

### A. Stream-Copy Trimming & Fast Merge (FFmpegKit XCFramework)
When cutting clips at keyframe boundaries, FFmpegKit performs container-level transmuxing. It copies video and audio bitstreams directly into the new MP4 container without decoding or re-encoding:

```swift
import Foundation
import ffmpegkit

final class VideoCutManager {
    static let shared = VideoCutManager()

    func cutClip(
        inputURL: URL,
        outputURL: URL,
        startSeconds: Double,
        durationSeconds: Double,
        completion: @escaping (Result<URL, Error>) -> Void
    ) {
        let command = "-ss \(startSeconds) -i \"\(inputURL.path)\" -t \(durationSeconds) -c copy -avoid_negative_ts make_zero \"\(outputURL.path)\""

        FFmpegKit.executeAsync(command) { session in
            guard let session = session else {
                completion(.failure(NSError(domain: "VideoCutManager", code: -1, userInfo: [NSLocalizedDescriptionKey: "Failed to initialize session"])))
                return
            }

            let returnCode = session.getReturnCode()
            if ReturnCode.isSuccess(returnCode) {
                completion(.success(outputURL))
            } else {
                let failLog = session.getFailStackTrace() ?? "Unknown error"
                completion(.failure(NSError(domain: "VideoCutManager", code: Int(returnCode?.getValue() ?? -1), userInfo: [NSLocalizedDescriptionKey: failLog])))
            }
        }
    }

    /// Fast Merge / Concat demuxer (merges multiple selected clips into one scenepack)
    func mergeClips(
        clipURLs: [URL],
        outputURL: URL,
        completion: @escaping (Result<URL, Error>) -> Void
    ) {
        let fileListURL = FileManager.default.temporaryDirectory.appendingPathComponent("concat_list.txt")
        let fileListContent = clipURLs.map { "file '\($0.path)'" }.joined(separator: "\n")
        try? fileListContent.write(to: fileListURL, atomically: true, encoding: .utf8)

        let command = "-f concat -safe 0 -i \"\(fileListURL.path)\" -c copy \"\(outputURL.path)\""
        FFmpegKit.executeAsync(command) { session in
            try? FileManager.default.removeItem(at: fileListURL)
            if ReturnCode.isSuccess(session?.getReturnCode()) {
                completion(.success(outputURL))
            } else {
                completion(.failure(NSError(domain: "VideoCutManager", code: -1, userInfo: [NSLocalizedDescriptionKey: session?.getFailStackTrace() ?? "Concat failed"])))
            }
        }
    }
}
```

### B. Scene Detection Methods
1. **Keyframe Sync Scanner (Default)**:
   * Uses `AVAssetReader` and `AVAssetReaderTrackOutput`.
   * Inspects `CMSampleBuffer` packet metadata to locate sync samples (`kCMSampleAttachmentKey_NotSync == false`).
   * Extracts all keyframe cut timestamps for a full 24-minute episode in 2 to 3 seconds without rendering frames.
2. **AI Cut Detection (TransNetV2 CoreML)**:
   * Converted from PyTorch / ONNX to Apple CoreML (`TransNetV2.mlmodelc`) utilizing FP16 quantization.
   * Runs directly on Apple's Neural Engine (ANE) and Apple Silicon GPU with minimal battery draw.

## 4. SwiftUI and Touch Interactions

### A. Clip Grid (LazyVGrid)
* **iPhone (Portrait)**: 2-column adaptive layout showing thumbnail posters, duration pills (`00:04.2`), and timestamp badges (`01:14 - 01:18`).
* **iPad & Landscape**: 3 to 5 columns with a collapsible sidebar and multi-window support.
* Grid cells display pre-generated cached images to ensure smooth 120Hz ProMotion scrolling without dropped frames.

### B. Touch Scrubbing Pattern
Because mobile screens lack mouse hover events:
* Dragging a finger horizontally across a clip card scrubs through a 10-frame pre-cached thumbnail strip.
* In the active video preview player, `AVPlayer` uses zero-tolerance precise scrubbing:
  ```swift
  player.seek(to: targetTime, toleranceBefore: .zero, toleranceAfter: .zero)
  ```

### C. Multi-Select Gestures
* **Long Press**: Holding a clip for 200ms triggers sensory haptic feedback (`UIImpactFeedbackGenerator(style: .medium)`) and enters multi-select mode.
* **Drag-to-Select**: Dragging across cards toggles selection status.
* **Floating Bottom Dock**: Shows selected clip count, total runtime, and quick actions:
  - Add to Scenepack
  - Save to Photos (Camera Roll)
  - Share via AirDrop / CapCut

### D. Native Share Sheet Integration
Uses `UIActivityViewController` to share exported clips directly into iOS editing tools (CapCut, Premiere Rush) or social platforms (Instagram, TikTok).

## 5. Local Storage & Sandbox Architecture

iOS enforces strict sandboxing:

| Storage Area | iOS Path | Access Method | Purpose |
|---|---|---|---|
| Caches Directory | `FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)` | Swift `URL` / `FileHandle` | Temporary video cuts, thumbnail cache |
| Application Support | `FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask)` | SwiftData / CoreData | Database storing episodes, cut points, and scenepacks |
| Photos Library | `PHPhotoLibrary.shared()` | PhotoKit API | Final exported clips and scenepacks saved to Camera Roll |
| Source Video Picking | Device / iCloud Drive | `UIDocumentPickerViewController` | Access source videos via security-scoped URLs |

### SwiftData Models
* `EpisodeModel`: id, title, sourceBookmark, durationSeconds, createdAt
* `SceneModel`: id, episode, sceneIndex, startSeconds, endSeconds, thumbnailPath
* `ScenepackModel`: id, name, createdAt, scenes

## 6. Device-Tier Optimizations & Performance Matrix

To ensure stability across all supported iPhone and iPad models—from older 3GB/4GB RAM devices (iPhone XR, 11, SE 2/3) up to the latest 8GB A17/A18 Pro devices—the application incorporates strict memory and hardware tiering.

### A. Defeating the iOS "Jetsam" Low-Memory Killer
Older iPhones impose strict per-process memory limits (~300MB to 400MB before the kernel sends `SIGKILL`). Naively decoding full-frame video thumbnails easily exceeds this ceiling.
* **Maximum Thumbnail Resolution Clamping**:
  ```swift
  let generator = AVAssetImageGenerator(asset: asset)
  // Clamp decoded frame size to exact display resolution (approx. 230 KB uncompressed vs. 33 MB for 4K)
  generator.maximumSize = CGSize(width: 320, height: 180)
  ```
* **Memory-Pressure Defense (`NSCache`)**:
  - Total memory cache limit capped at **25MB**.
  - Immediate purge triggered upon receiving `UIApplication.didReceiveMemoryWarningNotification`:
    ```swift
    NotificationCenter.default.addObserver(
        forName: UIApplication.didReceiveMemoryWarningNotification,
        object: nil,
        queue: .main
    ) { _ in
        ThumbnailCache.shared.clearMemoryCache()
    }
    ```

### B. 10-Bit Anime HEVC (H.265) Hardware Decoding
Most high-end anime releases and scenepacks are mastered in **10-bit HEVC (Main 10 Profile)**.
* **A12 Bionic and Newer (iPhone XR, XS, 11, 12, 13, 14, 15, 16)**: Full hardware-accelerated 10-bit HEVC decoding. Plays natively at 60 FPS with minimal battery drain.
* **Legacy Devices (Older than A12, e.g. iPhone X / 8)**: Hardware decoder only handles 8-bit HEVC; 10-bit triggers software fallback. On these devices, the app automatically generates a lightweight 480p H.264 proxy for smooth timeline scrubbing.

### C. Thermal Management & QoS Dispatching
* **Stream-Copy Trimming**: Utilizing container transmuxing (`-c copy`) requires **0% CPU/GPU re-encoding work**, enabling 10+ clips to export in seconds without triggering device heat.
* **Background Tasks Quality-of-Service**: Keyframe scanning runs on `DispatchQueue.global(qos: .utility)` to prevent CPU thread spikes and thermal throttling.

### D. Display & Orientation Adaptability
* **60Hz Standard iPhones (iPhone 11, 12, 13, 14, 15, 16)**: Touch scrub events throttled to 30–60 FPS to prevent decoder congestion.
* **120Hz ProMotion iPhones (iPhone 13 Pro – 16 Pro, iPad Pro)**: `LazyVGrid` operates at full 120 FPS adaptive frame rate for fluid browsing through hundreds of scene cuts.
* **Screen Layouts**:
  - Compact (iPhone SE / mini): 2-column layout with half-height swipe-down sheets.
  - Standard & Pro Max: 2-column layout with floating action bar.
  - iPadOS: 3 to 5 columns with Stage Manager, Split View, and native drag-and-drop.

## 7. Scene Scout (Mobile Semantic Scene Search)

Unlike heavy desktop AI features (Depth Map and Real-ESRGAN Upscaling, which require dedicated multi-gigabyte desktop GPUs and are omitted from mobile), **Scene Scout** is fully supported on iOS through native hardware acceleration and zero-shot vector retrieval.

### A. Core Engine & Shared SQLite Architecture
Desktop AMVerge's Scene Scout stores embeddings and scene metadata in a standard SQLite schema (`processed_videos` and `scene_embeddings`). The iOS implementation directly opens and queries these `.db` files:
* **Binary Compatibility**: Users can transfer an entire indexed anime library database (e.g. `JujutsuKaisen.db`) from Windows/macOS directly to iPhone via AirDrop, iCloud Drive, or the Files app.
* **Vector Representation**: Stored as L2-normalized float32 byte buffers (`BLOB`).

### B. Apple Neural Engine (ANE) Query Encoder
When an editor types a text query (e.g., *"sword fight in the rain"* or *"crying closeup"*):
* The text is tokenized and encoded using a quantized CoreML version of the text encoder (running via `apple/coreml-mobileclip` or `SigLIP2` CoreML).
* The text encoder executes on the **Apple Neural Engine (ANE)** in **under 15 milliseconds** with 0% GPU load and negligible battery impact.

### C. Accelerate Framework (SIMD vDSP) Vector Search
Comparing a query embedding against 10,000+ scene embeddings in a database is accelerated using Apple's native **Accelerate Framework (`vDSP`)**:
```swift
import Accelerate

final class SceneScoutEngine {
    /// Vectorized dot-product (cosine similarity) using hardware SIMD lanes
    static func cosineSimilarity(query: [Float], target: [Float]) -> Float {
        guard query.count == target.count else { return 0 }
        // For L2-normalized embeddings, dot product equals cosine similarity
        return vDSP.dot(query, target)
    }

    /// Evaluates top-k matches across the SQLite database in under 30ms
    static func rankScenes(queryVector: [Float], candidates: [(sceneId: Int, vector: [Float])], topK: Int = 50) -> [(sceneId: Int, score: Float)] {
        return candidates
            .map { (sceneId: $0.sceneId, score: cosineSimilarity(query: queryVector, target: $0.vector)) }
            .sorted { $0.score > $1.score }
            .prefix(topK)
            .map { $0 }
    }
}
```

### D. Workflow on Mobile
1. **Search**: Instant text search surfaces matching scenes with similarity ranking pills (e.g. `94% match`).
2. **Preview**: Tap any result to scrub the exact scene range using hardware-decoded video.
3. **One-Tap Action**: Add to mobile scenepack or stream-copy cut directly into Photos/CapCut.

## 8. Development Roadmap

1. **Phase 1**: SwiftUI navigation scaffold, document picker for video files, and security-scoped resource management.
2. **Phase 2**: `AVAssetReader` keyframe scanner and `FFmpegKit.xcframework` stream-copy trimming pipeline.
3. **Phase 3**: SwiftData persistence schema, `AVAssetImageGenerator` clamped batch thumbnail generation, and memory-pressure purge handlers.
4. **Phase 4**: `LazyVGrid` UI with touch scrubbing, haptic multi-select gestures, and floating action dock.
5. **Phase 5**: Scene Scout mobile integration (SQLite reader, Accelerate vDSP SIMD dot-product ranking, and CoreML ANE text encoder).
6. **Phase 6**: PhotoKit export integration, AirDrop/Share sheet support, and iPadOS split-screen layout.
