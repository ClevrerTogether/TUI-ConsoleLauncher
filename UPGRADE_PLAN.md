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

## Environment notes

- This sandbox mounts the repo via virtiofs (`\wsl.localhost\...`), which
  has two quirks worth remembering:
  - `chmod`/`fchmod` fail with I/O errors here, which breaks `git config`
    writes and the Edit tool'''s rename step. Workarounds: pass git identity
    via `GIT_AUTHOR_*`/`GIT_COMMITTER_*` env vars per-commit, use
    `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_*`/`GIT_CONFIG_VALUE_*` env vars for
    persistent git config (set in `/etc/sandbox-persistent.sh`), and write
    file edits via a small Python script instead of the Edit tool.
  - Deleting non-empty directories can fail with I/O errors too, which
    breaks Gradle'''s own clean/rebuild steps under `app/build`. Workaround:
    build from a copy of the repo under `/tmp` (overlay fs, not virtiofs)
    instead of building in place.
- `.gitattributes` (LF line endings) plus `core.fileMode=false` /
  `core.autocrlf=false` (via the env-var git config above) are already in
  place to stop the mount'''s CRLF/mode mangling from showing as noise diffs.
- Android SDK is installed at `~/android-sdk` (platform 34, build-tools
  34.0.0) with `ANDROID_HOME`/`PATH` persisted in
  `/etc/sandbox-persistent.sh`. JDK 17 (`/usr/lib/jvm/java-17-openjdk-amd64`)
  is required to run Gradle 8.2 -- the sandbox'''s default JDK is 25, which
  Gradle 8.2 can'''t run under, so pass
  `JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64` explicitly on gradlew
  invocations.
- No release keystore exists in this sandbox, so `assemble*Release`/signed
  debug builds fail at `validateSigning*`. Unsigned compile tasks
  (`compile<Flavor><BuildType>JavaWithJavac`) work fine for verifying code
  changes. A debug keystore will be needed before we can produce an
  installable APK for on-device testing.

## Log

- 2026-09-28: Created this tracking file. Reviewed current build config
  (already targets API 34). Next: identify and prioritize remaining
  security/compatibility issues.
- 2026-09-29: Fast-scanned the codebase for Android version/security gaps.
  Findings (crash-risk, Play policy risk, and minor, in priority order):
  1. Several `registerReceiver()` calls missing required
     `RECEIVER_EXPORTED`/`RECEIVER_NOT_EXPORTED` flag -- crashes on API 33+.
  2. `bluetooth` command silently broken on API 33+ (`enable()`/`disable()`
     are no-ops); also missing `BLUETOOTH_CONNECT` runtime permission.
  3. Music player likely broken on API 33+: no `READ_MEDIA_AUDIO`
     permission for `MediaStore` audio queries.
  4. `MANAGE_EXTERNAL_STORAGE`, `QUERY_ALL_PACKAGES`, and Device Admin
     (`PolicyReceiver`) usage carry Play Store policy/review risk.
  5. `requestLegacyExternalStorage="true"` is dead config (ignored since
     `targetSdk >= 30`).
  6. `Flashlight1.java` uses the deprecated Camera1 API (low urgency, still
     functional).
  - Fixed item 1: added SDK-guarded `RECEIVER_NOT_EXPORTED` to the four
    real (non-sticky-broadcast) receiver registrations in `UIManager.java`,
    `Tuils.java`, `AppsManager.java`, `MusicManager2.java`, matching the
    existing pattern in `LauncherActivity.java`. Verified via a clean javac
    compile of both `fdroid` and `playstore` flavors. Committed as
    `fbf2a1f`.
  - Next: item 2 (bluetooth command) or item 3 (music permission).
  - Fixed item 2: `bluetooth` command now checks `BLUETOOTH_CONNECT`
    (API 31+) before touching the adapter, and on API 33+ hands off to
    system UI instead of calling the now-no-op `enable()`/`disable()`
    (`ACTION_REQUEST_ENABLE` to enable, `ACTION_BLUETOOTH_SETTINGS` to
    disable, since there is no programmatic disable API left). Added
    `BLUETOOTH_CONNECT` to the manifest and two new strings
    (`output_bluetooth_request_enable`, `output_bluetooth_manual_disable`).
  - Fixed item 3: added `READ_MEDIA_AUDIO` to the manifest and to the
    startup permission-request block in `LauncherActivity.java`
    (API 33+), alongside the existing `POST_NOTIFICATIONS` request --
    matches the existing bulk-request pattern, so `MusicManager2`'''s
    `MediaStore` query is resolved by the time it runs.
  - Fixed item 5: removed the dead `requestLegacyExternalStorage="true"`
    attribute (no-op since `targetSdk >= 30`).
  - All three verified via a clean javac compile of both `fdroid` and
    `playstore` flavors.
  - Item 6 reassessed, not a real issue: `TorchManager.java` already
    routes to `Flashlight2` (modern `CameraManager.setTorchMode` /
    Camera2 API) for any device on API 23+; the deprecated Camera1
    `Flashlight1` path only ever runs on API 21-22 (Android 5.0/5.1),
    an effectively nonexistent population. No change made.
  - Remaining: item 4 (Play Store policy review for
    `MANAGE_EXTERNAL_STORAGE`/`QUERY_ALL_PACKAGES`/Device Admin -- not a
    code fix, needs a submission-time decision).
- 2026-09-29: User tested the built APK on-device -- confirmed working.
  Ran a deeper audit (dependency versions, command-injection risk in
  shell execution, other missed registerReceiver/PendingIntent gaps,
  hardcoded secrets). Findings:
  - No missed `registerReceiver` export-flag gaps -- full sweep confirms
    the four already fixed are the only real (non-`LocalBroadcastManager`,
    non-null-sticky) receivers in the codebase.
  - No command-injection risk in shell execution: `libsuperuser`'s
    `Runtime.exec` runs a fixed shell binary array, not raw strings;
    the one other exec call site concatenates an internal constant, not
    user input. The app's "run typed commands" terminal feature
    intentionally executes user-typed shell commands -- expected
    product behavior, not a bug.
  - No WebView, no deprecated TelephonyManager device-ID calls, no
    deprecated WifiManager mutation APIs, no live hardcoded `http://`
    endpoints.
  - Dependencies (okhttp 4.12.0, jsoup 1.17.2, json-path 2.9.0,
    appcompat 1.6.1, material 1.11.0) have no confidently-attributable
    CVEs; appcompat/material trail current AndroidX by a few minor
    versions but aren't security-flagged. `htmlcleaner` and the
    single-author `CompareString2` lib are too low-visibility to assess
    without a manual Maven Central check.
  - Found and fixed: a fully commented-out debug block in
    `MainManager.java` had a hardcoded OpenWeatherMap API key baked
    into a literal URL. Dead code (never executed), but no reason to
    leave a leaked key in source -- removed. The active weather feature
    already reads its key from user config, unaffected. Committed as
    `77d7444`.
  - No other actionable findings. All known crash-risk and code-level
    security items from this effort are now resolved; what remains is
    item 4 (Play Store policy decisions -- not code).
- 2026-09-29: Investigated what each of item 4's flagged
  permissions/features actually does, to inform the Play Store
  decision:
  - `QUERY_ALL_PACKAGES`: used exactly as a launcher should (enumerating
    `CATEGORY_LAUNCHER` activities for the app drawer, `LauncherApps`
    when set as default launcher). Play has an explicit "Core Launcher
    functionality" declaration for this. Decision: keep, declare in
    Play Console at submission time. No code change.
  - `MANAGE_EXTERNAL_STORAGE`: genuinely used by the terminal's
    `cd`/file commands and the file-editor activity to browse/edit
    files anywhere in shared storage (the app's own data already uses
    scoped storage and needs no permission). Decision: keep as a core
    terminal-emulator feature, submit Play Console's All Files Access
    justification form at submission time. No code change yet -- the
    justification writeup is a submission-time task.
  - Device Admin: the *entire* feature it powered was one opt-in
    preference (lock screen on double-tap) plus admin cleanup on
    self-uninstall -- no password policy, no wipe. Decision: remove
    entirely rather than fight Play's heavy scrutiny of Device Admin
    for consumer apps. Removed `PolicyReceiver.java`,
    `res/xml/policy.xml`, the manifest `BIND_DEVICE_ADMIN` receiver,
    the double-tap-to-lock logic in `UIManager.java` (the separate
    double-tap-to-run-a-command feature is untouched), `Tuils.
    requestAdmin()`, the `double_tap_lock` preference, the
    `admin_permission` string, and the now-unneeded
    `removeActiveAdmin()` cleanup call in `tui rm`. Verified via a
    clean javac compile of both flavors including manifest/resource
    processing. Committed as `b166939`.
  - Remaining before Play Store submission: write the All Files Access
    justification for Play Console, and declare the Launcher-app
    exemption for `QUERY_ALL_PACKAGES` -- both submission-time Play
    Console tasks, not code.
