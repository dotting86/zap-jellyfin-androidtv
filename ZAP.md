# Zap × Jellyfin: handoff and project notes

Read this before working on the fork. It records every decision taken with the user
(Flavio, writes in Italian) so work can continue on another machine or in another session.

## Separate project: do NOT modify the Zap app
This fork is a **separate project**. The existing Zap app and its repos
(`zap-tv-aggregator-frontend`, `-backend-local`, `-backend-server`) must **not be modified**
for this work; they keep living on their own. Use them only as a **reference** for logic to
re-implement in Kotlin here. Whether to depend on anything they publish, e.g. the
`@dotting86/zap-backend-local` data: ask the user first.

## Goal
Inside this fork we develop **two things** (user, 2026-09-28):
1. **Zap itself, fully re-implemented in Kotlin**, with all of Zap's features.
2. **The ad-free YouTube player.**

Jellyfin's own movies and series keep working as usual.

Details:
- Italian live TV channels (IPTV + official Rai streams)
- EPG guide
- zapping (channel list on the remote's MENU key)
- an **ad-free YouTube player**

The result is **one app** on the user's Fire TV Stick. The user's Jellyfin **server** stays
for movies and TV series only. Do **not** configure Live TV on the server; that option was
explicitly rejected.

## Repos (GitHub account `dotting86`)
| Repo | What it is |
|---|---|
| `zap-jellyfin-androidtv` (this repo) | Fork of `jellyfin/jellyfin-androidtv`, Kotlin, GPL-2.0. **Working branch: `zap`**, created from upstream tag `v0.19.10`. |
| `zap-tv-aggregator-frontend` | The existing Zap app: React Native / Expo (react-native-tvos), a separate project that must **not** be modified from here. Read-only reference for logic to re-implement in Kotlin. |
| `zap-tv-aggregator-backend-local` | Shared TS library `@dotting86/zap-backend-local` (GitHub Packages): channel registry, XMLTV parser, Rai stream resolution, deep links. Currently 0.2.2. |
| `zap-tv-aggregator-backend-server` | Fastify/SQLite server deployed on TrueNAS. **Has uncommitted local changes on the original Mac** (XMLTV EPG source, channel data fixes). |

## Upstream-sync strategy (agreed)
- All Zap code goes in its **own Gradle module** (e.g. `zap/`). Touch Jellyfin code only at **2-3 minimal hook points**:
  - a menu entry and navigation;
  - `app/build.gradle.kts` (already done: `ZAP_APPLICATION_ID`).
- Mark every change to upstream files with a `// ZAP` comment.
- Track **official release tags only** (`v0.19.x`), never upstream `master`.
- A scheduled GitHub Action should:
  1. detect a new upstream release;
  2. merge it into `zap`, build, and run tests (plus an Android TV emulator smoke test if feasible);
  3. auto-deploy to the Fire TV through the existing self-hosted TrueNAS runner (see `deploy-app.yml` in the frontend repo).
- On a merge conflict or failure, the Action **stops**: the Fire TV keeps the last good build and a PR is opened. The user then asks Claude in a session ("sistema l'aggiornamento Jellyfin") to fix it, verify on emulator **and** Fire TV, and release. The user chose this over running a Claude agent inside the Action.

## Done so far
- Forked the repo and created branch `zap` from `v0.19.10`.
- `app/build.gradle.kts`: release `applicationId` changed to **`com.zap.jellyfin`**, including the search-suggest provider authority (the official app's authority would clash), and the release app name set to **"Jellyfin Pro"**. This lets the fork install next to the official Jellyfin app.
  **Standalone Zap stays** (user, 2026-09-28, final): the Fire TV keeps **both** Jellyfin Pro and the standalone Zap app (`com.zap.tv`). Do not uninstall it. Its repos stay untouched and keep their own auto-deploy (`deploy-app.yml`).
  **App name: "Jellyfin Pro"** (user, 2026-09-28). It **replaces the official Jellyfin app** on the Fire TV.
  **NEVER uninstall anything (official Jellyfin, Zap, or any other app) without the user's explicit consent**, asked each time.
- **Not built yet.** The build needs **JDK 21** (Gradle toolchain `languageVersion=21`); the original Mac only had JDK 17.

## Workflow across machines (user, 2026-09-28)
- **Other PC:** only for the actual development (writing code), with unlimited tokens.
- **Original Mac (this one):** everything else, meaning builds, tests on the emulator and Fire TV, signing, commits, pushes and deploys. The keystore, the Fire TV adb access and the emulator live here.
- The Mac also needs **JDK 21** to build.
- Code moves between the machines **only by manual copy-paste of folders**; the other PC does no git.
  - **First setup on the other PC:** `git clone https://github.com/dotting86/zap-jellyfin-androidtv.git` (public repo, no login), or copy the folder from the Mac.
  - **Each hand-over:** the user copies the whole `zap-jellyfin-androidtv` folder from the other PC over the Mac's copy in `~/Desktop/Sviluppi/`, **excluding** `.git`, every `build/` folder, `.gradle`, `.kotlin` and `local.properties`.
  - On the Mac, `git status` / `git diff` then show exactly what changed. Build and test there, and commit/push only if it works.
  - **Claude on the other PC must not rely on git** for commits or pushes; they happen on the Mac.

## Release signing
- Signing config is read from Gradle properties or env vars: `KEYSTORE_FILE`, `KEYSTORE_PASSWORD`, `SIGNING_KEY_ALIAS`, `SIGNING_KEY_PASSWORD`.
- A keystore was generated on the original Mac at `~/.android/zap-jellyfin.keystore`, with its env file at `~/.android/zap-jellyfin-signing.env`. **Neither is in git.**
- To work on another PC, either copy both files over or generate a new keystore. A new keystore is fine as long as nothing signed with the old one is installed yet.
- **Never commit keystores or passwords.**

## YouTube player
- **SmartTube:** the app (`yuliskov/SmartTube`) is MIT, but its YouTube-extraction submodules `yuliskov/MediaServiceCore` and `yuliskov/SharedModules` have **no license**, so all rights are reserved. **Do not copy them.**
- **Chosen instead: `TeamNewPipe/NewPipeExtractor`.**
  - It is a Java library, GPL-3.0, very active (v0.26.5, 2026-08-15), and fits directly in the Kotlin fork.
  - Extract the stream URLs and play them in Jellyfin's own player, so there are no ads.
  - Licensing: GPL-3.0 inside a GPL-2.0 codebase is fine for personal, undistributed use; check the "or later" wording before any distribution.
  - Stream extraction that skips ads violates YouTube's ToS; the user knows and accepted this.
- **Still open:** a YouTube section with search and playback, selected YouTube live channels shown as TV channels, or both. Ask the user.

## Environment
- **Jellyfin server:** `http://192.168.2.18:30013` (server name "TN_CM", version 12.1.0). Movies and series only.
- **Fire TV Stick:**
  - Model AFTSSS, Fire OS 7.7, API 28, ARM32, about 1 GB RAM. It is **very slow**: JavaScript that takes ms on V8 can take minutes there.
  - adb over the LAN on `192.168.2.141:5555`. The IP is DHCP and **drifts**: it was `.137` before. Ping first; if it's unreachable, ask the user for the current IP.
  - The GitHub secret `FIRESTICK_HOST` in the frontend repo must be `IP:5555`.
  - **Never run `adb shell svc wifi disable`** or anything else that cuts its networking: adb runs over WiFi, and doing so stranded the device.
  - Fire OS caches launcher tiles per package name. The app needs a TV banner and the `LEANBACK_LAUNCHER` category.
  - Use `ANDROID_SERIAL=<ip>:5555` to target it and release builds only.
- **Android TV emulator:** AVD `ZapTV` exists on the original Mac.

## Lessons learned the hard way (from the RN app)
- **Always verify on the real Fire TV before calling something done.** The emulator is 10-20x faster and hides freezes. The user is very sensitive to being told "it works" when it doesn't.
- Measure, don't guess: use `adb shell top -H -p <pid>` (per-thread CPU) and timing logs.
- Global regex exec'd repeatedly over a 1 MB XMLTV string took 16 s on Hermes and minutes on the Fire TV. Slice each `<programme>` out with `indexOf` first (fixed in backend-local 0.2.2).
- FlatList rendered blank (2 px high) on react-native-tvos. Irrelevant in Kotlin, but lists of 250+ channels must be lazy or paged.
- Every channel row must have a **focusable** element, even with no EPG; otherwise D-pad focus gets lost.
- IPTV playlists:
  - use multiple sources and pick the best quality capped at the device's resolution;
  - keep **only Italian channels** (tvg-id ending in `.it`), not foreign channels merely available in Italy;
  - **Rai channels stay on the official Rai stream**; everything else prefers IPTV.

## User preferences
- Communicate in Italian and keep answers short. Test before claiming success, and don't make the user find breakage.
- Don't spend long investigations on tangents the user didn't ask for.
- Ask before risky or remote-device actions.
- Commit and push only when asked. Commit trailer: `Co-Authored-By: Claude <model> <noreply@anthropic.com>`.
