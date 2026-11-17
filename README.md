# AMVerge iOS

> This branch is for the native iOS (iPhone & iPad) version of AMVerge. It is currently under planning and development.

A native iOS app designed for anime music video (AMV) creators and editors who need to browse, split, organize, and export scenepacks directly on iPhone and iPad.

- **UI Framework**: Swift 6 and SwiftUI (with 120Hz ProMotion support)
- **Media Engine**: AVFoundation (`AVPlayer`, `AVAssetReader`, `AVAssetImageGenerator`)
- **Native Video Processing**: `FFmpegKit.xcframework` (C/C++ stream-copy trimming with zero re-encoding)
- **AI Scene Detection**: CoreML (Apple Neural Engine acceleration)
- **Persistence**: SwiftData and Apple Sandbox Storage

## Current Status

This version is currently in the planning and design phase. Complete architectural specifications, cutting pipelines, and touch gesture interactions are documented in [IOS_ARCHITECTURE.md](IOS_ARCHITECTURE.md).

## Planned Features

- Fast keyframe scene detection using `AVAssetReader` sync sample parsing
- Instant stream-copy trimming with zero re-encoding via `FFmpegKit`
- Touch-friendly 2-column clip grid with horizontal thumb-scrubbing
- Long-press multi-selection with sensory haptics and drag-to-select
- Direct export into Apple Photos (Camera Roll), AirDrop, and iOS Share Sheet (CapCut, TikTok, Instagram)
- iPadOS split-screen and stage manager support
