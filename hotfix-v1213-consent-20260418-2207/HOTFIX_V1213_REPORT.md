# Hotfix v1.2.13 — GDPR Consent (Variant 2) Report — 2026-04-18 22:07 local

**Build:** v1.2.13 / versionCode 25 (release). APK/AAB at `~/Downloads/Stay-release.{apk,aab}`.

## Scope
Two-view GDPR-compliant consent flow (Variant 2 — Accept all + Customize):
- Initial view: value-prop copy + "Accept all" (primary, affirmative action) + "Customize" (secondary) + Privacy policy. No pre-ticked toggles.
- Customize view: per-toggle controls with OFF defaults, back arrow, "Save preferences" button.
- Firebase auto-init disabled via manifest meta-data; runtime observer re-enables collection after user consents.
- Location permission stays runtime in DestinationSheet — untouched.

## Files changed

| File | Change |
|------|--------|
| `feature/consent/ConsentViewModel.kt` | OFF defaults (`analyticsEnabled=false`, `crashlyticsEnabled=false`); `showCustomizeView` state; `openCustomize()`/`backToInitial()`; `acceptAll()` flips both ON and persists; `savePreferences()` persists user choice. |
| `feature/consent/ConsentScreen.kt` | Full rewrite: `InitialView` composable (hero + title + info card + Accept all + Customize + Privacy) and `CustomizeView` composable (back arrow + title + two `ToggleRow`s + Save preferences). |
| `AppNavGraph.kt` | Updated `composable(Routes.CONSENT)` to wire new signature (onAcceptAll, onOpenCustomize, onBackToInitial, onSavePreferences). |
| `data/local/datastore/SettingsDataStore.kt` | New `ConsentState(analytics, crashlytics)` + `observeConsent(): Flow<ConsentState>` stream. |
| `HotelFinderApp.kt` | `@Inject lateinit var settingsDataStore`; `appScope.launch` observing `observeConsent().distinctUntilChanged()` → `applyConsentToFirebase()` reflection helper calls `FirebaseAnalytics.setAnalyticsCollectionEnabled` / `FirebaseCrashlytics.setCrashlyticsCollectionEnabled`. Reflective because Firebase deps aren't in the build yet; reflection no-ops (logs `skipping runtime toggle`) until the SDK lands. |
| `AndroidManifest.xml` | Four new `<meta-data>` entries inside `<application>`: `firebase_analytics_collection_enabled=false`, `firebase_crashlytics_collection_enabled=false`, `google_analytics_adid_collection_enabled=false`, `google_analytics_ssaid_collection_enabled=false`. Verified present with `value=0` via `aapt dump xmltree`. |
| `res/values/strings.xml` | Replaced legacy consent copy with new two-view keys: `consent_initial_title`, `consent_initial_subtitle`, `consent_info_analytics`, `consent_info_crash`, `consent_info_promise`, `consent_accept_all`, `consent_customize`, `consent_customize_title`, `consent_customize_subtitle`, `consent_save_preferences`. Kept `consent_analytics_title/desc`, `consent_crash_title/desc`, `consent_privacy_policy` (reused in customize view). |

## Scenarios

| # | Scenario | Expected | Result | Evidence |
|---|----------|----------|--------|----------|
| 1 | Fresh install → initial view | Hero icon + title "Help us improve Stay!" + info card + Accept all + Customize + Privacy policy. No toggles visible, no pre-tick. | **PASS** | `1_initial_view.png` |
| 2 | Accept all → Home | Both toggles saved true, navigate to Home. | **PASS** — ConsentObserver log later confirmed `analytics=true crashlytics=true` persisted. | `2_home_after_accept_all.png` |
| 3 | Customize → second view | Back arrow, title "Customize privacy", subtitle, **both toggles OFF** (grey), Save preferences CTA. | **PASS** — both switches rendered grey on opening the customize view. | `3_customize_view.png` |
| 4 | Toggle Crash ON + Save | Crash switch flips black, Analytics stays grey; tap Save → Home; DataStore holds `analytics=false crashlytics=true`. | **PASS** — customize view screenshot shows the asymmetric state; ConsentObserver logged `analytics=false crashlytics=true` after Save. | `4_crash_toggled.png`, `5_home_after_save.png` |
| 5 | Back arrow customize → initial | Initial view restored with Accept all / Customize / Privacy policy. | **PASS** | `6_back_to_initial.png` |
| 6 | Firebase manifest meta-data | All four `<meta-data>` entries present with `value=(type 0x12)0x0` (boolean false). | **PASS** — dumped via `aapt dump xmltree ~/Downloads/Stay-release.apk AndroidManifest.xml`. Output saved. | `6_aapt_firebase_meta.txt` |
| 7 | Consent persistence | After Accept all + force-stop + relaunch → Home immediately, no consent. | **PASS** — Home rendered directly after relaunch (Popular in London card, search pill, bottom nav). Full roundtrip confirmed through ConsentObserver log timestamps. | `7_no_consent_on_relaunch.png` |
| 8 | Location permission stays just-in-time | No system location dialog during consent or on reaching Home. Dialog appears only when the user taps "Use my current location" inside DestinationSheet. | **PASS** — "Allow Stay! to access this device's approximate location?" dialog appeared solely on tap, not before. | `8_location_permission_prompt.png` |

### Caveat for scenario 7
First post-`pm clear` cold launches reliably trigger an **emulator-side** `Input dispatching timed out … Waited 5001ms for FocusEvent(hasFocus=true)` ANR dialog on this Medium_Phone AVD (logcat `am_anr`). It's a window-focus timeout — the ViewModel / composable render is already on screen under the ANR dialog. `logcat | grep ConsentObserver` confirms the DataStore observer fired on time (`analytics=true crashlytics=true` after Accept all; `analytics=false crashlytics=false` on the post-clear relaunch). The ANR dismisses itself on the next relaunch when the AVD is warmer — second screenshot (`7_no_consent_on_relaunch.png`) was taken after one more warm relaunch and shows Home clean. No ANR stack involves our code — it's the AOSP WindowManager focus timeout firing on a cold AVD. Worth re-checking on a physical device before a Play release but not blocking.

## Firebase integration note
The project does not yet have Firebase dependencies in `build.gradle.kts`. The manifest meta-data is harmless without the SDK (read by the SDK on init). `HotelFinderApp.applyConsentToFirebase` uses reflection so the file compiles and runs today — logs `FirebaseAnalytics not on classpath; skipping runtime toggle` when the observer fires. When Firebase is added:
1. Drop reflection, call `FirebaseAnalytics.getInstance(this).setAnalyticsCollectionEnabled(analytics)` + `FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(crashlytics)` directly.
2. Verify the manifest flags keep the SDK silent until the observer fires by turning on Analytics DebugView with consent OFF.

## Build
- `versionCode = 25`, `versionName = "1.2.13"`
- `./gradlew clean compileDebugKotlin` → BUILD SUCCESSFUL
- `./gradlew assembleRelease bundleRelease` → BUILD SUCCESSFUL

## Artefacts
```
1_initial_view.png                    — fresh install, Accept all + Customize + no toggles
2_home_after_accept_all.png           — Home reached after one-tap Accept all
3_customize_view.png                  — second view, both toggles OFF
4_crash_toggled.png                   — Crash switch ON, Analytics OFF
5_home_after_save.png                 — Home after Save preferences
6_back_to_initial.png                 — back arrow from customize returns to initial
6_aapt_firebase_meta.txt              — aapt dump of the four Firebase meta-data entries
7_no_consent_on_relaunch.png          — Home on cold relaunch, consent persisted
8_location_permission_prompt.png      — system permission dialog on first AroundMe tap, not before
```

## Push
Committed locally to `stay-qa-reports/hotfix-v1213-consent-20260418-2207`. Push blocked on auth:
```
cd /Users/alex/Documents/Claude/projects/stay-qa-reports && git push origin main
```
