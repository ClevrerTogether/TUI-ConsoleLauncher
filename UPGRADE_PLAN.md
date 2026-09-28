# Goal: Modernize TUI-ConsoleLauncher for Latest Android & Google Play

## Objective

Bring this app up to date with current Android security/version requirements,
verify it works via a locally-built/tested APK, and eventually publish it to
the Google Play Store.

## Approach

Step by step, in small verifiable increments:

1. Apply security updates and target/compile SDK bumps.
2. Build an APK and manually test on-device/emulator after each meaningful
   change.
3. Once stable, prepare and go through Google Play Store submission
   requirements (Play Console listing, privacy policy, data safety form,
   target API level policy, signing, etc.).

## Current baseline (as of 2026-09-28)

- `compileSdk 34`, `targetSdk 34`, `minSdk 21` (`app/build.gradle`)
- versionName `v6.15.1-updated-v2`, versionCode `301`
- Flavors: `fdroid`, `playstore`
- Recent related commits already on `master`:
  - `46569e5` Modernization and Security Hardening for Android 14 (API 34)
  - `16e4a57` fix: resolve location crash and implement on-demand permissions (#408)
  - `470c606` fix: improve 'uninstall' command reliability and manifest permissions (#409)
  - `058f541` feat: implement dynamic transparency for theme presets (#410)

## Working agreement

- Go step by step — don't batch large unrelated changes into one pass.
- Test via a built APK before moving to the next step.
- Play Store submission work comes after the app is verified stable via APK
  testing, not in parallel.

## Log

- 2026-09-28: Created this tracking file. Reviewed current build config
  (already targets API 34). Next: identify and prioritize remaining
  security/compatibility issues.
