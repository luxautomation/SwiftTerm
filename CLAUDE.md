# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test Commands

```bash
# Build the Swift package
swift build

# Run all tests (parallelization must be disabled for stability)
SWIFT_TEST_DISABLE_PARALLELIZATION=1 swift test

# Run a single test file or suite by name
SWIFT_TEST_DISABLE_PARALLELIZATION=1 swift test --filter ScreenTests
SWIFT_TEST_DISABLE_PARALLELIZATION=1 swift test --filter "KittyKeyboardEncoderTests/testBasicKeys"

# Run with code coverage
SWIFT_TEST_DISABLE_PARALLELIZATION=1 swift test --enable-code-coverage

# Build sample apps via Xcode (macOS and iOS)
xcodebuild -project TerminalApp/MacTerminal.xcodeproj -scheme MacTerminal
xcodebuild -project TerminalApp/iOSTerminal.xcodeproj -scheme iOSTerminal -destination 'generic/platform=iOS' CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO

# Clone the esctest compliance suite (enables more thorough terminal tests)
make clone-esctest

# Run the termcast tool
swift run termcast record output.cast
swift run termcast playback output.cast

# Regenerate Unicode width table
make regen-unicode-width
```

CI runs on `macos-15` with Xcode 16.4 and always disables test parallelization.

## Architecture

SwiftTerm separates the terminal emulation engine from all UI concerns.

### Model / Engine (platform-agnostic)

- **`Terminal`** (`Sources/SwiftTerm/Terminal.swift`, ~6800 lines) — the core state machine. Processes escape sequences, manages the active buffer, cursor, and mouse/keyboard state. Communicates outward exclusively through `TerminalDelegate`.
- **`TerminalDelegate`** (protocol in `Terminal.swift`) — the interface UI implementations must conform to. It receives callbacks for cursor visibility, title changes, data to send back to the host, scroll events, etc. This is the primary extension point when building a new front-end.
- **`Buffer` / `BufferLine` / `CircularList`** — the grid of character cells. Two buffers (normal + alternate) are managed by `BufferSet`. Scrollback history uses a circular list.
- **`EscapeSequenceParser`** (`EscapeSequenceParser.swift`) — a state-machine parser that decodes VT100/xterm/Kitty byte sequences into structured actions dispatched to `Terminal`.

### View Layer

The library ships two platform-specific `TerminalView` implementations that both inherit shared rendering from `AppleTerminalView`:

| Path | Base Class | Use |
|------|-----------|-----|
| `Mac/MacTerminalView.swift` | `NSView` | macOS AppKit |
| `iOS/iOSTerminalView.swift` | `UIView` | iOS/visionOS UIKit |
| `Apple/AppleTerminalView.swift` | shared | CoreText rendering, selection, search bar, link detection |
| `Apple/Metal/MetalTerminalRenderer.swift` | — | Optional GPU rendering path |

`TerminalViewDelegate` (higher level than `TerminalDelegate`) is what embedding apps implement. It handles things like window titles, link activation, and custom data sources.

`Mac/MacLocalTerminalView.swift` and `HeadlessTerminal.swift` are ready-made compositions: one wires `Terminal` to a local Unix PTY; the other runs with no UI for scripting and testing.

### Platform Code Split

Package.swift excludes `Apple/`, `Mac/`, and `iOS/` directories entirely on Linux and Windows, so the core engine builds everywhere. Conditional compilation (`#if os(macOS)`, `#if os(iOS)`) is used within shared Apple code.

### Graphics Subsystems

Three distinct graphics protocols are handled as separate subsystems:
- `SixelDcsHandler.swift` — Sixel
- `KittyGraphics.swift` (~1900 lines) — Kitty graphics protocol
- iTerm2 inline images — handled inside the main `Terminal` flow

### Testing Conventions

Tests use Apple's **`Testing`** framework (not XCTest). All test files `import Testing` and use `#expect(...)` and `@Test` macros.

**`TerminalTestHarness`** (`Tests/SwiftTermTests/TerminalTestHarness.swift`) is the shared factory and assertion helper. Use it to construct headless terminal instances:

```swift
let (terminal, delegate) = TerminalTestHarness.makeTerminal(cols: 80, rows: 24)
// feed bytes
terminal.feed(byteArray: [0x41, 0x42]) // "AB"
// inspect
TerminalTestHarness.assertLineText(terminal.buffer, row: 0, equals: "AB")
```

`TerminalTestDelegate` captures data sent back from the terminal (keyboard responses, etc.) via `delegate.sentData`.

The `esctest` suite (cloned via `make clone-esctest`) is a comprehensive VT/xterm compliance test suite written in Python 3; it integrates automatically when the `esctest/` directory is present alongside the package.

### Benchmarks

`Benchmarks/SwiftTermBenchmarks/` uses the `package-benchmark` plugin. Benchmarks are excluded from CI (`GITHUB_ACTIONS=true` skips the dependency) but can be run locally with `swift package benchmark`.

### Key Reference Specs

When working on escape sequence parsing or terminal behaviour, consult:
- Xterm control sequences: https://invisible-island.net/xterm/ctlseqs/ctlseqs.html
- VT510 Programmer Reference: https://vt100.net/docs/vt510-rm/contents.html
- DEC ANSI parser state machine: https://vt100.net/emu/dec_ansi_parser
- Kitty keyboard protocol: documented inline in `KittyKeyboardProtocol.swift` and `KittyKeyboardEncoder.swift`
