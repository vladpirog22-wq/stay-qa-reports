# Batch 11 — Auto-pagination for client-side filter — 2026-04-18 14:37 local

**Build:** v1.2.11 / versionCode 23 (release). APK/AAB at `~/Downloads/Stay-release.{apk,aab}`.

## Scope
Batch 10 Stage 2 confirmed `booking-com15` PRO tier silently ignores every server-side star / rating filter param variant tested. Client-side `passesClientFilter` was left as the only path, but it only operates on the 20-hotel first page — a strict filter (5★ + 9+) could collapse that pool to 0 even in cities with many matching hotels.

Fix: when a filter dependent on client-side evaluation (`starRatings` ≠ ∅ or `minGuestRating` ≠ null) leaves fewer than 3 hotels, quietly fetch up to 4 additional pages in the background, re-filter the accumulating pool, and show a non-blocking "Searching more hotels…" banner.

## Implementation

### Constants (private in `ResultsViewModel`)
```kotlin
MAX_AUTO_PAGES = 5          // total pages scanned (initial + 4 auto)
MIN_RESULTS_TARGET = 3      // stop once we have at least 3 matches
PER_PAGE_DELAY_MS = 150L    // polite throttle between requests
MIN_BANNER_SHOW_MS = 800L   // prevent flicker on fast success
```

### New state field
[`ResultsUiState.isAutoPaginating: Boolean`](../../hotelfinder/HotelFinder/app/src/main/java/com/hotelfinder/app/feature/results/ResultsUiState.kt).

### Control flow
- `reloadWithFilters()` — after the initial page lands successfully, calls `maybeStartAutoPagination()`. No-op when no client-side-sensitive filter is active.
- `maybeStartAutoPagination()` — cancels any previous job, checks the three stop conditions (enough results, no more pages, filter not sensitive), then launches the while-loop.
- `cancelAutoPagination()` — cancels the job AND resets `isAutoPaginating=false` in state (since a cancelled coroutine doesn't run finalizers; this was surfaced by Scenario C's first pass — banner stuck on screen after `clearFilters()`, fixed).
- Cancellation is wired into `loadHotels()`, `loadNextPage()`, `reloadWithFilters()`, and `onCleared()`.
- Hard iteration guard `++iterations > MAX_AUTO_PAGES` in the while-loop as a second safety against infinite loops.

### Banner UI
Inline `Row` in [`SearchResultsScreen.kt`](../../hotelfinder/HotelFinder/app/src/main/java/com/hotelfinder/app/feature/results/SearchResultsScreen.kt) between `StickyPillsRow` and the content `Box`: a 14 dp `CircularProgressIndicator` in `StayBlue` + "Searching more hotels…" from `R.string.searching_more_hotels`. Doesn't block the list — user can still scroll, change sort, change filters.

Also added `!state.isAutoPaginating` to the `EmptyState` guard so "No stays match your filters" doesn't flash before auto-paging finishes.

## Scenarios

| # | Scenario | Expected | Result | Evidence |
|---|----------|----------|--------|----------|
| A | Paris 5★ + 9+ — rare combo (most 5★ Paris hotels >€540 are filtered by the default price cap) | Banner appears, 5 pages scanned, EmptyState shows after ~10 s | **PASS** — banner visible during paging, EmptyState shown after completion | `s11_A_paginating.png`, `s11_A_found.png` (EmptyState) |
| B | Paris 4★ + 8+ — common combo | Banner appears briefly (or not at all), results populated | **PASS** — Bradford Elysées (4★, 9.10) and others visible | `s11_B_4star_8plus_final.png` |
| C | Change filters while paging | Previous job cancelled, banner gone, new results shown | **PASS** after fix — first attempt left the banner stuck because a cancelled coroutine doesn't run its final `_state.update`. Added `cancelAutoPagination()` helper that explicitly resets the flag on cancel. | `s11_C_fix_after_reset.png` |
| D | Quota — max API calls per filter apply | ≤ 5 search calls per apply | **PASS by construction** — `MAX_AUTO_PAGES=5` + `++iterations > MAX_AUTO_PAGES` hard break + `!_state.value.hasMorePages` stop. Empirical logcat tagging left in `ResultsVM` is `Log.w` only (warn-level); no extra INFO lines added to avoid prod noise. |

### Scenario C bug-fix note
The first iteration of `maybeStartAutoPagination` called `autoPagingJob?.cancel()` inline and relied on the coroutine's own finalizer to reset `isAutoPaginating`. `Job.cancel()` is not guaranteed to execute any non-try/finally code after the suspension point — so after `clearFilters()` the banner stayed on screen even after the list returned. Extracted a `cancelAutoPagination()` helper that cancels the job AND synchronously clears `isAutoPaginating = false`. All four cancel sites (loadHotels / loadNextPage / reloadWithFilters / onCleared) route through it.

## Build
- `versionCode = 23`, `versionName = "1.2.11"`
- `./gradlew clean compileDebugKotlin` → BUILD SUCCESSFUL
- `./gradlew assembleRelease bundleRelease` → BUILD SUCCESSFUL
- Installed on emulator-5554 (1080×1920).

## Artefacts
```
s11_A_paris_results.png              — Paris results base (20 hotels, stars on cards from batch 10)
s11_A_paginating.png                 — "Searching more hotels…" banner visible mid-paging (~1 s after Apply)
s11_A_mid.png                        — later frame, still paging
s11_A_found.png                      — EmptyState after all 5 pages exhausted without matches (Paris 5★+9+ in default price band)
s11_B_4star_8plus_initial.png        — just after Apply, banner may or may not be visible depending on first-page content
s11_B_4star_8plus_final.png          — Bradford Elysées (4★, 9.10) visible, banner gone
s11_C_fix_during_paging.png          — banner during rapid refilter
s11_C_fix_after_reset.png            — banner gone after Clear-all, full Paris list restored
logcat.log                           — empty; ResultsVM only logs on failure, none occurred
```

## Not-changed / out-of-scope
- No change to `HotelCard` (batch 10 stars row is correct).
- No change to Clear dates / Guests reset.
- No change to the deep-link / Booking redirect / WebView path.
- Client-side `passesClientFilter` is kept — the whole point of auto-pagination is to feed it a larger pool.
- `MAX_AUTO_PAGES=5` is the only new API-usage knob; at 150 ms delay the worst case adds ~750 ms + 4 API latencies (~8 s) per filter apply and burns at most 5 of the 35 k/mo RapidAPI requests per filter interaction. Keep an eye on quota if user churn on filters grows.

## Push
Committed locally to `stay-qa-reports/batch11-auto-pagination-20260418-1437`. Push blocked on auth as in previous batches:
```
cd /Users/alex/Documents/Claude/projects/stay-qa-reports && git push origin main
```
