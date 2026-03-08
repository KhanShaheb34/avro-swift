# Swift Rewrite Plan: Avro Silicon

## Context

Avro Silicon is a macOS Bengali phonetic input method (~2,700 lines of Objective-C with manual retain/release). The codebase works but uses legacy patterns: no ARC, raw NSDictionary JSON access, binary search over patterns, SQLite C API for a read-only dictionary. This rewrite modernizes the codebase to Swift for better maintainability, type safety, and performance — without adding new features.

**Key decisions:**
- **InputMethodKit stays** — no modern alternative exists. Use from Swift with `@objc` bridging.
- **Replace SQLite with binary plist** — current code already loads everything into memory at startup; SQLite is just a file format.
- **Replace binary search with hash maps** — `[Int: [String: Pattern]]` for O(1) lookups instead of O(log n).
- **Use NSRegularExpression** (not Swift Regex) — preserves identical ICU regex behavior.
- **Convert String to Array<Character>** in hot paths — avoids O(n) indexed access.

---

## Directory Structure

```
Sources/
  AvroEngine/              (pure Swift logic — no AppKit/IMK)
    Models/
      Pattern.swift
      MatchRule.swift
      PhonemeTable.swift
      AutoCorrectEntry.swift
    Parser/
      PhoneticParser.swift
      RegexGenerator.swift
      CharacterClassifier.swift
    Dictionary/
      DictionaryStore.swift
      TableRouter.swift
    Suggestion/
      SuggestionEngine.swift
      SuffixMorphology.swift
      LevenshteinDistance.swift
    AutoCorrect/
      AutoCorrectStore.swift
    Cache/
      CacheManager.swift
    Preferences/
      PreferenceKeys.swift
    Resources/
      ResourcePath.swift
    AvroEngine.swift         (facade)
  AvroApp/                   (IMK + AppKit)
    main.swift
    AvroKeyboardController.swift
    CandidatesManager.swift
    IMPreferences.swift
    MainMenuAppDelegate.swift
    PreferencesController.swift
Resources/
  dictionary.plist            (NEW — converted from database.db3)
  data.json                   (unchanged)
  regex.json                  (unchanged)
  autodict.plist              (unchanged)
Tests/
  AvroEngineTests/
    PhoneticParserTests.swift
    RegexGeneratorTests.swift
    DictionaryStoreTests.swift
    SuggestionEngineTests.swift
    LevenshteinDistanceTests.swift
    IntegrationTests.swift
scripts/
  convert_db_to_plist.swift   (one-time migration)
```

---

## Phase 0: Build System + Data Migration

**Goal:** Swift-capable Xcode project + converted dictionary plist.

**Create:**
- `scripts/convert_db_to_plist.swift` — reads `database.db3` via sqlite3 C API, writes `Resources/dictionary.plist` as binary plist with structure: `{ tables: { "a": [...], "aa": [...] }, suffix: { "e": "ে", ... } }`
- `Resources/dictionary.plist` — output of conversion
- `AvroKeyboard-Bridging-Header.h` — empty initially

**Xcode project changes:**
- `SWIFT_VERSION = 5` (project-level)
- `CLANG_ENABLE_MODULES = YES`
- `SWIFT_OBJC_BRIDGING_HEADER` on target
- `DEFINES_MODULE = YES`

**Verify:** `xcodebuild` still succeeds, plist decodes correctly, word counts match SQLite.

---

## Phase 1: Pure Logic Layer

All pure computation — no IMK, no AppKit, no singletons. Each module independently testable.

### Phase 1a: Models + CharacterClassifier

**Files:** `Models/*.swift`, `Parser/CharacterClassifier.swift`, `Preferences/PreferenceKeys.swift`, `Resources/ResourcePath.swift`

- Codable structs for `data.json` and `regex.json` (replacing raw NSDictionary casts)
- `CharacterClassifier` uses `Set<Character>` for O(1) classification (replacing linear `inString:` scans)
- Custom `Decodable` for `Match.negative` field (JSON uses string `"YES"`/`"NO"`)

**Source ref:** `AvroParser.m:252-313` (character classification), `AvroPreferenceKeys.m` (constants)

### Phase 1b: PhoneticParser (replaces AvroParser)

**Files:** `Parser/PhoneticParser.swift`, `Tests/PhoneticParserTests.swift`

**Key change — hash map instead of binary search:**
```swift
private let patternMap: [Int: [String: Pattern]]  // length -> find -> Pattern
// Lookup: patternMap[chunkLen]?[chunk] — O(1) vs O(log 289)
```

Algorithm (same logic, different data structure):
1. `fix()` normalizes case (lowercase non-case-sensitive chars)
2. For each position, try chunk lengths 5→1
3. Hash map lookup instead of binary search
4. Evaluate contextual rules (prefix/suffix + vowel/consonant/punctuation/exact/number scopes)
5. XOR with negative flag for rule inversion
6. Output `.precomposedStringWithCanonicalMapping`

**Critical:** Convert input to `Array<Character>` for O(1) indexed access. Preserve exact `isExact` boundary checking logic from `AvroParser.m:286-290`.

**Source ref:** `AvroParser.m:89-250` (parse method), `data.json` (289 patterns)

**Verify:** All 20 parser regression cases from `tests/fixtures/regression_cases.json`.

### Phase 1c: RegexGenerator (replaces RegexParser) — parallel with 1b

**Files:** `Parser/RegexGenerator.swift`, `Tests/RegexGeneratorTests.swift`

Same hash map structure as PhoneticParser but:
- Loads `regex.json` (168 patterns)
- `clean()` strips non-case-sensitive chars entirely (vs `fix()` which keeps but lowercases them)
- Appends `(্[যবম])?(্?)([ঃঁ]?)` suffix to output
- No `.precomposedStringWithCanonicalMapping`

**Source ref:** `RegexParser.m:89-250`, `regex.json`

### Phase 1d: LevenshteinDistance — parallel with 1e

**Files:** `Suggestion/LevenshteinDistance.swift`, `Tests/LevenshteinDistanceTests.swift`

- Standard DP algorithm, returns `-1` for empty strings (matching current behavior at `NSString+Levenshtein.m:83-85`)
- Also port `isMatchedByRegex:` and `captureComponentsMatchedByRegex:` as String extensions using `NSRegularExpression`

**Source ref:** `NSString+Levenshtein.m:68-109`

### Phase 1e: AutoCorrectStore — parallel with 1d

**Files:** `AutoCorrect/AutoCorrectStore.swift`, `Tests/AutoCorrectStoreTests.swift`

- Loads `autodict.plist` into `[String: String]` dictionary
- `find()` takes parser as dependency (not singleton) for `fix()` call

**Source ref:** `AutoCorrect.m:36-78`

### Phase 1f: DictionaryStore + TableRouter (replaces Database)

**Files:** `Dictionary/DictionaryStore.swift`, `Dictionary/TableRouter.swift`, `Tests/DictionaryStoreTests.swift`

- Loads `dictionary.plist` via `PropertyListDecoder` into `[String: [String]]` tables + `[String: Set<String>]` lookup sets
- `TableRouter` extracts the first-character → table-names mapping (the giant switch from `Database.m:266-373`)
- Dual path: literal (`Set.contains`) or regex (`NSRegularExpression` scan)
- `NSCache` with 1024 item limit for query results

**Source ref:** `Database.m:76-443`

**Verify:** 9 `database_contains` + 3 `database_empty` regression cases.

### Phase 1g: CacheManager

**Files:** `Cache/CacheManager.swift`, `Tests/CacheManagerTests.swift`

- Weight cache: persistent plist at `~/Library/Application Support/OmicronLab/Avro Keyboard/weight.plist`
- Phonetic cache: in-memory `[String: [String]]`
- Suffix base cache: in-memory `[String: [String]]`

**Source ref:** `CacheManager.m:30-130`

### Phase 1h: SuggestionEngine + SuffixMorphology

**Files:** `Suggestion/SuggestionEngine.swift`, `Suggestion/SuffixMorphology.swift`, `Tests/SuggestionEngineTests.swift`

SuggestionEngine orchestrates: parser → autocorrect → dictionary → Levenshtein sort → suffix expansion.

SuffixMorphology handles Bengali-specific joining rules:
- Vowel + kar → insert য় between
- Base ends with ৎ → replace with ত + suffix
- Base ends with ং → replace with ঙ + suffix
- Otherwise → direct concatenation

**Source ref:** `Suggestion.m:82-248` (pipeline), `Suggestion.m:188-201` (morphology rules)

**Verify:** 7 `suggestion_contains` regression cases.

---

## Phase 2: AvroEngine Facade

**Files:** `AvroEngine.swift`

Single entry point that wires all components together with dependency injection (no singletons):
```swift
final class AvroEngine {
    let parser: PhoneticParser
    let regexGenerator: RegexGenerator
    let dictionary: DictionaryStore
    let autoCorrect: AutoCorrectStore
    let cacheManager: CacheManager
    let suggestionEngine: SuggestionEngine
}
```

**Verify:** End-to-end: `engine.suggestionEngine.suggestions(for: "ami")` contains "আমি".

---

## Phase 3: IMK Integration + AppKit UI

### Phase 3a: main.swift + MainMenuAppDelegate

- `main.swift`: IMKServer init, AvroEngine init, CandidatesManager init, nib loading, run loop
- `MainMenuAppDelegate`: persist cache on termination, menu wiring

### Phase 3b: AvroKeyboardController

**Must use `@objc(AvroKeyboardController)`** — Info.plist references this name as `InputMethodServerControllerClass`.

IMK method overrides:
- `inputText(_:client:)` — character input
- `didCommand(by:client:)` — command keys (backspace, enter, tab, arrows)
- `candidates(_:)` — return candidate list
- `candidateSelected(_:)` / `candidateSelectionChanged(_:)` — selection callbacks
- `composedString(_:)` — return composition buffer

**Preserve all IMK quirks:**
- `selectCandidate:` broken → use moveDown/moveRight loop
- `selectedCandidateString` returns nil → track index manually
- `panelType` immutable after init → deallocate/reallocate

**Source ref:** `AvroKeyboardController.m:34-407`

### Phase 3c: CandidatesManager

`@objc(Candidates)` — shared instance with explicit allocate/deallocate/reallocate for panel type changes.

### Phase 3d: PreferencesController

`@objc(PreferencesController)` — nib references this name. `autoCorrectItemsArray` must be `@objc dynamic` for Cocoa Bindings. `AutoCorrectItem` stays as `@objc class` (NSArrayController requires KVO).

**Verify:** Build, install to `~/Library/Input Methods/`, register in System Settings, manual smoke test (compose, select, backspace, enter, preferences).

---

## Phase 4: Cleanup

1. Remove all 15 `.h` + 15 `.m` files
2. Remove `database.db3` from bundle resources
3. Remove `libsqlite3.dylib` and `libicucore.dylib` from linked frameworks
4. Remove `Podfile`, `Podfile.lock`, bridging header
5. Port `tests/regression_runner.m` to XCTest
6. Update CI to run `xcodebuild test`
7. Update README

---

## Risk Mitigations

| Risk | Mitigation |
|------|------------|
| Unicode normalization differences | Use `.precomposedStringWithCanonicalMapping` (same Foundation method). Byte-level regression tests. |
| Regex behavior changes | Use `NSRegularExpression` (not Swift Regex) to preserve ICU engine. |
| NIB class name mismatch | Explicit `@objc(ClassName)` on all nib-referenced classes. |
| Swift String O(n) indexing | Convert to `Array<Character>` in parser hot paths. |
| Complex split regex in controller | Port as raw string literal, add dedicated tests. |
| IMKCandidates memory with ARC | Use `Optional` and set to `nil` for deallocation. |

---

## Verification

Each phase verifies against `tests/fixtures/regression_cases.json`:
- 20 parser cases (Phase 1b)
- 9 database_contains + 3 database_empty cases (Phase 1f)
- 7 suggestion_contains cases (Phase 1h)

Final verification:
1. `xcodebuild test` — all XCTest suites pass
2. `xcodebuild build` — Release build succeeds
3. `scripts/install.sh` — installs to ~/Library/Input Methods/
4. Manual IME testing: compose Bengali text in TextEdit, verify candidates, selection, backspace, enter/tab commit, preferences window
5. CI workflow passes (regression tests + build)
6. Perf: run `scripts/perf_report.sh` and compare latencies to baseline

