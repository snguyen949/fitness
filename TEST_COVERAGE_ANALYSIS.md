# Test Coverage Analysis

## Current State

**Test coverage: 0%** -- The codebase has no test files, no test framework, no test configuration, and no CI/CD pipeline. The entire application lives in a single monolithic `index.html` file (~6,600 lines) containing CSS, HTML, and 120+ JavaScript functions.

---

## Architecture Challenges for Testing

Before diving into specific recommendations, it's worth noting that the monolithic single-file architecture makes testing harder than it needs to be. All functions are defined inside a `<script>` tag within `index.html`, which means they can't be directly imported by a test runner. To enable testing, two approaches are possible:

1. **Extract JS into a separate module** (recommended): Move the JavaScript into one or more `.js` files that export functions. This is the highest-leverage refactor for testability.
2. **Evaluate the script in tests**: Load and `eval()` the script block in a test harness -- fragile and not recommended long-term.

The analysis below assumes approach (1) would be done as a prerequisite.

---

## Priority 1: Pure Business Logic (High Impact, Easy to Test)

These functions have **zero DOM dependencies** and contain critical calculation logic. They are the easiest to extract and test, and bugs here directly affect the user experience.

### 1.1 `getParticipantScore(userId, challenge)` -- Challenge Scoring
**Risk: HIGH** | Lines 4385-4435

This function calculates leaderboard scores across 4 different challenge types (`volume`, `workouts`, `sets`, `reps`). Bugs here produce incorrect rankings in the competitive challenge feature.

**Recommended tests:**
- Volume calculation: verify `weight * reps` summing across exercises and workouts
- Volume with bodyweight (`'BW'`) exercises -- currently treated as weight=0, verify this
- Workouts count: simple count of workouts in date range
- Sets count: sum of `total_sets` across workouts
- Reps count: sum of reps across all exercises
- Edge case: workouts exactly on `startDate` and `endDate` boundaries (inclusive)
- Edge case: user with no workouts in range returns 0
- Edge case: exercises with missing `reps` or `weight` fields

### 1.2 `checkPR(exerciseName, weight, reps)` -- Personal Record Detection
**Risk: HIGH** | Lines 4551-4562

Determines whether a lift is a new personal record. The logic has a subtle behavior: it returns `false` on the very first record (no previous to beat), but still saves it. Subsequent records return `true` only if they beat the prior.

**Recommended tests:**
- First-ever record for an exercise: should return `false` but save the record
- New PR by volume (`weight * reps`): should return `true`
- New PR by weight alone: should return `true`
- New PR by reps alone: should return `true`
- Non-PR (lower across all metrics): should return `false`
- Case insensitivity: `"Bench Press"` and `"bench press"` should match

### 1.3 `formatValue(value, type)` -- Value Formatting
**Risk: MEDIUM** | Lines 4499-4504

**Recommended tests:**
- Volume >= 1000 formats as `"X.XK"` (e.g., 1500 -> `"1.5K"`)
- Volume < 1000 formats with `toLocaleString()`
- Non-volume types use `toLocaleString()` directly
- Edge cases: 0, 999, 1000, 1000000

### 1.4 `formatTime(seconds)` -- Time Formatting
**Risk: MEDIUM** | Lines 4544-4549

**Recommended tests:**
- `null`/`undefined`/`0` returns `'--'`
- Seconds only (e.g., 45 -> `"45s"`)
- Minutes and seconds (e.g., 90 -> `"1:30"`)
- Exact minute (e.g., 60 -> `"1:00"`)
- Large values (e.g., 3661 -> `"61:01"`)

### 1.5 `getDaysLeft(endDate)` -- Days Remaining Calculation
**Risk: MEDIUM** | Lines 4511-4516

**Recommended tests:**
- Future date returns positive number
- Past date returns 0 (clamped via `Math.max`)
- Today returns 0 or 1 depending on time of day
- Edge case: end-of-day boundary (`T23:59:59`)

### 1.6 `formatDate(dateStr)` and `formatDateNice(dateStr)` -- Date Display
**Risk: LOW** | Lines 4506-4509, 5729-5740

**Recommended tests:**
- `formatDate`: standard date string produces `"Mon DD"` format
- `formatDateNice`: today's date returns `"Today"`
- `formatDateNice`: yesterday's date returns `"Yesterday"`
- `formatDateNice`: older dates return `"Wed, Mon DD"` format

### 1.7 `getTypeLabel(type)` and `getTypeUnit(type)` -- Challenge Type Helpers
**Risk: LOW** | Lines 4491-4497

**Recommended tests:**
- All 4 known types return correct label/unit
- Unknown type returns the type string itself / empty string

### 1.8 `generateId()` -- ID Generation
**Risk: LOW** | Line 4519

**Recommended tests:**
- Returns a string starting with `'w_'`
- Two consecutive calls produce different IDs
- ID contains timestamp component

---

## Priority 2: Data Management (High Impact, Medium Difficulty)

These functions manage localStorage persistence and cloud sync. Bugs here can cause data loss.

### 2.1 `loadUserData()` / `saveUserData()` -- Local Persistence
**Risk: HIGH** | Lines 3012-3048

**Recommended tests (with mocked localStorage):**
- Loading saved data merges with defaults (exercises, muscles arrays preserved)
- Loading with no saved data returns clean defaults
- Loading with corrupted JSON doesn't crash (catches error)
- Saving serializes `appData` correctly
- Default fields are always present even if saved data is partial

### 2.2 `getAllWorkouts()` -- Workout Aggregation
**Risk: HIGH** | Lines 4521-4537

Merges imported workouts (for the "Song" user) with manually logged workouts. Bugs here affect the calendar, log, dashboard, and stats.

**Recommended tests:**
- Song user gets both imported and manual workouts
- Non-Song users only get manual workouts
- Workouts are grouped by date correctly
- Manual workouts have `source: 'manual'`
- Imported workouts have `source: 'imported'`

### 2.3 `importData(event)` -- Data Import
**Risk: HIGH** | Lines 4082-4100

**Recommended tests:**
- Valid JSON import merges into existing appData
- Invalid JSON shows error alert, doesn't corrupt state
- Import triggers save and refresh

### 2.4 `migrateSongProfile()` -- Data Migration
**Risk: MEDIUM** | Lines 2906-2933

**Recommended tests:**
- Skips migration when users already exist
- Creates Song profile when no users exist
- Migrates old `workoutTrackerData` from localStorage
- Handles corrupted old data gracefully

### 2.5 `getLatestBenchmarkScores()` -- Benchmark Score Aggregation
**Risk: MEDIUM** | Lines 6118-6126

**Recommended tests:**
- Returns latest score per benchmark exercise across all trials
- Later trials overwrite earlier ones for the same exercise
- Null/empty scores are skipped
- Values are parsed as floats

---

## Priority 3: Workout Logic (High Impact, Medium Difficulty)

### 3.1 `saveWorkout()` -- Workout Save Logic
**Risk: HIGH** | Lines 5899-5940

The most critical user-facing function. Requires DOM mocking but has complex aggregation logic worth testing.

**Recommended tests:**
- Exercises with 0 reps are excluded
- `total_sets` is computed correctly across exercises
- Bodyweight exercises: when all sets are BW, weight is `'BW'`; when mixed, max non-BW weight is used
- Empty workout (no exercises with reps) shows alert, doesn't save
- `checkPR` is called for each exercise
- Workout gets a unique ID from `currentWorkout`

### 3.2 `calculateLeaderboard(challenge)` -- Leaderboard Ranking
**Risk: MEDIUM** | Lines 4365-4383

**Recommended tests:**
- Results are sorted descending by value
- Missing users (not in `allUsers`) are skipped
- Each participant entry includes name, color, initial, value
- Ties are handled (stable sort)

### 3.3 `finalizeChallenge(challenge)` -- Challenge Completion
**Risk: MEDIUM** | Lines 4438-4445

**Recommended tests:**
- Winner is set to the top-ranked participant
- Status changes to `'completed'`
- Empty leaderboard: no winner is set

### 3.4 `getExerciseMuscle(exerciseName)` -- Muscle Group Lookup
**Risk: LOW** | Lines 4539-4542

**Recommended tests:**
- Known exercise returns correct muscle group
- Case-insensitive matching
- Unknown exercise returns `'other'`

---

## Priority 4: Cloud Sync (Medium Impact, Harder to Test)

### 4.1 `syncToCloud()` -- Cloud Push
**Risk: HIGH** | Lines 3069-3125

**Recommended tests (with fetch mocking):**
- Sends all users and current user's data
- Includes user settings (weeklyGoal, theme, benchmark settings)
- Sets `lastUpdated` timestamp
- Handles network failure gracefully (sets status to 'offline')
- Prevents concurrent syncs (`isSyncingTo` guard)

### 4.2 `syncFromCloud()` -- Cloud Pull
**Risk: HIGH** | Lines 3127+

**Recommended tests (with fetch mocking):**
- Loads users and userData from cloud response
- Applies user settings to localStorage
- Handles network failure gracefully
- Prevents concurrent syncs (`isSyncingFrom` guard)

### 4.3 `syncChallengesToCloud()` / `loadChallengesFromCloud()`
**Risk: MEDIUM** | Lines 4119-4160

**Recommended tests:**
- Challenge data round-trips correctly through sync
- Handles missing syncUrl (returns early)

---

## Priority 5: Edge Cases and Defensive Logic

### 5.1 Input Validation
- `saveNewExercise()`: duplicate exercise IDs, empty names, no muscle selected
- `saveNewMuscle()`: duplicate muscle names, empty input
- `saveTrial()`: no data entered shows alert

### 5.2 Date Boundary Handling
- Calendar rendering across month/year boundaries
- Challenge start/end date filtering (inclusive vs exclusive)
- Streak calculation logic across gaps

### 5.3 Profile Switching
- Switching profiles saves current data before loading new
- Profile deletion doesn't corrupt other profiles' data

---

## Recommended Test Infrastructure Setup

```bash
# 1. Initialize the project
npm init -y

# 2. Install test framework (Vitest is lightweight and fast)
npm install --save-dev vitest jsdom

# 3. Add test script to package.json
# "scripts": { "test": "vitest run", "test:watch": "vitest" }
```

### Suggested file structure after refactoring:

```
fitness/
  index.html              # HTML + CSS + script tag importing app.js
  js/
    utils.js               # formatDate, formatTime, formatValue, generateId, etc.
    workout.js             # saveWorkout logic, checkPR, getAllWorkouts
    challenges.js          # scoring, leaderboard, finalization
    storage.js             # localStorage + cloud sync
    benchmarks.js          # benchmark scoring and trial management
    profiles.js            # user profile management
  tests/
    utils.test.js
    workout.test.js
    challenges.test.js
    storage.test.js
    benchmarks.test.js
    profiles.test.js
  package.json
  vitest.config.js
```

### Quick-win: extractable pure functions for immediate testing

These functions can be extracted with **zero refactoring** of the rest of the app -- just copy them into a module and export:

| Function | Lines | Reason |
|---|---|---|
| `formatTime(seconds)` | 4544-4549 | Zero dependencies |
| `formatValue(value, type)` | 4499-4504 | Zero dependencies |
| `formatDate(dateStr)` | 4506-4509 | Zero dependencies |
| `getDaysLeft(endDate)` | 4511-4516 | Zero dependencies |
| `getTypeLabel(type)` | 4491-4492 | Zero dependencies |
| `getTypeUnit(type)` | 4495-4496 | Zero dependencies |
| `generateId()` | 4519 | Zero dependencies |

---

## Summary

| Priority | Category | Functions | Estimated Tests | Difficulty |
|---|---|---|---|---|
| P1 | Pure Business Logic | 10 | ~45 | Easy |
| P2 | Data Management | 5 | ~25 | Medium |
| P3 | Workout Logic | 4 | ~20 | Medium |
| P4 | Cloud Sync | 4 | ~15 | Hard |
| P5 | Edge Cases | ~8 | ~15 | Medium |
| **Total** | | **~31 functions** | **~120 tests** | |

**Recommended starting point:** Extract the 7 pure utility functions (Priority 1, "Quick-win" table above) into a `js/utils.js` module and write ~25 unit tests. This gives immediate coverage of the most reusable logic with minimal refactoring risk.
