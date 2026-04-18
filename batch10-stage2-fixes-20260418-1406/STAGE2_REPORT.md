# Batch 10 Stage 2 Report — 2026-04-18 14:06 local

**Build:** v1.2.10 / versionCode 22 (release). APK/AAB at `~/Downloads/Stay-release.{apk,aab}`.

Follow-up to Stage 1 diagnostics. Three items + cleanup.

---

## 10-1 Clear dates — no code change

Stage 1 `clear.log` confirmed the full chain (DatePickerSheet → HomeContainer → VM.clearDates → state → SearchPill recompose) is intact on v1.2.9. Re-verified on the clean v1.2.10 build: Clear dismisses the sheet and SearchPill reverts to "Any week" immediately. If the user still reports stale dates it's almost certainly a stale install — they need `adb install -r ~/Downloads/Stay-release.apk` or a Play update.

Status: **NOT A BUG**.

---

## 10-2 Stars on Results cards — FIXED

### Root cause (from Stage 1)
`Hotel.starClass` was already populated correctly from `PropertyDto.accuratePropertyClass ?: propertyClass`. The `HotelCard` composable simply never rendered it.

### Fix
[`app/src/main/java/com/hotelfinder/app/ui/components/HotelCard.kt`](../../hotelfinder/HotelFinder/app/src/main/java/com/hotelfinder/app/ui/components/HotelCard.kt) — added a star row right below the name + review chip row:

```kotlin
val stars = hotel.starClass
if (stars != null && stars in 1..5) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        repeat(stars) {
            Icon(
                imageVector = Icons.Filled.Star,
                contentDescription = null,
                tint = Color(0xFFF5A623),
                modifier = Modifier.size(12.dp),
            )
        }
    }
}
```

`starClass in 1..5` guard keeps class 0 (apartments / unclassified — Stage 1 observed `Cozy Paris Nest` coming through as 0) from rendering an empty row.

### Proof
- `s2_2_results_with_stars.png` — "Hôtel Mercure Paris Boulogne Pont de Saint Cloud" shows **4 gold stars** under the name.
- `s2_2_results_scrolled.png` — "ibis Paris Boulogne-Billancourt" shows **3 gold stars**.
- Apartments without a class render without the row (no empty whitespace regression visible between name and location).

---

## 10-3 Server-side star/rating filter — NOT HONOURED UPSTREAM (definitive)

### Investigation

#### Worker (`hotelfinder-api.vladpirog22.workers.dev`)
Source at `hotelfinder/hotelfinder-worker-spec/worker.js`. The Worker forwards query string **verbatim**:

```javascript
const upstreamUrl = `https://${env.RAPIDAPI_HOST}${endpoint.upstream}${url.search}`;
```

No allowlist strips filter params. Worker is **innocent**.

#### Direct RapidAPI `/api/v1/hotels/searchHotels` — baseline
```bash
GET booking-com15.p.rapidapi.com/api/v1/hotels/searchHotels
    ?dest_id=-1456928&…&categories_filter_ids=class::5&review_score_filter_min=9.0
```
**Response: 20 hotels. Classes: 0, 4, 3, 4, 3, 4, 3, 4, … — scores 8.0 through 9.1.**
Identical distribution to the unfiltered call. Filter params **silently ignored**.

#### Alternative param names tested (all ignored)
| Param name tried | Distribution of first 10 `accuratePropertyClass` |
|------------------|--------------------------------------------------|
| `categories_filter_ids=class::5`                       | `[0, 4, 3, 4, 3, 4, 4, 3, 3, 3]` |
| `categories_filter_ids=class::5,reviewscorebuckets::90` | `[0, 4, 3, 4, 3, 4, 3, 4, 3, 3]` |
| `filters=class::5;reviewscorebuckets::90`              | `[0, 4, 3, 4, 3, 4, 3, 4, 3, 3]` |
| `filter_option=class::5`                               | `[0, 4, 3, 4, 3, 4, 4, 3, 3, 3]` |
| `category_filters=class::5`                            | `[0, 4, 3, 4, 3, 4, 3, 4, 3, 3]` |
| `filter_by_id=class::5`                                | `[0, 4, 3, 4, 3, 4, 4, 3, 3, 4]` |
| `categories=class::5`                                  | `[0, 4, 3, 4, 3, 4, 3, 4, 3, 3]` |
| `selected_filter=class::5`                             | `[0, 4, 3, 4, 3, 4, 4, 3, 3, 4]` |
| `filter=class::5`                                      | `[0, 4, 3, 4, 3, 4, 4, 3, 3, 3]` |

Same results regardless of param name. The companion `/api/v1/hotels/getFilter` endpoint **does** return a full filter catalogue (with token format `class::5`, `reviewscorebuckets::90`, `facility::107` etc. for the filter sheet UI), but `/searchHotels` doesn't accept selections back.

### Conclusion
**`booking-com15` PRO tier's `/api/v1/hotels/searchHotels` does not honour any server-side star-class or review-score filter.** This is a platform limitation, not a code bug.

### Action taken
- Removed the three dead query parameters (`categories_filter_ids`, `review_score_filter_min`) from [`BookingApiService.kt`](../../hotelfinder/HotelFinder/app/src/main/java/com/hotelfinder/app/data/remote/api/BookingApiService.kt) — they polluted the Worker KV cache key per-filter-combo with no benefit.
- Removed the matching `starClasses` / `minReviewScore` params from [`HotelRepository`](../../hotelfinder/HotelFinder/app/src/main/java/com/hotelfinder/app/domain/repository/HotelRepository.kt) / [`HotelRepositoryImpl`](../../hotelfinder/HotelFinder/app/src/main/java/com/hotelfinder/app/data/repository/HotelRepositoryImpl.kt) / [`ResultsViewModel`](../../hotelfinder/HotelFinder/app/src/main/java/com/hotelfinder/app/feature/results/ResultsViewModel.kt). Client-side `passesClientFilter` is now the sole path and is now commented as such.
- Changed the filter CTA from "Show N stays" to **"Show results"** in [`FilterSheet.kt`](../../hotelfinder/HotelFinder/app/src/main/java/com/hotelfinder/app/feature/results/FilterSheet.kt) + [`strings.xml`](../../hotelfinder/HotelFinder/app/src/main/res/values/strings.xml). The client-side count misled users once filters started hitting zero-result pools. Screenshot `s2_3_filter_sheet_show_results.png`.

### Status
**PARTIAL**. The bug the user reported (5★+9+ → 0 stays) is mitigated by the honest CTA. To actually yield non-empty 5★+9+ results a higher page_size or aggressive auto-pagination is needed — deferred, flagged as next-session work.

### Next-session options (not attempted this batch)
- **A — bigger page.** RapidAPI searchHotels has no `page_size` param (20 is fixed). `searchHotelsByCoordinates` may accept different page sizing — not tested.
- **B — aggressive auto-pagination.** On 0-result after client filter, auto-fetch page 2-3 in the background until N≥3 match, showing a "Searching more hotels…" hint.
- **C — plan upgrade.** booking-com15 ULTRA or ENTERPRISE tier may expose filter params. Not verified.

---

## Cleanup

Reverted Stage 1 instrumentation:

| File | Change |
|------|--------|
| `NetworkModule.kt` | `HttpLoggingInterceptor.Level.BODY` → `BASIC` on DEBUG, `NONE` on release. Removed custom `HttpFilter` tag. |
| `HotelMapper.kt` | Removed `StarsMapping` diagnostic block; mapping logic unchanged. |
| `DatePickerSheet.kt` | Removed `ClearDebug` logs from onClear/onSave. |
| `HomeContainer.kt` | Removed `ClearDebug` logs from the three sheet callbacks and the SearchPill recompose tracer. |
| `MainViewModel.kt` | Removed `ClearDebug` logs from `setDates` / `clearDates`. |
| `HotelDetailViewModel.kt` | Removed `StarsDebug` log (left over from batch 9). |

Verified post-cleanup:
```
$ grep -rn "ClearDebug|StarsMapping|StarsDebug|HttpFilter" app/src/main/java
(no matches)
$ grep -rn "HttpLoggingInterceptor.Level.BODY" app/src/main/java
(no matches)
```

---

## Build
- `versionCode = 22`, `versionName = "1.2.10"`
- `./gradlew clean compileDebugKotlin` → BUILD SUCCESSFUL
- `./gradlew assembleRelease bundleRelease` → BUILD SUCCESSFUL
- APK/AAB copied to `~/Downloads/Stay-release.{apk,aab}`
- Installed on emulator-5554, verified Search Paris flow.

---

## Artefacts
```
10_1_v1210_clear_works.png              — Clear dates flow verified on v1.2.10 ("Any week" after tap)
s2_2_results_with_stars.png             — 4★ Mercure on Results list (10-2 proof)
s2_2_results_scrolled.png               — 3★ ibis on Results list (10-2 proof)
s2_3_filter_sheet_show_results.png      — filter CTA reads "Show results" (10-3 action)
10_3_filter_applied.png                 — 5★+9+ applied → EmptyState (client-side collapse, server ignored)
10_3_direct_categories_filter_ids.json  — raw RapidAPI body with categories_filter_ids=class::5 (34690 bytes, mixed classes in hotels)
10_3_direct_hotel_class.json            — raw RapidAPI body with hotel_class=5 (34690 bytes, mixed classes too)
```

## Push note
Committed locally to `stay-qa-reports` (commit will be `origin/main` + 1). Push requires user — Stage 1's auth issue still applies:
```bash
cd /Users/alex/Documents/Claude/projects/stay-qa-reports
git push origin main
```
