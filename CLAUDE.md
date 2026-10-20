# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

CleanYourTV debloats Android TV / Google TV sets over network ADB and replaces the home screen with a small
native launcher. It has four parts:

- `PROMPT.md`: the prompt most users paste into a coding assistant. It is the main product and stays
  brand-neutral. When an assistant gets stuck, the fix is usually one more sentence here, not a script.
  Brand-specific knowledge goes in `examples/`.
- `examples/<brand>-<model>/debloat.txt`: contributed per-TV package lists, plus the tables that index them.
- `tools/`: Python and shell helpers that talk to the TV with no `adb` binary.
- `launcher/`: the Android TV launcher app (`com.gathin.tvhome`).

## Commands

Python tests cover the example lists, the README tables, the shell helpers and the Hebrew resources. They
use simulated TVs, so no device is needed. Run them from the repo root:

```bash
python3 -m unittest discover -s tests -v                          # everything CI runs
python3 -m unittest discover -s tests -p test_examples.py -v      # one file
CLEANYOURTV_STRICT=1 python3 -m unittest discover -s tests        # also checks the README counts (CI on push to main)
```

`tests/` has no `__init__.py`. Run a single method with `cd tests && python3 -m unittest test_examples.ExampleListsTest.test_list_format`.

Launcher (JDK 17, Android SDK platform 34 + build-tools 34.0.0; `launcher/local.properties` needs `sdk.dir`):

```bash
cd launcher
./gradlew testReleaseUnitTest lintRelease assembleRelease          # what CI runs
./gradlew testReleaseUnitTest --tests com.gathin.tvhome.DockSlotsTest
```

CI fails if `app/build/outputs/apk/release/app-release.apk` is 400 KB or larger.

Against a real TV (tools venv: `python3 -m venv tools/venv && tools/venv/bin/pip install -r tools/requirements.txt`):

```bash
export TV_HOST=192.168.1.50
tools/venv/bin/python -I tools/tv.py --probe            # or '<shell cmd>', --batch < file, --push, --pull
tools/disable.sh examples/<tv>/debloat.txt              # restore.sh <same list> [stock launcher] undoes it
launcher/build-and-install.sh                           # build, install, hand Home over; INPUT_KEY_SERVICE=1 / HOME_KEY_SERVICE=1 opt-ins
tools/theme.sh themes/aurora                            # or --reset
```

To verify launcher UI changes on a TV, inject keys and pull a screenshot:
`tools/tv.py 'input keyevent KEYCODE_DPAD_DOWN; screencap -p /sdcard/s.png'` and then `tools/tv.py --pull /sdcard/s.png s.png`.

## Tools: architecture and invariants

- `tools/tv.py` is a minimal ADB-over-TCP client built on `adb-shell`, because macOS Local Network
  privacy blocks the stock `adb` binary. `tools/b64push.py` uploads large files such as APKs in base64
  chunks and checks the md5. The shell scripts always call these through `tools/venv/bin/python -I`
  (or `venv/Scripts/python.exe` on Windows). The tests replace that interpreter path with a fake, so
  keep calling it that way.
- **Every package name ends up in a shell command on the TV.** `disable.sh` and `recovery.py` refuse a
  whole list if any entry is not a bare package name (`^[A-Za-z_][A-Za-z0-9_]*(\.[A-Za-z0-9_]+)+$`,
  #165). They also strip a BOM and CRLF from lists saved on Windows (#208).
- **Recovery record (#239).** Before the first command reaches the TV, `disable.sh` runs
  `recovery.py --save`. This writes `~/.cleanyourtv/recovery-<host>-<hash of the list's absolute path>.json` with each package's prior
  state, the component that owned Home and the three animation scales. The first record wins, so a
  retry never overwrites it; a rerun only adds packages that are new to the list. `restore.sh` pipes
  `recovery.py --commands` into `tv.py --batch`. Without a record it falls back to enabling the whole list.
- **setupwraith guard.** `disable.sh` skips `com.google.android.tungsten.setupwraith` while it owns the
  saved Wi-Fi network (`dumpsys wifi`, `cname=`), because disabling it deletes the network and the TV
  drops off ADB. It also skips when the check fails. `FORCE_SETUPWRAITH=1` overrides.
- **Launcher handover order** (`build-and-install.sh`): install, then `am start -W` must report
  `Status: ok`, then disable the stock launcher, then `set-home-activity`, then query Home again. If
  Home is not ours, it rolls back and writes the recovery commands to `~/cleanyourtv-handover-<host>.txt`.
  The stock launcher is detected, not guessed. Only `KNOWN_STOCK_LAUNCHERS` are disabled, and
  `STOCK_LAUNCHER=<pkg>|none` overrides. `tests/test_launcher_handover.py` simulates these failure paths.
- Debloat list paths resolve against the caller's directory, not the repo, so users can keep lists
  outside the repo.
- Every `.sh` has a `.ps1` twin with the same behaviour. Change both together.
- Packages are only ever disabled (`pm disable-user --user 0`), never uninstalled.

## Launcher: architecture and constraints

Plain Java 17, minSdk 26, targetSdk 34. The only dependencies are AndroidX RecyclerView and
DynamicAnimation (JUnit is test-only). No Kotlin, no Compose, no network permission, no background
services apart from the opt-in accessibility services. It targets TVs with 1 GB of RAM: no blur, no
video, no polling. Adding a dependency needs an issue first. Release builds use R8 with
`-dontobfuscate` and are signed with the debug key, because the APK is sideloaded and never published.

- `MainActivity` is the home screen. The dock is the **first row of a single RecyclerView grid**
  (`DockSlots`, `DockDecoration`), not a separate view. It also owns the top-shelf hero and ambient
  glow, the single `GlassView` focus highlight that springs between tiles, the Inputs, network and
  Settings pills and their popups, and the module card prompt. `AppAdapter` builds the tiles.
- D-pad navigation is deterministic and handled explicitly (`onTileVerticalKey`, `GridNavigation`):
  Up and Down move one row, top-bar ends never fall through to tiles, and RTL (Hebrew) mirrors left and
  right. Check every edge after a navigation change.
- `Config` loads `res/values/config.xml` and then overlays `config.json` from
  `/sdcard/Android/data/com.gathin.tvhome/files/`, which `tools/theme.sh` pushes from `themes/<name>/`.
  On-TV choices from `LauncherSettingsActivity` live in SharedPreferences and take priority until a new
  `theme_revision` marker clears them. **A new setting needs a key in `config.xml`, parsing in
  `Config`, and a row in the `themes/README.md` key table.**
  Overlay keys that are unknown or of the wrong type are ignored, so a theme can never break Home.
  Anything specific to one TV belongs in `config.xml` or a theme, not in code.
- Module cards (`Cards`, `CardFeature`): third-party apps declare a tile through
  `com.gathin.tvhome.category.DOCK_CARD` plus XML meta-data. The contract is `docs/specs/card.md`,
  which must stay in sync with the code. Every call into `Cards` goes through `CardFeature.run`, so
  a failure switches cards off for the session instead of crashing Home. Decisions are pinned to the
  signer digest in `files/cards.json`. Rejections are logged under the `TvHomeCards` tag.
- Optional extras: `ClockDream` is the screensaver. `InputKeyService` and `HomeKeyService` are opt-in
  accessibility services for the remote's Input, Home and Settings keys. `QuickSettingsActivity` is
  the panel opened when `settings_key_panel` is set. `Festival`, `*Drawable` and `PetalShowerView`
  draw the festive skin. `Sky`/`SkyView` draw the time-of-day backdrop. `Shamsi` gives the Solar Hijri date.
- Unit tests in `app/src/test` are plain JVM tests with no Robolectric. Logic that needs testing goes
  in small final classes with static methods and no Android dependencies (`DockSlots`,
  `GridNavigation`, `Sky`, `WallpaperSize`, `Festival`, the `Cards` validators). Android calls are
  kept behind seams such as `CardFeature.reporter`.
- Strings: add new ones to `values/strings.xml` and the `values-*` folders. Missing translations fall
  back to English and are not a lint error. Hebrew is the exception: `tests/test_hebrew_resources.py`
  requires every default string in `values-iw` with the same `%n$x` format arguments. It also requires
  `layout-iw/activity_main.xml` to keep every view id and type from `layout/activity_main.xml`. Edit
  both layouts together.

## Adding or changing a TV list

`tests/test_examples.py` enforces the following:

- Every folder in `examples/` has a `debloat.txt`.
- The first line of each list is a `#` comment naming the model, Android version and firmware build.
- Every other non-comment line is one package name, with no duplicates.
- Every folder has a row in `examples/README.md` (first cell `` `folder/` ``).
- `README.md`'s `| TV | OS |` table has the same TVs.
- With `CLEANYOURTV_STRICT`, the "N TV lists" and "N TV models" counts in `README.md` match the
  number of folders. Maintainers fix these up on merge, so PR branches skip this check.

Notes about a device that has no package list go in `docs/devices/`. Lists should only contain
packages the contributor actually disabled and lived with, and should note anything they re-enabled.

## Scope

This repo keeps one small reference launcher, plus debloat lists, prompt and tool fixes, and small
opt-in launcher features. Alternative launchers, restyles, ports, and anything with background
services or network access belong in their own repos and get a row in `SHOWCASE.md`. Out of scope:
recommendation rows, content feeds, ads, telemetry, prebuilt APKs. `docs/TROUBLESHOOTING.md` collects
device quirks; consult it before debugging TV behaviour. Code comments cite issue and PR numbers
(`#239`) for why something is done a certain way, so follow those references before changing guarded
behaviour.
