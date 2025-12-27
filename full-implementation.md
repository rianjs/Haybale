# Haybale - Full Implementation Plan

> A fast, flexible file search CLI tool for macOS

---

## Table of Contents

1. [Overview](#overview)
2. [CLI Interface](#cli-interface)
3. [Architecture](#architecture)
4. [Implementation Phases](#implementation-phases)
5. [Project Structure](#project-structure)
6. [Test Cases](#test-cases)
7. [Decisions Summary](#decisions-summary)

---

## Overview

Haybale is a macOS-native file search tool built with a **CLI-first, TDD approach**. The core search logic is implemented as a testable library (`HaybaleCore`), with a CLI wrapper on top. A GUI can be added later.

### Key Principles

- **Object-functional separation**: Data types (`struct`) are separate from services (`class`/`actor`)
- **Composability**: Pure finders at the bottom, context-aware matchers above
- **Testability**: All components injectable and mockable
- **TDD**: Tests written before implementation for business logic

---

## CLI Interface

### Interactive Prompts

```
File name [*]: ______                      # wildcards: *, \*, \;  |  regex: r:pattern
Containing text: ______                    # wildcards: *, \*      |  regex: r:pattern
Look in [.]: _________                     # tab-completable, default current dir
Recursion depth [unlimited]: ______        # empty=unlimited, 0=current dir only, N=depth
Case sensitive [N]: Y/N                    # content search only (filenames always case-insensitive)
Include hidden files [N]: Y/N              # dotfiles and dot-directories
Min size (KB) [none]: ______               # units: KB, MB (no suffix = KB)
Max size (KB) [none]: ______               # units: KB, MB (no suffix = KB)
Modified after [none]: ______              # ISO-8601 (2024-01-15) or relative (7d, 2w, 3m)
Modified before [none]: ______             # ISO-8601 (2024-01-15) or relative (7d, 2w, 3m)
Search binary files [N]: Y/N               # extract embedded strings from binaries
```

### Pattern Syntax

| Pattern | Meaning |
|---------|---------|
| `*` | Match any characters (zero or more) |
| `\*` | Literal asterisk |
| `\;` | Literal semicolon |
| `\r:` | Literal "r:" (escape regex prefix) |
| `*.html;*.htm` | Multiple patterns (semicolon-delimited) |
| `r:pattern` | Regex mode (full control) |

**Note**: No `?` wildcard. Use regex for single-character matching.

### Output Format

```
/path/to/matching/file.txt
  123.  line containing the match
  456.  another match in same file

/another/file.txt
  789.  match here
```

- Results stream to **stdout** as found
- Progress/errors go to **stderr**

---

## Architecture

### Layer Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         Layer 6: CLI                         │
│  Interactive prompts, tab completion, output formatting      │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                  Layer 5: SearchCoordinator                  │
│  Orchestrates traversal, applies filters/matchers, streams   │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼───────┐   ┌─────────▼─────────┐   ┌──────▼──────┐
│ FileNameMatcher│   │  ContentSearcher   │   │  FileFilter │
│ (Layer 3)      │   │  (Layer 3)         │   │  (Layer 4)  │
└───────┬───────┘   └─────────┬─────────┘   └─────────────┘
        │                     │
        │           ┌─────────┴─────────┐
        │           │                   │
        │   ┌───────▼───────┐   ┌───────▼────────┐
        │   │BinaryDetector │   │StringExtractor │
        │   │ (Layer 2)     │   │ (Layer 2)      │
        │   └───────────────┘   └────────────────┘
        │
┌───────▼────────────────────────────────────────┐
│            Layer 1: Pure Finders                │
│     StringFinder  |  RegexFinder                │
│     (context-free, just match text)             │
└─────────────────────────────────────────────────┘
```

### Layer 1: Pure Finders

Context-free text matchers. Know nothing about files.

```swift
protocol Finder {
    func findMatches(in text: String) -> [MatchRange]
}
```

| Component | Description |
|-----------|-------------|
| `StringFinder` | Handles `*` wildcard, `\*`, `\;`, `\r:` escapes. Has `caseSensitive` param. |
| `RegexFinder` | Wraps `NSRegularExpression`. No case param—user controls via `(?i)`. |

**Case Sensitivity:**
- `StringFinder`: Respects app toggle. Uses `String.range(of:options:)` with `.caseInsensitive` (Unicode folding) or `.literal`.
- `RegexFinder`: No app override. Pattern executed exactly as written.

**Regex Detection:** Pattern prefixed with `r:` → RegexFinder. Otherwise → StringFinder.

### Layer 2: Binary Detection + String Extraction

| Component | Description |
|-----------|-------------|
| `BinaryDetector` | Checks first 8KB for null bytes. Null present → binary. |
| `StringExtractor` | Scans bytes for printable sequences (ASCII, UTF-8, UTF-16LE). Returns strings + byte offsets. |

**Binary Toggle Behavior:**
- **N (default)**: Skip files detected as binary
- **Y**: Search all. Binary → `BinaryContentSearcher`, Text → `ContentSearcher`

### Layer 3: Context-Aware Matchers

| Component | Description |
|-----------|-------------|
| `FileNameMatcher` | Splits on `;`, composes with Finder. Always case-insensitive. Matches filename only. |
| `ContentSearcher` | Reads file, composes with Finder. Supports multi-line regex. 2 lines context. UTF-8 → Latin-1 fallback. |
| `BinaryContentSearcher` | Uses `StringExtractor` then applies Finder to extracted strings. |

**ContentSearcher Details:**
- Multi-line regex: Supported via `(?s)` or `\n`
- Size limit: Files >50MB use line-by-line only (multi-line skipped with warning)
- Context: 2 lines before/after. Multi-line match spanning N-M gets context N-2 to M+2.
- Empty pattern: No content search, filename match only.

### Layer 4: File Filtering

`FileFilter` determines if a file should be searched:
- **Size**: KB/MB units. No suffix = KB. No bytes support.
- **Date**: ISO-8601 (`2024-01-15`) or relative (`7d`, `2w`, `3m`)
- **Hidden**: Skip dotfiles/dot-directories unless `includeHidden = true`

### Layer 5: Coordination

`SearchCoordinator` orchestrates everything:
- **Depth**: 0 = current dir files only, N = N levels, empty = unlimited
- **Symlinks**: Follow with cycle detection (track visited inodes)
- **Errors**: Warn inline to stderr, continue searching
- **Binary routing**: Check first 8KB, route to appropriate searcher
- **Output**: Yields results as async stream

### Layer 6: CLI

- Interactive prompts with defaults shown
- Tab completion for paths
- Progress on stderr, results on stdout
- **Future**: Non-interactive CLI args, ANSI match highlighting

### Data Types (structs)

```swift
struct MatchRange           // start/end indices
struct SearchQuery          // all search parameters
struct SearchOptions {
    caseSensitive: Bool     // content only
    minSize: Int?           // in KB
    maxSize: Int?           // in KB
    modifiedAfter: Date?
    modifiedBefore: Date?
    searchBinaries: Bool
    includeHidden: Bool
    maxDepth: Int?          // nil = unlimited
}
struct SearchResult         // path + content matches
struct ContentMatch         // line number, text, ranges, context
struct ExtractedString      // value, byte offset, encoding
```

---

## Implementation Phases

### Phase 1: Project Setup
1. Create Swift Package with CLI target
2. Add test target
3. Create folder structure
4. Delete existing Xcode project files

### Phase 2: Pure Finders
1. `StringFinder` - `*` wildcard, `\*`, `\;`, `\r:` escapes, case sensitivity
2. `RegexFinder` - wrapper with error handling, multi-line support
3. `Finder` protocol
4. Pattern parsing (`r:` prefix detection)

### Phase 3: FileNameMatcher
1. Parse `;` delimiters (respecting `\;`)
2. Compose with Finder (always case-insensitive)
3. Match against filename only
4. Support `r:` prefix

### Phase 4: ContentSearcher
1. Encoding detection (UTF-8 → Latin-1)
2. Line-by-line and multi-line regex
3. Compose with Finder (respect case toggle)
4. Return matches with 2-line context
5. Support `r:` prefix

### Phase 5: Binary Detection + Extraction
1. `BinaryDetector` - null byte check
2. `StringExtractor` - ASCII extraction
3. `StringExtractor` - UTF-8 extraction
4. `StringExtractor` - UTF-16LE extraction
5. `BinaryContentSearcher` - compose with Finder

### Phase 6: FileFilter
1. Size parsing (KB/MB, no suffix = KB)
2. Date parsing (ISO-8601, relative)
3. Hidden file detection
4. Combined filtering logic

### Phase 7: File System Abstraction
1. `FileSystemProvider` protocol
2. `RealFileSystemProvider` with symlink handling
3. `MockFileSystemProvider` for tests
4. Cycle detection (track visited inodes)

### Phase 8: SearchCoordinator
1. Traversal with configurable depth
2. Apply filters and matchers
3. Binary detection and routing
4. Symlink cycle detection
5. Hidden file filtering
6. Error handling (warn and continue)
7. Async stream output

### Phase 9: CLI
1. Interactive prompts with hints
2. Tab completion for paths
3. Output formatting (path + indented matches)
4. Progress on stderr
5. Input parsing (sizes, dates, depth)

---

## Project Structure

```
Haybale/
├── Package.swift
├── README.md
├── Sources/
│   ├── HaybaleCore/                    # Library (testable)
│   │   ├── Models/
│   │   │   ├── MatchRange.swift
│   │   │   ├── SearchQuery.swift
│   │   │   ├── SearchOptions.swift
│   │   │   ├── SearchResult.swift
│   │   │   ├── ContentMatch.swift
│   │   │   └── ExtractedString.swift
│   │   ├── Finders/
│   │   │   ├── Finder.swift            # Protocol
│   │   │   ├── StringFinder.swift
│   │   │   └── RegexFinder.swift
│   │   ├── Extraction/
│   │   │   ├── BinaryDetector.swift
│   │   │   └── StringExtractor.swift
│   │   ├── Matchers/
│   │   │   ├── FileNameMatcher.swift
│   │   │   ├── ContentSearcher.swift
│   │   │   └── BinaryContentSearcher.swift
│   │   ├── Filtering/
│   │   │   └── FileFilter.swift
│   │   ├── FileSystem/
│   │   │   ├── FileSystemProvider.swift
│   │   │   └── RealFileSystemProvider.swift
│   │   └── Coordination/
│   │       └── SearchCoordinator.swift
│   └── Haybale/                        # CLI executable
│       └── main.swift
└── Tests/
    └── HaybaleCoreTests/
        ├── StringFinderTests.swift
        ├── RegexFinderTests.swift
        ├── FileNameMatcherTests.swift
        ├── ContentSearcherTests.swift
        ├── BinaryDetectorTests.swift
        ├── StringExtractorTests.swift
        ├── FileFilterTests.swift
        └── SearchCoordinatorTests.swift
```

---

## Test Cases

### StringFinder

| Input | Expected |
|-------|----------|
| `*` | Matches empty string, any characters |
| `foo*bar` | Matches `foobar`, `foo123bar`, `foo/path/bar` |
| `\*` | Matches literal `*` |
| `\;` | Matches literal `;` |
| `\r:foo` | Matches literal `r:foo` |
| Case toggle | Respects `caseSensitive` parameter |

### RegexFinder

| Input | Expected |
|-------|----------|
| Valid regex | Returns matches |
| Invalid regex | Returns error (no crash) |
| `(?i)hello` | Matches "HELLO" (user-controlled case) |
| `(?s)` patterns | Multi-line matching works |

### FileNameMatcher

| Input | Expected |
|-------|----------|
| `*.html` | Matches `index.html`, not `index.htm` |
| `*.htm*` | Matches `index.html` and `index.htm` |
| `*.html;*.htm` | Splits and matches both |
| `foo\;bar.txt` | Matches literal `foo;bar.txt` |
| `*` | Matches files with/without extensions |
| `*.*` | Only matches files with extensions |
| Any pattern | Always case-insensitive |
| `r:pattern` | Triggers regex mode |

### ContentSearcher

| Scenario | Expected |
|----------|----------|
| Match found | Returns line number + 2 lines context |
| Multi-line match (N-M) | Context is N-2 to M+2 |
| File > 50MB | Line-by-line only, multi-line regex skipped with warning |
| Non-UTF-8 file | Falls back to Latin-1 |
| Empty pattern | No content search |

### BinaryDetector

| Input | Expected |
|-------|----------|
| File with null bytes in first 8KB | Binary |
| File without null bytes | Text |
| Empty file | Text |
| File < 8KB | Check entire file |

### FileFilter

| Input | Expected |
|-------|----------|
| `100` | 100 KB |
| `100KB` | 102,400 bytes |
| `50MB` | 52,428,800 bytes |
| `7d` | 7 days ago |
| `2w` | 2 weeks ago |
| `2024-01-15` | ISO-8601 date |

### StringExtractor

| Scenario | Expected |
|----------|----------|
| ASCII in binary | Extracts strings of minimum length |
| UTF-16LE | Detects alternating null byte pattern |
| Mixed content | Returns byte offsets for each string |

### SearchCoordinator

| Scenario | Expected |
|----------|----------|
| `maxDepth = 0` | Files in target dir only |
| `maxDepth = N` | N levels deep |
| Symlink cycle | Detected and avoided |
| `includeHidden = false` | Skips dotfiles |
| Permission denied | Warns to stderr, continues |

---

## Decisions Summary

| Topic | Decision |
|-------|----------|
| `?` wildcard | Not supported—use regex |
| Escape `;` | `\;` |
| Escape `r:` prefix | `\r:` |
| Regex mode | `r:` prefix |
| Filename case | Always case-insensitive |
| Content case | Toggle, default case-insensitive |
| Binary detection | Null byte check in first 8KB |
| Binary toggle N | Skip binary files |
| Binary toggle Y | Search all (auto-route by detection) |
| Multi-line regex | Supported (>50MB: line-by-line only) |
| Context lines | 2 before/after |
| Empty content pattern | Filename match only |
| Symlinks | Follow with cycle detection |
| Hidden files | Configurable, default exclude |
| Max depth | 0=current dir, N=levels, empty=unlimited |
| Errors | Warn inline to stderr, continue |
| Encoding | UTF-8 first, fallback Latin-1 |
| CLI args | Deferred to later phase |
| Output format | Path + indented line numbers |
| Match highlighting | Future (ANSI colors) |
| Progress | Streaming + stderr status |
| Default path | Current directory |
| Size units | KB/MB only, no suffix = KB |
| Date formats | ISO-8601 + relative (7d, 2w, 3m) |

---

## Future Enhancements

- [ ] Non-interactive CLI argument mode
- [ ] ANSI color match highlighting
- [ ] macOS GUI (SwiftUI)
- [ ] JSON output format
- [ ] Configurable context lines
