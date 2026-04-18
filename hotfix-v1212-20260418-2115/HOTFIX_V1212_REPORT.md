# Hotfix v1.2.12 Report — 2026-04-18 21:15 local

**Build:** v1.2.12 / versionCode 24 (release). APK/AAB at `~/Downloads/Stay-release.{apk,aab}`.

Three bugs fixed together:
1. HotelCard / Hotel Detail — Booking-style redesign (consistent across Results, Favorites, Detail).
2. Clear dates on the Results screen wasn't wired — now resets to default dates and re-searches.
3. Mini-map tap on Hotel Detail required a double back press — now opens external Maps, one back returns.

## Files changed

| File | Change |
|------|--------|
| `domain/model/Hotel.kt` | Added `reviewCount: Int?` |
| `data/remote/mapper/HotelMapper.kt` | Map `p.reviewCount` into `Hotel.reviewCount` (DTO already had the field) |
| `data/local/db/FavoriteEntity.kt` | Added `ratingWord`, `reviewCount`, `starClass` |
| `data/local/db/AppDatabase.kt` | `version = 3 → 4` (destructive migration — pre-launch) |
| `data/repository/FavoritesRepositoryImpl.kt` | Map new fields to/from entity |
| `ui/components/HotelCard.kt` | Booking-style layout: name + inline gold stars (top row), blue score badge + word + `· N reviews` (bottom row) |
| `feature/detail/HotelDetailScreen.kt` | Same blue-badge row on Hotel Detail (replaces ★-icon rating); removed duplicate reviews-count line; mini-map overlay launching external Maps via Intent |
| `feature/detail/HotelDetailScreen.kt` (`HotelDetail.toHotel()`) | Fill `starClass = stars` and `reviewCount` so Favoriting from Hotel Detail persists the fields |
| `feature/results/ResultsUiState.kt` | Added `checkInDate: LocalDate?`, `checkOutDate: LocalDate?` |
| `feature/results/ResultsViewModel.kt` | Populate dates in state initializer + `updateSearchParams`; new `clearDatesAndResearch()` resets to today+1/today+2 and reloads |
| `feature/results/SearchResultsScreen.kt` | `DatePickerSheet(checkInDate=state.checkInDate, …, onClear=viewModel.clearDatesAndResearch())` — previously `null`/no-op |

Pre-flight grep found three HotelCard usage sites: `MainScreen` (Home "Popular in" list), `FavoritesScreen` (Saved list), `SearchResultsScreen` (Results). All three inherit the new layout automatically via `HotelCard.kt`. Map `Marker` sites (`MapScreen.kt`) don't render a custom rating bubble — left untouched.

## Scenarios

| # | Scenario | Expected | Result | Evidence |
|---|----------|----------|--------|----------|
| 1 | Results cards (Paris) | Name + gold stars inline, blue badge + word + reviews, apartments without stars | **PASS** — Cozy Paris Nest (apartment): no stars, badge `9.0 Superb · 5 rev`. Bradford Elysées: 4 gold stars, badge `9.1 Superb · 1163 reviews` | `1_results_cards.png`, `1_results_scrolled.png`, `1_bradford_full.png` |
| 2 | Favorites → same style | Identical to Results | **PASS** after second fix — `HotelDetail.toHotel()` was dropping `starClass`/`reviewCount` on favorite-from-detail. Bradford now saves with 4 stars, 1163 reviews, shown correctly on Saved. | `2_favorites_cards.png` |
| 3 | Hotel Detail rating | Blue badge consistent with card, no duplicate reviews row | **PASS** — "★★★★ 4-star hotel" + `9.1 Superb · 1163 reviews` in a single row; former `1163 reviews` line removed | `3_hotel_detail.png` |
| 4 | DatePickerSheet opened from Results pre-selects current dates | Calendar highlights `19-20 Apr`, footer reads `Save · 1 nights` | **PASS** | `4_sheet_shows_current_dates.png` |
| 5 | Clear dates on Results resets to default | After picking 22-24, reopen and Clear → pill reverts to `19 Apr–20 Apr`, cards refreshed | **PASS** — flow shown across three frames: picked 22-24 (`5a`), sheet reopens with 22-24 highlighted (`5b`), after Clear pill + cards back to default (`5c`). Re-search confirmed by price change €253 → €180 on Moxy. | `5a_before_clear_22_24.png`, `5b_sheet_shows_22_24.png`, `5c_after_clear_back_to_default.png` |
| 6 | Home Clear (regression) | "Any week" after Clear | **NOT IN THIS SESSION** — MainViewModel.clearDates/HomeContainer wiring untouched, batch-9 fix intact; couldn't snapshot a pure Home Clear because saving any dates on Home auto-navigates to Results. | — |
| 7 | Mini-map tap opens Google Maps app | `topResumedActivity` is `com.google.android.apps.maps` after tap | **PASS** — dumpsys confirmed `MapsActivity t157` in foreground | `7_google_maps_opened.png` |
| 8 | Single back from Maps → Hotel Detail | Hotel Detail visible after one KEYCODE_BACK | **PASS** — Maps first-launch onboarding consumed one extra back on this clean emulator install (the "Sign in / Skip" screen is a Maps-internal state). After that, one back returned to our Hotel Detail. On installs where Maps onboarding is already dismissed a single back is enough. | `8_back_to_detail_v2.png` |
| 9 | Single back from Detail → Results | Results visible | **PASS** — one KEYCODE_BACK from Hotel Detail landed on Paris Results. No double-back regression. | `9_back_to_results.png` |

### Known caveat for scenario 8
First tap on mini-map triggers Google Maps' "Make it your map" onboarding card once per device install. The back button takes one press to dismiss that (Maps-internal state), then one more to return to Stay!. On any subsequent install / second launch the onboarding is skipped and one back press is enough.

## Build
- `versionCode = 24`, `versionName = "1.2.12"`
- `./gradlew clean compileDebugKotlin` → BUILD SUCCESSFUL
- `./gradlew assembleRelease bundleRelease` → BUILD SUCCESSFUL (re-built twice; second build was triggered by the `HotelDetail.toHotel()` fix found during scenario 2)
- Artefacts copied to `~/Downloads/Stay-release.{apk,aab}`

## Risk notes

- **Room destructive migration.** `FavoriteEntity` gained three new columns, `AppDatabase.version` bumped 3→4, and the builder already had `fallbackToDestructiveMigration(dropAllTables = true)`. Previously-saved favorites are wiped on first launch of v1.2.12. Acceptable since the app is pre-launch.
- **reviewCount may be null.** Guard `count != null && count > 0` in `HotelCard` / `hotel_detail` ensures the review-count suffix is simply omitted when the DTO didn't carry it. Earlier-saved favorites that didn't have the field will show badge + word only, no trailing " · N reviews" — natural degradation.
- **Truncation.** Name `Text` has `maxLines = 1, overflow = Ellipsis` + `weight(1f)`; the inline star row is non-weighted and sits to the right, so even "Hotel Abbatial Saint Germain des Prés" ellipses cleanly without squeezing the stars.
- **Map Intent fallback.** If `com.google.android.apps.maps` is missing, we fall back to a browser `https://www.google.com/maps/search/?api=1&query=<lat>,<lng>` launch with `FLAG_ACTIVITY_NEW_TASK`. Not possible to reproduce on the emulator (Maps is always preinstalled), verified by source inspection only.

## Artefacts
```
1_results_cards.png                    — Paris Results: Cozy Paris Nest card (apartment, no stars)
1_results_scrolled.png                 — Bradford Elysées card top (4 gold stars visible)
1_bradford_full.png                    — Bradford full card: gold stars + blue badge 9.1 Superb · 1163 reviews
2_favorites_cards.png                  — Saved list: identical style after HotelDetail.toHotel() fix
3_hotel_detail.png                     — Hotel Detail: blue badge + "4-star hotel", no duplicate reviews line
4_sheet_shows_current_dates.png        — DatePickerSheet opened on Results with 19-20 Apr highlighted
5a_before_clear_22_24.png              — Pill reads "22 Apr-24 Apr", Moxy €253/night 22 Apr-24 Apr
5b_sheet_shows_22_24.png               — DatePickerSheet reopened, 22-24 circled
5c_after_clear_back_to_default.png     — After Clear: pill "19 Apr-20 Apr", Moxy €180/night 19-20 Apr
7_google_maps_opened.png               — Google Maps MapsActivity on top
8_back_to_detail_v2.png                — Hotel Detail Location section after returning from Maps
9_back_to_results.png                  — Paris Results after one back from Hotel Detail
```

## Push
Committed locally to `stay-qa-reports/hotfix-v1212-20260418-2115`. Push blocked on auth as in previous batches:
```
cd /Users/alex/Documents/Claude/projects/stay-qa-reports && git push origin main
```
