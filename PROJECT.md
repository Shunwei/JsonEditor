# JsonEditor

A cross-platform JSON editor for **macOS** and **iPadOS** built with SwiftUI from a single shared codebase.

---

## Features

- Live JSON validation with line-number error reporting
- Pretty-print formatting (configurable 2 or 4 space indent)
- Minify / compact output
- Interactive tree sidebar for exploring the JSON structure
- Native document model — Open/Save on macOS, Files app on iPad
- macOS menu bar commands (Format, Minify, Toggle Tree)
- iPad share sheet for exporting JSON

---

## Requirements

| Platform | Minimum Version |
|----------|----------------|
| macOS    | 14.0 (Sonoma)  |
| iPadOS   | 17.0           |

No external dependencies. Pure Swift + SwiftUI.

---

## Project Structure

```
JsonEditor/
├── App/
│   └── JsonEditorApp.swift
├── Models/
│   ├── JSONDocument.swift
│   └── JSONNode.swift
├── Services/
│   └── JSONService.swift
├── ViewModels/
│   └── EditorViewModel.swift
├── Views/
│   ├── EditorView.swift
│   ├── EditorToolbar.swift
│   ├── JSONTextEditor.swift
│   ├── TreeView.swift
│   └── ValidationBar.swift
└── Platform/
    ├── PlatformBridge.swift
    └── JSONEditorCommands.swift
```

---

## Architecture

The codebase is split into five layers. Dependencies only flow downward.

```
┌─────────────────────────────────────────┐
│                   App                   │  JsonEditorApp.swift
├─────────────────────────────────────────┤
│                  Views                  │  EditorView, TreeView, …
├─────────────────────────────────────────┤
│               ViewModel                 │  EditorViewModel
├─────────────────────────────────────────┤
│         Models        Services          │  JSONDocument, JSONNode, JSONService
├─────────────────────────────────────────┤
│            Platform Glue                │  PlatformBridge, JSONEditorCommands
└─────────────────────────────────────────┘
```

### Layer responsibilities

| Layer | File(s) | Responsibility |
|-------|---------|---------------|
| App | `JsonEditorApp.swift` | `@main` entry, `DocumentGroup` scene, macOS commands |
| Models | `JSONDocument.swift` | `ReferenceFileDocument` — read/write `.json` files |
| Models | `JSONNode.swift` | Lightweight tree model built from parsed JSON |
| Services | `JSONService.swift` | Validate, format, minify, parse — pure Swift, no UI |
| ViewModel | `EditorViewModel.swift` | All editor state; debounced validation; document mutations |
| Views | `EditorView.swift` | `NavigationSplitView` layout (sidebar + detail) |
| Views | `EditorToolbar.swift` | `ToolbarContent` — buttons, picker, placement per platform |
| Views | `JSONTextEditor.swift` | `TextEditor` wrapper with monospace font |
| Views | `TreeView.swift` | Recursive outline of the JSON tree |
| Views | `ValidationBar.swift` | Bottom status bar showing valid / error state |
| Platform | `PlatformBridge.swift` | `FocusedValue` injection (macOS) + share sheet (iOS) |
| Platform | `JSONEditorCommands.swift` | macOS-only menu commands via `Commands` |

---

## Shared vs Platform-Specific

| File | macOS | iPadOS | Notes |
|------|:-----:|:------:|-------|
| `JsonEditorApp.swift` | ✓ | ✓ | `DocumentGroup` works on both |
| `JSONDocument.swift` | ✓ | ✓ | |
| `JSONNode.swift` | ✓ | ✓ | |
| `JSONService.swift` | ✓ | ✓ | Pure Swift, zero platform imports |
| `EditorViewModel.swift` | ✓ | ✓ | |
| `EditorView.swift` | ✓ | ✓ | |
| `EditorToolbar.swift` | ✓ | ✓ | Two `#if os(macOS)` for placement only |
| `JSONTextEditor.swift` | ✓ | ✓ | One `#if os(iOS)` for keyboard hints |
| `TreeView.swift` | ✓ | ✓ | |
| `ValidationBar.swift` | ✓ | ✓ | |
| `PlatformBridge.swift` | ✓ | ✓ | Has macOS and iOS branches internally |
| `JSONEditorCommands.swift` | ✓ | — | Entire file guarded by `#if os(macOS)` |

**10 of 12 files contain zero platform-conditional code.**

---

## Key Design Decisions

### `ReferenceFileDocument` over `FileDocument`
`JSONDocument` is a class (`ReferenceFileDocument`) so SwiftUI views can hold a direct reference to it without copying on every edit. Combined with `ObservableObject`, changes to `document.text` propagate to all observers automatically.

### `@Observable` for `EditorViewModel`
Uses the Swift 5.9 `@Observable` macro instead of `ObservableObject`. No `@Published` boilerplate; views only re-render for the properties they actually read.

### Debounced validation
`EditorViewModel.textDidChange(_:)` cancels any pending `Task` and schedules a new one with a 250 ms delay. This avoids re-parsing on every keystroke without needing Combine or a timer.

### `NavigationSplitView` for layout
A single `NavigationSplitView` in `EditorView` serves both platforms:
- **macOS** — classic sidebar + detail window chrome
- **iPadOS** — collapsible panel with swipe gestures

Column visibility is bound to `viewModel.showTree`, so the toolbar toggle and sidebar drag gesture stay in sync automatically.

### `FocusedValues` for macOS Commands
Menu `Commands` can't hold direct references to a document's view model. Instead, `PlatformBridge` injects the active window's `EditorViewModel` and `JSONDocument` into `FocusedValues` so `JSONEditorCommands` can read them from `@FocusedValue`.

---

## Xcode Setup

1. **New Project** → Multiplatform App (check both iOS and macOS destinations).
2. Add all 12 `.swift` files to the single shared target.
3. Set deployment targets: **macOS 14**, **iOS 17**.
4. No Swift Package dependencies required.
5. The app uses `UTType.json` — no custom `Info.plist` keys needed; JSON is a system type.

---

## Extending the Editor

| Goal | Where to change |
|------|----------------|
| Add syntax highlighting | `JSONTextEditor.swift` — swap `TextEditor` for a `NSViewRepresentable`/`UIViewRepresentable` |
| Add a JSON Schema validator | `JSONService.swift` — add a `validateAgainstSchema(_:schema:)` method |
| Support JSON5 or YAML | `JSONService.swift` + `JSONDocument` content types |
| Add find & replace | New `SearchViewModel` + a sheet presented from `EditorView` |
| iPad keyboard shortcuts | `EditorView.swift` — add `.keyboardShortcut()` modifiers to toolbar buttons |
| Persist settings (indent size, theme) | `EditorViewModel` — replace stored properties with `@AppStorage` |
