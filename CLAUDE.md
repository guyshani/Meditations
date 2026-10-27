# Meditation App — Build Brief for Sonnet

Oct 27, 2026 · @guy shani

This is the self-contained brief for building Guy's personal Android meditation app. Hand it to Claude Sonnet (ideally saved as `CLAUDE.md` at the repo root) and work through it phase by phase. The reasoning behind each decision lives in the [Build Plan](https://claude.ai/artifact/DYuR9sPv8kMX9oEmVsG4Rz).

## 1. How to use this brief

You are the engineer building this app with Guy, who is the only user. Work one phase at a time, in small verified steps, and stop whenever a step needs Guy's hands (a console, a device, a key).

**Session loop**

1. Read this brief and `PROGRESS.md` (create it in Phase 0).
2. Propose the next 3–8 tasks from the current phase; wait for Guy's OK.
3. Implement one task at a time; after each, run `./gradlew assembleDebug` and the unit tests (`pytest` for Python parts).
4. Commit with a short message per task.
5. Update `PROGRESS.md`: done, next, open questions.

**Rules**

- Follow the locked decisions (section 2). If something here seems wrong, say so and propose a change before deviating; record agreed changes in `DECISIONS.md`.
- Never put secrets in the repo or the APK. Keys go in `local.properties` / `.env` (both gitignored) or Google Secret Manager. The Anthropic key never ships in the app.
- When Guy must act (Google Cloud console, phone, watch), stop and give exact numbered steps, then wait.
- Prefer the latest stable library versions, managed in a Gradle version catalog (`gradle/libs.versions.toml`). Check current docs rather than relying on memory for APIs that change (Credential Manager, Health Connect, Media3, Wear Tiles).
- Every user-facing string goes in resources in English and Hebrew from day one.
- Keep each phase shippable: the app must build and run after every task.

## 2. Locked decisions

| Area | Decision |
| --- | --- |
| Platform | Native Android, Kotlin + Jetpack Compose; Wear OS module for Galaxy Watch (Watch4 or later) |
| Distribution | Personal sideload (debug/release APK), no Play Store for now |
| Languages | English and Hebrew: UI, meditations, voices; full RTL in Hebrew |
| Voices | Female in both languages; exact voices chosen after a listening test |
| LLM | Claude Sonnet 5.5 (`claude-sonnet-5-5`) for scripts; Claude Haiku 5.5 (`claude-haiku-5-5`) allowed for cheaper custom requests; DictaLM 3.0 optional local |
| TTS | Google Cloud TTS Chirp 3: HD; A/B against ElevenLabs Eleven v3 (`language_code="he"` for Hebrew) before locking voices |
| Where generation runs | Python package on Guy's computer for packs; same package on Google Cloud Run for custom meditations |
| Account | Sign in with Google (Credential Manager), skippable; scopes requested only when needed |
| Backup | Automatic to Google Drive app-data folder + Android Auto Backup |
| YouTube playback | Default: hand off to the YouTube app (Premium works there); option: in-app IFrame player |
| Home total | Default period: this month; options Today / Week / Month / All time |
| Categories (13) | Relaxation, Sleep, Focus, Presence, Sound healing, Timer (no guidance), Self-love, Breathwork, Body scan, Loving-kindness, SOS/anxiety, Morning, NSDR/yoga nidra |
| Extra features | Animated daily streak, auto DND, lock-screen/headphone controls, favorites + recents, ambient mixer, sleep timer with fade-out, Quick Settings tiles, mood check-in, auto backup, Galaxy Watch tile, custom AI meditation |
| Out of scope for now | Home-screen widget, reminders, journal, insights screen, courses |

## 3. Repository layout

One monorepo holds the Android app, the watch app, the generator and the cloud service, so the script format and manifest schema live in one place.

```
meditation-app/
├── CLAUDE.md                 # this brief
├── PROGRESS.md               # running log: done / next / questions
├── DECISIONS.md              # changes agreed after this brief
├── android/
│   ├── settings.gradle.kts
│   ├── gradle/libs.versions.toml
│   ├── app/                  # phone app (Compose)
│   ├── wear/                 # Galaxy Watch app + tile
│   ├── core/model/           # shared Kotlin data classes (Session, Meditation…)
│   ├── core/data/            # Room, DataStore, repositories
│   ├── core/playback/        # Media3 service, mixer, bells, timers
│   └── core/sync/            # Drive backup, Health Connect, calendar, Wear Data Layer
├── generator/                # Python package `medgen`
│   ├── medgen/ (script.py, tts.py, mix.py, pack.py, cli.py)
│   ├── prompts/en/ prompts/he/   # one template per category
│   ├── assets/ambient/ assets/bells/   # CC0 loops and bells
│   ├── tests/
│   └── pyproject.toml
├── cloud/                    # Cloud Run service wrapping `medgen`
│   ├── main.py  Dockerfile  requirements.txt
└── packs/                    # build output (gitignored except a tiny sample)
```

## 4. Tech stack and conventions

| Concern | Use |
| --- | --- |
| Language / min SDK | Kotlin, minSdk 26, target the latest stable SDK |
| UI | Jetpack Compose, Material 3, Navigation Compose; Lottie (or Compose animation) for the streak |
| Architecture | MVVM + repositories, unidirectional state (`StateFlow` UI state per screen), Hilt for DI, coroutines |
| Storage | Room (entities in section 5), DataStore Preferences for settings |
| Audio | Media3 ExoPlayer + `MediaSessionService`; separate looping players for ambient layers |
| Background work | WorkManager (backup, pack downloads, custom-generation jobs, sync retries) |
| Sign-in / scopes | Credential Manager "Sign in with Google"; `AuthorizationClient` for `drive.appdata` and `youtube.readonly` |
| Network | Retrofit or Ktor + kotlinx.serialization |
| YouTube | Data API v3 (REST), intent hand-off to the YouTube app, `android-youtube-player` for the in-app option |
| Health / calendar | Health Connect client (`MindfulnessSessionRecord`); `CalendarContract` for events |
| Tiles | `TileService` (phone Quick Settings); Wear Tiles API + Compose for Wear OS (watch) |
| Phone ↔ watch | Wearable Data Layer API |
| Generator | Python 3.11+, `anthropic`, `google-cloud-texttospeech`, `pydub` + ffmpeg, `typer` CLI, `pytest` |
| Cloud | Cloud Run (FastAPI), Cloud Storage, Secret Manager |

**Conventions**

- Packages by feature (`feature/home`, `feature/library`, `feature/player`, `feature/youtube`, `feature/journey`, `feature/settings`, `feature/account`).
- All times stored as UTC epoch millis; day boundaries computed in the device time zone.
- Use start/end instead of left/right in layouts so Hebrew mirrors correctly.
- Unit-test pure logic (totals, streaks, timer schedule, backup rotation, script parser); UI tests only for the Home total card and the log-session sheet.

## 5. Data model

Six Room tables plus DataStore settings. `Session` is the single source of truth for all totals, streaks and syncs.

| Entity | Fields | Notes |
| --- | --- | --- |
| Session | id, startTime, endTime, playedSeconds, source (AI / TIMER / YOUTUBE / MANUAL / WATCH), contentId?, category?, language?, moodBefore?, moodAfter?, healthConnectId?, calendarEventId?, createdAt, updatedAt | Totals = sum of `playedSeconds` |
| Meditation | id, title, category, language (EN / HE), durationSec, voice, audioPath, origin (PACK / CUSTOM), packId?, prompt?, tags, isFavorite, lastPlayedAt? | Filled from pack manifests or custom generation |
| ContentPack | id, name, version, language, installedAt, sourceUrl? | Update or remove a batch of tracks |
| YouTubeItem | videoId, title, channel, durationSec, thumbnailUrl, category?, language?, isFavorite, lastPlayedAt?, cachedAt | Search cache + favorites |
| PendingYouTubeSession | videoId, startedAt, durationSec | At most one row |
| CustomJob | id, request (JSON), status (QUEUED / RUNNING / DONE / FAILED), error?, meditationId?, createdAt | Tracks Cloud Run requests |

**Settings (DataStore):** totalPeriod (default MONTH), uiLanguage (SYSTEM / EN / HE), contentLanguage (EN / HE / BOTH), weekStart (default from locale), dailyGoalMin (default 10), minSessionSec (default 60), dndDuringSession (default true), askMoodBefore, askMoodAfter, ytPlayMode (YOUTUBE\_APP / IN\_APP), healthConnectSync, calendarSync, backupEnabled, lastBackupAt, googleUserId, googleEmail, savedMixes (JSON), timerPresets (JSON).

## 6. Behavior specs

### Tracking and totals

- A session records only played time; pauses don't count. Sessions shorter than `minSessionSec` are discarded silently.
- A session belongs to the local calendar day it started on.
- Home total card: sum of `playedSeconds` for the chosen period (default: current calendar month), shown as hours + minutes. Tapping the card cycles Today → Week → Month → All time without changing the saved default.
- Manual entry: date, start time, minutes, category; source = MANUAL.
- Journey: month calendar heatmap (shade by minutes); tapping a day lists its sessions with edit and delete.

### Streak

- A day counts when its total ≥ `dailyGoalMin`. The streak = consecutive counted days ending today, or ending yesterday if today isn't counted yet (so the streak doesn't reset in the morning).
- Home shows an animated ring filling toward today's goal, plus the streak number. Milestones at 7, 30, 100 and 365 days trigger a one-time celebration animation.

### Player, timer and focus

- All audio plays through the Media3 service: notification, lock screen, Bluetooth/headphone buttons and watch media controls work.
- Auto DND: if `dndDuringSession` and Do Not Disturb access is granted, switch to priority-only at start and restore the previous filter at end or on crash recovery. Without access, show a one-time prompt to grant it.
- Ambient mixer: up to 4 looping layers (rain, ocean, fire, forest, drone, bowls) with per-layer volume; works under any track or the timer; mixes can be saved.
- Sleep timer: 15/30/45/60 min or end of track; volume fades to zero over the last 45 s, then stops.
- Timer: duration, warm-up, interval bells, ending bell, ambient mix; presets savable.
- Mood check-in: optional 1–5 picker before (if `askMoodBefore`) and after (if `askMoodAfter`).
- Favorites: heart on any meditation or YouTube video. Recents: last 10 distinct items from `Session`.

### YouTube

- Search with `search.list` (100 quota units each; cache results 24 h) and `relevanceLanguage` set to the content language. Curated rows per category use stored queries, e.g. "20 minute body scan meditation" / "מדיטציה סריקת גוף".
- My YouTube (after `youtube.readonly` consent): subscriptions and playlists via 1-unit `list` calls.
- Play: in YOUTUBE\_APP mode, open the video with an intent to the YouTube app and save a `PendingYouTubeSession`. On return (`onResume`), show the log sheet pre-filled with min(elapsed, video duration), editable. Pending rows older than 6 h expire. In IN\_APP mode, log exact played time from player events.
- Never block or strip ads.

### Account and backup

- First launch offers Sign in with Google (skippable). If a backup exists in Drive, offer Restore.
- Backup = zip of a consistent Room snapshot (checkpoint the WAL first) + settings JSON, uploaded to the `appDataFolder` as `backup-<yyyyMMdd-HHmm>.zip`; keep the newest 7. Triggers: after each saved session (debounced 5 min) and nightly (Wi-Fi + charging).
- Custom AI tracks upload once each to `appDataFolder/custom/`; pack tracks are not backed up.
- Restore replaces local data after a confirmation dialog, then re-downloads packs and custom tracks.
- Enable Android Auto Backup for the database and settings only (exclude audio).

### Sync out (toggles)

- Health Connect: write each session as `MindfulnessSessionRecord` (type matches category where possible); check feature availability first.
- Calendar: Guy creates a "Meditation" calendar once in Google Calendar on the web. The app lets him pick it in Settings and inserts events into it via `CalendarContract`, so they sync to his Google account (a calendar the app creates itself would be local-only). One event per session, e.g. "Meditation · Sleep · 20 min"; update or delete the event when the session changes.

### Tiles and watch

- Quick Settings tiles: "Start timer" (last preset) and "Breathe 3 min".
- Galaxy Watch: a tile with Start timer / Breathe buttons and today's minutes; a small watch app runs the timer with haptic bells and a breathing pacer. Finished sessions go to the phone over the Data Layer (source = WATCH); the phone owns the session log.

### Languages

- UI in English and Hebrew via resources and the per-app language setting; Hebrew uses RTL.
- Library defaults to `contentLanguage`, with a toggle to show both.

## 7. Content generator (`medgen`, Python)

One package turns (category, language, minutes, optional prompt) into a finished audio track plus manifest. The CLI builds packs on Guy's computer; the Cloud Run service imports the same functions.

**Pipeline**

1. **Script** (`script.py`): call Claude with the category template from `prompts/<lang>/<category>.md`. Hebrew scripts are written natively in Hebrew, never translated. Output is plain text with markers; validate it with the parser and retry once if invalid.
2. **Parse:** split into speech segments and markers.
3. **TTS** (`tts.py`): synthesize each speech segment with the chosen engine and voice. Pluggable backends: `google_chirp3` (default), `elevenlabs_v3`, `azure`, `local`. Cache segments by hash of (text, voice, engine) to avoid paying twice.
4. **Mix** (`mix.py`): concatenate segments and silences; add bells; lay the ambient bed under the voice with fade-in 10 s and fade-out 20 s. Normalize loudness so tracks play at similar volume. Export M4A (AAC, 128 kbps, 44.1 kHz).
5. **Duration fit:** if the result is more than 10% off the target length, scale the pause lengths (not the speech) to fit.
6. **Pack** (`pack.py`): write the track manifest and zip tracks into a pack.

**Script markers** (one per line)

```
[pause 20s]        silence of N seconds
[bell]             single bell
[breath in 4]      spoken cue handled by the text; marker adds a 4 s silence
[breath out 6]     same, 6 s
[section Body scan]  label only, not spoken
```

**Prompt template contents** (per category and language): purpose, structure (arrival → main practice → closing), tone (warm, slow, second person, short sentences), target word count from minutes (about 60–80 spoken words per minute of track, the rest silence), marker rules, things to avoid (medical claims, guilt, complex words that TTS may misread in Hebrew).

**Track manifest** (`<id>.json`)

```json
{
  "id": "sleep-sea-10-he-01",
  "title": "ים של שקט",
  "category": "sleep",
  "language": "he",
  "durationSec": 604,
  "voice": "google_chirp3:<voice-name>",
  "ambient": ["ocean"],
  "tags": ["sea", "body-scan"],
  "file": "audio/sleep-sea-10-he-01.m4a",
  "origin": "PACK",
  "createdAt": "2026-11-02T10:00:00Z"
}
```

**Pack:** `pack.json` (`id`, `name`, `version`, `language`, `tracks[]`) plus `audio/`, zipped as `<pack-id>-v<version>.zip`.

**CLI**

```
medgen track --category sleep --lang he --minutes 10 --prompt "the sea" --voice <id>
medgen pack  --lang en --per-category 3 --minutes 5,10,20 --out packs/
medgen voices --lang he --gender female --sample "text"   # listening test
```

Sound healing and timer content skip the LLM and TTS: generate tones, drones and binaural beats procedurally, or use CC0 recordings from `assets/`.

## 8. Cloud Run service (custom meditations)

A small FastAPI service in `cloud/` wraps `medgen` so the app can request a custom meditation without holding any API keys.

**API**

```
POST /v1/generate
  Authorization: Bearer <Google ID token from Credential Manager>
  { "language": "he", "category": "sleep", "minutes": 12,
    "prompt": "the sea, letting go of the day", "voice": "<id>", "ambient": ["ocean"] }
  → 202 { "jobId": "…" }

GET /v1/jobs/{jobId}
  → { "status": "queued|running|done|failed", "error": null,
      "manifest": { … }, "audioUrl": "<signed Cloud Storage URL, 1 h>" }
```

**Security and limits**

- Verify the ID token: issuer Google, audience = the Web OAuth client ID, email in the `ALLOWED_EMAILS` env var (Guy only).
- `DAILY_LIMIT` env var (default 10 generations/day); `minutes` capped at 45.
- Anthropic key from Secret Manager; TTS via the service's own service account. Nothing secret in the image.

**Execution**

- MVP: run the job in the request handler with Cloud Run's request timeout raised to 15 min and CPU always allocated; store status JSON and audio under `gs://<bucket>/custom/<jobId>/`.
- If that proves flaky, move generation to a Cloud Run Job triggered by the API, same storage layout.
- App side: a WorkManager worker submits, polls every 15 s with backoff, downloads the M4A, inserts a `Meditation` (origin CUSTOM) and queues its Drive backup.

**Deploy:** `gcloud run deploy medgen-api --source cloud/ --region <region> --allow-unauthenticated`. Cloud Run's IAM gate stays open because the app authenticates with a Google sign-in ID token, not IAM credentials; the token and email check in the service is the access control.

## 9. Setup checklist for Guy

These steps need Guy's accounts or hardware; Sonnet should walk through each when its phase starts.

**Computer**

- [ ] Android Studio (latest stable) with an Android SDK and a Wear OS emulator image
- [ ] Python 3.11+, ffmpeg, Google Cloud CLI (`gcloud`)
- [ ] Git repo created (private) with the layout in section 3

**Google Cloud project**

- [ ] Create a project and set a billing budget alert
- [ ] Enable APIs: YouTube Data API v3, Google Drive API, Cloud Text-to-Speech, Cloud Run, Cloud Storage, Secret Manager, Cloud Build, Artifact Registry
- [ ] Google Auth Platform: External audience, app name and support email; declare scopes `drive.appdata` and `youtube.readonly`; set publishing status to **In production** (in Testing, authorizations expire after 7 days)
- [ ] OAuth clients: one **Web** client (its ID is the `serverClientId` for Credential Manager and the audience for Cloud Run) and one **Android** client with the package name and the debug keystore SHA-1 (add the release SHA-1 later)
- [ ] API key for YouTube Data API, restricted to the Android app (package + SHA-1) and to that API only. Android-restricted keys require the app to send `X-Android-Package` and `X-Android-Cert` headers on REST calls
- [ ] Private Cloud Storage bucket with `packs/` and `custom/` prefixes
- [ ] Secret Manager secret `anthropic-api-key`
- [ ] On the computer: `gcloud auth application-default login` so `medgen` can call TTS

**Other accounts**

- [ ] Anthropic API key (Claude Console), also in `generator/.env` for local runs
- [ ] Optional: ElevenLabs API key for the voice A/B test

**Devices**

- [ ] Phone: developer options + USB debugging; Health Connect installed and up to date
- [ ] Galaxy Watch (Watch4 or later): developer options, ADB debugging over Wi-Fi
- [ ] Google Calendar on the web: create a calendar named "Meditation"

## 10. Build phases

Five phases, each ending in something Guy uses daily. Do them in order; within a phase, the order of tasks is a suggestion.

### Phase 0 — Project skeleton

- [ ] Monorepo layout, `CLAUDE.md`, `PROGRESS.md`, `DECISIONS.md`, `.gitignore` (keys, `local.properties`, `.env`, `packs/`)
- [ ] Gradle modules from section 3, version catalog, Hilt, Room, DataStore, Navigation
- [ ] Calm Material 3 theme (light + dark) with a font that covers Hebrew
- [ ] Bottom nav: Home, Library, YouTube, Journey, Settings (placeholders)
- [ ] English + Hebrew string resources, per-app language setting

**Done when:** the app installs on the phone and switches between English and Hebrew with correct RTL mirroring.

### Phase 1 — Foundation and tracking

- [ ] Silent timer (duration, warm-up, interval and end bells, presets) running in the Media3 service
- [ ] Session log rules (section 6) + manual entry
- [ ] Home: total card (month default, tap to cycle), animated streak ring, milestone celebration, 7-day bars, Start-timer button
- [ ] Journey: calendar heatmap, day detail with edit and delete
- [ ] Settings: period, week start, daily goal, minimum session, languages
- [ ] Welcome screen with Sign in with Google (skippable)
- [ ] Drive app-data backup (after session + nightly, keep 7), restore flow, Auto Backup rules
- [ ] Unit tests: totals per period, streak (incl. morning-before-goal case), backup rotation

**Done when:** a week of logged sits survives uninstall → reinstall → restore.

### Phase 2 — Generator, library and player

- [ ] `medgen` package: parser, Claude script step, Chirp 3: HD backend, mixer, duration fit, manifest, pack; tests for the parser and duration fit
- [ ] Prompt templates for all 13 categories in English and Hebrew
- [ ] `medgen voices` listening test → Guy picks one female voice per language; record in `DECISIONS.md`
- [ ] First packs: 1 track per category per language to validate, then 3 per category per language (5/10/20 min)
- [ ] Procedural sound-healing tracks and CC0 ambient loops
- [ ] App: pack import (bundled assets), Library (category grid, filters, language toggle), track detail
- [ ] Player: lock-screen and headphone controls, auto DND, ambient mixer + saved mixes, sleep timer with fade-out, mood check-in hooks
- [ ] Favorites and recents rows on Home

**Done when:** at least 3 tracks per category play offline in both languages, with screen-off playback and DND working.

### Phase 3 — YouTube

- [ ] Data API client with Android-restricted key, 24 h cache, `relevanceLanguage`
- [ ] Search screen + curated rows per category and language; favorites
- [ ] My YouTube: `youtube.readonly` consent, subscriptions and playlists
- [ ] Play via YouTube app intent (default) and in-app player option
- [ ] Pending session + log-session sheet on return; exact logging in in-app mode

**Done when:** a YouTube sit played in the YouTube app (ad-free with Premium) shows up in the month total.

### Phase 4 — Companion features

- [ ] Health Connect write (with availability check and permission flow)
- [ ] Calendar events into the picked "Meditation" calendar
- [ ] Quick Settings tiles: Start timer, Breathe 3 min
- [ ] Mood check-in UI before/after, shown in day detail
- [ ] `wear` module: tile + minimal timer and breathing app with haptics; Data Layer sync to the phone's session log

**Done when:** Guy starts a sit from the watch and it appears in the phone's calendar view and in Health Connect.

### Phase 5 — Custom AI and polish

- [ ] Cloud Run service (section 8) deployed; secrets in Secret Manager
- [ ] App: Create screen (prompt, language, length, voice, ambient), WorkManager job, download, save to Library, Drive backup of the track
- [ ] Daily limit and error states
- [ ] Polish: accessibility (TalkBack labels, font scaling), RTL review of every screen, battery check during long sessions, release-signed APK for sideload

**Done when:** Guy types a request in Hebrew and plays the generated meditation within a few minutes.

## 11. Gotchas

- **YouTube Premium** works only in the YouTube app. The in-app player runs in a WebView where Google blocks sign-in, so ads may appear there. Default to the YouTube-app hand-off.
- **The old YouTube Android Player SDK** is deprecated; don't use it. Use the IFrame-based `android-youtube-player`.
- **YouTube quota:** 10,000 units/day; each `search.list` costs 100. Cache aggressively; use `playlistItems.list` (1 unit) for channels.
- **OAuth Testing status** expires authorizations after 7 days. Keep the Google Auth Platform app In production.
- **Credential Manager needs the Web client ID** as `serverClientId`, not the Android client ID. A SHA-1 mismatch between the build and the Android OAuth client fails sign-in silently; check it first.
- **Hebrew TTS:** written Hebrew omits vowels, so words can be misread. Keep vocabulary simple, test niqqud on ambiguous words, and listen to every Hebrew track. ElevenLabs Multilingual v2 is unusable for Hebrew; use v3 with `language_code="he"`.
- **Chirp 3: HD SSML** supports only a few tags, so all pauses are silences inserted by the mixer, not SSML breaks.
- **Auto DND** needs Notification Policy Access; always restore the previous mode, including after a crash.
- **Calendar:** events must go into a calendar on the Google account; app-created calendars stay local.
- **Health Connect** mindfulness records need a recent Health Connect version; check the feature before showing the toggle.
- **Galaxy Watch:** only Watch4 and later run Wear OS. Google is moving watch tiles toward Wear Widgets; check which to target when Phase 4 starts.
- **Cloud Run CPU** is throttled after a response unless CPU is always allocated; long generation must run inside the request (MVP) or in a Cloud Run Job.
- **Secrets:** the Anthropic key lives only in Secret Manager and `generator/.env`; the YouTube API key in the APK must be Android-restricted.

## 12. References

- [Meditation App — Build Plan (rationale and diagrams)](https://claude.ai/artifact/DYuR9sPv8kMX9oEmVsG4Rz)
- [Android — Authorize access to Google user data](https://developer.android.com/identity/authorization)
- [Google Drive API — application data folder](https://developers.google.com/drive/api/guides/appdata)
- [YouTube Data API — quota costs](https://developers.google.com/youtube/v3/determine_quota_cost)
- [android-youtube-player](https://github.com/PierfrancescoSoffritti/android-youtube-player)
- [Health Connect — Track mindfulness](https://developer.android.com/health-and-fitness/health-connect/features/mindfulness)
- [Wear OS — Create your first tile (codelab)](https://developer.android.com/codelabs/wear-tiles)
- [Wear OS — Wear Widgets](https://developer.android.com/training/wearables/widgets)
- [Google Cloud TTS — Chirp 3: HD voices](https://docs.cloud.google.com/text-to-speech/docs/chirp3-hd)
- [ElevenLabs — supported languages](https://elevenlabs.io/docs/help-center/other/what-languages-do-you-support)
- [Hebrew TTS provider comparison (community test, 2025)](https://github.com/danielrosehill/Hebrew-TTS-Providers)
- [Dicta-LM 3.0](https://huggingface.co/dicta-il/DictaLM-3.0-1.7B-Instruct)
- [Google — Testing-status tokens expire after 7 days](https://developers.google.com/google-ads/api/docs/get-started/common-errors)
