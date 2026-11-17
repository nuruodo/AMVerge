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

### A. Stream-Copy Trimming (FFmpegKit XCFramework)
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

## 6. Development Roadmap

1. **Phase 1**: SwiftUI navigation scaffold, document picker for video files, and security-scoped resource management.
2. **Phase 2**: `AVAssetReader` keyframe scanner and `FFmpegKit.xcframework` stream-copy trimming pipeline.
3. **Phase 3**: SwiftData persistence schema and `AVAssetImageGenerator` batch thumbnail generation.
4. **Phase 4**: `LazyVGrid` UI with touch scrubbing, haptic multi-select gestures, and floating action dock.
5. **Phase 5**: PhotoKit export integration, AirDrop/Share sheet support, and iPadOS split-screen layout.
