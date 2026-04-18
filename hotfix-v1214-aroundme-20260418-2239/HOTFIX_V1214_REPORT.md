# Hotfix v1.2.14 — Booking-style AroundMe row Report — 2026-04-18 22:39 local

**Build:** v1.2.14 / versionCode 26 (release). APK/AAB at `~/Downloads/Stay-release.{apk,aab}`.

## Scope
Minimalist redesign of the "Use my current location" row inside `DestinationSheet` to match the Booking.com mobile layout the user referenced:

- Removed the 44 dp `AccentRose` circle + `Icons.Outlined.MyLocation` crosshair glyph.
- Removed the "Barcelona, Spain"-style subtitle line under the label.
- Switched to outlined paper-plane `Icons.Outlined.Navigation` (22 dp), tinted `TextPrimary`.
- Dropped the "AROUND ME" `SectionTitle` header so the row sits flush under the search input.
- Permission flow (runtime Android dialog via `onUseCurrentLocation` → `rememberLauncherForActivityResult` in `HomeContainer`) is **unchanged**.

## Files changed

| File | Change |
|------|--------|
| `feature/home/sheets/DestinationSheet.kt` | `import …outlined.MyLocation` → `outlined.Navigation`. `AroundMeRow` rewritten: bare `Icon` + `Spacer(Space.l)` + single-line `Text` with `FontWeight.SemiBold`. `subtitle` parameter kept for signature compat and marked `@Suppress("UNUSED_PARAMETER")`. Removed the `item { SectionTitle(destination_section_around_me) }` that sat above the row. |

Preview at the bottom of the file (which still passes `subtitle = "Madrid, Spain"`) compiles unchanged — the parameter just gets ignored by the composable.

## Scenarios

| # | Scenario | Expected | Result | Evidence |
|---|----------|----------|--------|----------|
| 1 | Visual of the sheet on empty query | Outlined paper-plane + "Use my current location" label, no coloured chip, no subtitle, no "AROUND ME" header | **PASS** | `1_new_around_me_row.png` |
| 2 | Permission dialog still fires | System "Allow Stay! to access this device's approximate location?" appears on tap | **PASS** — runtime dialog identical to v1.2.13 scenario 8 | `2_permission_dialog.png` |
| 3 | Grant → graceful behaviour on emulator without GPS fix | Sheet remains (emulator has no reverse-geocodable location), no crash | **PASS with caveat** — on a real device the flow auto-closes the sheet and pushes Results. `LocationHelper.getFreshLocation()` / `reverseGeocode()` return null on this AVD, so `MainViewModel.selectCurrentLocationAsDestination()` logs `no destination found for '…'` and returns early. No crash, no regression. | `3_after_grant.png` |
| 4 | Deny → sheet stays open, user can type | Sheet is still there with Search input focusable; no crash | **PASS** | `4_after_deny.png` |

## Build
- `versionCode = 26`, `versionName = "1.2.14"`
- `./gradlew assembleRelease bundleRelease` → BUILD SUCCESSFUL

## Risk notes
- `subtitle` parameter intentionally kept. Call sites and the preview still pass a value (`aroundMeSubtitle` / `"Madrid, Spain"`); the composable just ignores it. Cleanup to remove the arg entirely is a separate refactor.
- `Icons.Outlined.Navigation` is not auto-mirrored. No RTL locale is shipped today — revisit when Arabic/Hebrew is added.

## Artefacts
```
1_new_around_me_row.png       — Booking-style minimal row inside DestinationSheet
2_permission_dialog.png       — system runtime permission dialog on tap
3_after_grant.png             — sheet after grant on emulator (no GPS fix → graceful no-op)
4_after_deny.png              — sheet after deny (stays open for manual search)
```

## Push
Committed locally to `stay-qa-reports/hotfix-v1214-aroundme-20260418-2239`. Push blocked on auth:
```
cd /Users/alex/Documents/Claude/projects/stay-qa-reports && git push origin main
```
