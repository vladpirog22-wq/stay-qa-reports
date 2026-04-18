# Batch 10 Diagnostics Report — 2026-04-18 13:38 local

**Build:** v1.2.9 release APK with additional `Log.d` at each handoff + `HttpLoggingInterceptor.Level.BODY` routed to tag `HttpFilter`. **No code fixes applied**; only instrumentation.

Device: emulator-5554 · API 36 · 1080×1920.

All diagnostic log lines and screenshots are in this folder.

---

## 10-1 Clear dates

### Scenario
Home → SearchPill (Any week) → DatePickerSheet → pick 22 & 24 Apr → Save → SearchPill reopen → tap Clear.

### Visual result
After Clear: SearchPill reads **"Any week"** (default). Sheet auto-dismisses. See `10_1_after_clear.png`.

### clear.log chain (complete, no breaks)
```
SearchPill recompose: state.checkInDate=null state.checkOutDate=null finalText='Any week'   (before save)
DatePickerSheet Save tapped: startDate=2026-04-22 endDate=2026-04-24 -> calling onSave
HomeContainer onSave received: checkIn=2026-04-22 checkOut=2026-04-24
VM.setDates called ci=2026-04-22 co=2026-04-24. BEFORE: state.ci=null state.co=null
VM.setDates AFTER: state.ci=2026-04-22 state.co=2026-04-24 sheetOpen=false
SearchPill recompose: state.checkInDate=2026-04-22 state.checkOutDate=2026-04-24 finalText='22 Apr – 24 Apr (2 nights)'
=== DatePickerSheet Clear tapped === local.startDate=2026-04-22 local.endDate=2026-04-24
DatePickerSheet calling parent onClear()
HomeContainer onClear received -> vm.clearDates()
VM.clearDates called. BEFORE: state.ci=2026-04-22 state.co=2026-04-24
VM.clearDates AFTER: state.ci=null state.co=null sheetOpen=false
SearchPill recompose: state.checkInDate=null state.checkOutDate=null finalText='Any week'
```

### Hypothesis
**10-1 is NOT a live bug on v1.2.9.** The batch-9 fix (adding `isDateSheetOpen = false` to `clearDates()`) is working end-to-end. The chain Sheet → HomeContainer → VM → state → SearchPill recompose is intact; SearchPill correctly falls back to "Any week" fewer than 10 ms after the Clear tap. If the user is still seeing stale dates, it is almost certainly on a pre-1.2.9 install — recommend `adb install -r ~/Downloads/Stay-release.apk` to confirm they're on vc=20.

Same shape expected for GuestsSheet Reset (not re-captured here; the code path is symmetric).

Artifacts: `clear.log`, `10_1_home_initial.png`, `10_1_date_sheet_open.png`, `10_1_dates_picked.png`, `10_1_dates_saved.png`, `10_1_date_sheet_reopened.png`, `10_1_after_clear.png`.

---

## 10-2 Stars in Search results

### Scenario
Home → SearchPill → Paris → auto-search → Results list → scroll → open a hotel card.

### Visual result
- **Results cards DO NOT render stars**, even for hotels where the DTO clearly carries `accuratePropertyClass=4` or `=3`. Only a `★ 9.00` style *review* chip is shown in the top-right. See `10_2_paris_results.png`.
- **Hotel Detail page DOES render stars** ("★★★★ 4-star hotel" for Bradford Elysées – Astotel). This is the batch-9 nav-arg fallback working. See `10_2_hotel_detail.png`.

### stars.log — key findings (20 hotels mapped)
- Every hotel in the Paris response populates at least one star field. Representative samples:
  ```
  Cozy Paris Nest            p.accuratePropertyClass=0  p.propertyClass=0  p.reviewScore=9.0
  Bradford Elysées - Astotel p.accuratePropertyClass=4  p.propertyClass=4  p.reviewScore=9.1
  Moxy Paris Bastille        p.accuratePropertyClass=3  p.propertyClass=3  p.reviewScore=8.0
  Hôtel Botaniste            p.accuratePropertyClass=4  p.propertyClass=4  p.reviewScore=8.5
  Hotel Meslay Republique    p.accuratePropertyClass=3  p.propertyClass=3  p.reviewScore=8.3
  Hilton Paris Opera         p.accuratePropertyClass=4  p.propertyClass=4  p.reviewScore=7.6
  Hotel Abbatial Saint Germain p.accuratePropertyClass=3  p.propertyClass=3  p.reviewScore=8.7
  Pullman Paris Tour Eiffel  p.accuratePropertyClass=4  p.propertyClass=4  p.reviewScore=8.2
  ```
- `HotelDto` declared fields: `accessibilityLabel, hotelId, property`.
- `PropertyDto` declared fields: `accuratePropertyClass, checkinDate, checkoutDate, currency, id, latitude, longitude, name, photoUrls, priceBreakdown, propertyClass, reviewCount, reviewScore, reviewScoreWord, wishlistName`.
- `HotelMapper.HotelDto.toDomain()` writes `starClass = p.accuratePropertyClass ?: p.propertyClass` — the domain value is already correct.

### Hypothesis
**Data is present; the `HotelCard` composable just does not render stars.** The search DTO already carries `accuratePropertyClass` reliably (no null cases observed in 20 hotels), `HotelMapper` already passes it through to `Hotel.starClass`. The gap is purely in the UI layer on the results list screen.

The fix should be limited to `HotelCard.kt` (or wherever the results-list card is composed): add a row of gold `Icons.Filled.Star` icons (size 12-14.sp, tint `0xFFF5A623`) next to the hotel name when `hotel.starClass in 1..5`. Hotels with `starClass == 0` (apartments/hostels — e.g. Cozy Paris Nest) should render without stars — that's the current behaviour on Hotel Detail and is correct.

Artifacts: `stars.log` (101 lines, 20 hotels × 5 log lines each), `10_2_paris_results.png` (no stars), `10_2_results_scrolled.png` (no stars), `10_2_hotel_detail.png` (stars present = batch-9 fix).

---

## 10-3 Server-side filter

### Scenario
Results for Paris (20 hotels loaded) → Filters sheet → tap `5★` + `9+` → Show N stays → Apply.

### Visual result
- Filter sheet preview: **"Show 0 stays"** (client-side preview over the 20 already-loaded).
- After Apply: fresh API call fires, EmptyState `No stays match your filters` appears. See `10_3_results_filtered.png`.

### HTTP evidence (from `http.log`)
**Outgoing URL** — the filter params ARE being sent:
```
GET https://hotelfinder-api.vladpirog22.workers.dev/v1/searchHotels
    ?dest_id=-1456928
    &search_type=CITY
    &arrival_date=2026-04-19
    &departure_date=2026-04-20
    &adults=2
    &room_qty=1
    &page_number=1
    &currency_code=EUR
    &languagecode=en-gb
    &categories_filter_ids=class%3A%3A5          ← class::5, URL-encoded
    &review_score_filter_min=9.0                 ← 9.0 threshold
```

**Response body** (visible snippet — first 3 hotels of 20+ returned):
```
Cozy Paris Nest            accuratePropertyClass=0   reviewScore=9.0    ← 0★, passes 9+
Bradford Elysées - Astotel accuratePropertyClass=4   reviewScore=9.1    ← 4★ (NOT 5), passes 9+
Moxy Paris Bastille        accuratePropertyClass=3   reviewScore=8.0    ← 3★ (NOT 5), 8.0 < 9
```

`accuratePropertyClass` distribution across the captured subset: `{0:1, 3:1, 4:1}` — **zero 5★ hotels in the returned list, and scores below 9.0 still present.**

### Hypothesis — the upstream IGNORES both filter params

Proof:
1. The URL unambiguously contains `categories_filter_ids=class::5` and `review_score_filter_min=9.0`.
2. The response body contains hotels that violate both filters (class 0/3/4 and score 8.0).
3. Response size: `34735-byte body`, identical in shape to the unfiltered Paris query.

Why "No stays" then shows:
- The client-side safety net `passesClientFilter` in `ResultsViewModel` (left in place after batch-9 as a fallback) filters the 20-odd returned hotels down to zero, because none of them match `starClass == 5 AND rating >= 9`. Without that safety net the app would display all 20 hotels as if no filter were applied.

Two possible root causes for upstream ignoring the params:
- **(A)** RapidAPI `booking-com15` doesn't accept `categories_filter_ids` / `review_score_filter_min` on the current (PRO?) tier. Parameter names may differ (`classes`, `hotel_class_filter`, `review_score_filter`, or a different JSON shape).
- **(B)** The user's Cloudflare Worker proxy (`hotelfinder-api.vladpirog22.workers.dev`) has an allowlist of forwarded query params and strips any it doesn't explicitly pass through. Worth inspecting `wrangler tail` while reproducing.

### Follow-ups required
1. Inspect the Worker source: does it forward `categories_filter_ids` / `review_score_filter_min` to the RapidAPI origin, or strip them?
2. If the Worker forwards them, hit the RapidAPI origin directly (curl + RapidAPI key) to confirm which param name the upstream accepts for star filtering on this plan.
3. Once param naming is confirmed, drop `passesClientFilter` from `ResultsViewModel` for star + guest-rating (keep it only as a soft fallback for future fields).

Artifacts: `http.log` (outgoing URL + response headers + body snippet), `10_3_urls.txt` (request URL only), `10_3_filter_sheet_5star_9plus.png` (chips highlighted), `10_3_results_filtered.png` (EmptyState after Apply).

---

## Summary table

| # | Root cause (short) | Code area |
|---|--------------------|-----------|
| 10-1 | Not a live bug on v1.2.9 — batch-9 fix is observable end-to-end | n/a, tell user to reinstall |
| 10-2 | Data flows correctly to `Hotel.starClass`; `HotelCard` composable in Results list simply doesn't render stars | `feature/home/components/HotelCard.kt` (or similar) |
| 10-3 | App sends `categories_filter_ids` + `review_score_filter_min` correctly; **upstream (Worker or RapidAPI) ignores them and returns the full list**. Client-side safety net then filters to zero | Worker config OR `BookingApiService.kt` param naming |

---

## Artefacts in this folder
```
DIAGNOSTICS_REPORT.md                     — this file
clear.log                                 — ClearDebug log chain
stars.log                                 — StarsMapping log (20 hotels)
http.log                                  — HttpFilter log (outgoing URL + response)
10_3_urls.txt                             — extracted request URL only

10_1_home_initial.png                     — Home before Clear scenario
10_1_date_sheet_open.png                  — DatePickerSheet empty
10_1_dates_picked.png                     — 22 & 24 Apr selected
10_1_dates_saved.png                      — pill reads "22 Apr – 24 Apr (2 nights)"
10_1_date_sheet_reopened.png              — sheet re-opened after save
10_1_after_clear.png                      — pill reverted to "Any week" ← fix confirmed

10_2_dest_search.png                      — Paris typed into search
10_2_paris_results.png                    — Results list (no stars on cards)
10_2_results_scrolled.png                 — scrolled, still no stars
10_2_hotel_detail.png                     — Detail shows ★★★★ 4-star hotel

10_3_filter_sheet_5star_9plus.png         — 5★ + 9+ chips selected, "Show 0 stays"
10_3_results_filtered.png                 — "No stays match your filters" EmptyState
```

## Build note
This APK has both `HttpLoggingInterceptor.Level.BODY` and the `StarsMapping`/`ClearDebug` logs enabled **in release**. Must be reverted before the next production release — logs leak HMAC request signatures, Worker URL, and full response bodies.
