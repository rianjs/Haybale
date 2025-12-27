# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

Haybale is a macOS-native file search application.

## Developer Context

The developer is experienced with C#/.NET but new to Swift, Xcode, and macOS development. Explain Swift/Xcode concepts when introducing them.

## Architecture Preferences

- **Object-functional separation**: Separate "types that do" (services, logic) from "types that hold data" (like C# records/record structs). In Swift, this means using `struct` for data types and `class` or `actor` for services.
- **Clean separation of concerns**: Don't mix abstraction levels. Keep UI, business logic, and data access distinct.
- **TDD for business logic**: Components with significant business logic should be developed test-first. Write tests before implementation.
- **UI responsiveness**: Keep heavy work off the main thread. Use Swift's async/await and actors for concurrent operations.

## Build Commands

This is a macOS SwiftUI application built with Xcode. Use xcodebuild from the command line:

```bash
# Build the project
xcodebuild -project Haybale/Haybale.xcodeproj -scheme Haybale -configuration Debug build

# Build for release
xcodebuild -project Haybale/Haybale.xcodeproj -scheme Haybale -configuration Release build

# Run tests (when test targets are added)
xcodebuild -project Haybale/Haybale.xcodeproj -scheme Haybale test
```

## Project Structure

- `Haybale/Haybale/` - Main application source code
  - `HaybaleApp.swift` - App entry point (@main)
  - `ContentView.swift` - Root SwiftUI view
  - `Assets.xcassets/` - Asset catalog (images, colors, app icon)

## Technical Details

- **Platform**: macOS (deployment target 26.2)
- **UI Framework**: SwiftUI
- **Swift Version**: 5.0
- **Bundle ID**: com.TidyApi.Haybale
- **Concurrency**: Uses Swift 6 concurrency features (SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor, SWIFT_APPROACHABLE_CONCURRENCY enabled)
