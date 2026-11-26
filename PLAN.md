# ftmsLedger implementation plan

Status: planning only. No application code is scaffolded yet.

This document is the build plan for **ftmsLedger** (that casing in every user-visible string, type name, and comment). It logs kilometres, time, and speed from an under-desk walking pad over Bluetooth Low Energy, using the Bluetooth SIG Fitness Machine Service, so the owner can keep her km without a vendor app.

Pinned at the time of writing (26 November 2026):

| Choice | Pin |
| --- | --- |
| Expo SDK | 58 (released 15 October 2026), React Native 0.88 |
| UI | `@expo/ui` ~58.0.29 (SwiftUI on iOS, Jetpack Compose on Android, universal components on web) |
| Navigation | Expo Router ~58.0.32, with `expo-router/native-tabs` |
| BLE, native | `@sfourdrinier/react-native-ble-plx` 3.9.x (3.9.3 is the current stable line) |
| BLE, web | Our own thin Web Bluetooth transport. Not the library's unfinished 4.x `/web` entry |
| Storage | `expo-sqlite` ~58.0.18 |
| Live Activity | `expo-widgets` ~58.0.29 |
| Share sheet | `expo-sharing` ~58.0.23 and `expo-file-system` |

Install Expo libraries with `npx expo install` so the SDK 58 set stays compatible. A later SDK upgrade is its own `chore` commit; do not bump one package ahead of the SDK.

Platform identifiers that cannot preserve casing (bundle id, URL scheme, Android package) use the lowercase form `ftmsledger`. Everything a person reads says `ftmsLedger`.

## 1. Goals

- [ ] Show walked distance in kilometres, taken from the pad, without FitShow, Kinomap, or any other vendor app.
- [ ] Record each walk as a session: distance, moving time, elapsed time, speed, and incline when the pad reports it.
- [ ] Start and stop a session from the belt itself, with a manual override.
- [ ] Survive a counter reset, a brief disconnect, and pads that only publish distance while the belt is moving.
- [ ] Show running totals for today, this week, this month, and all time.
- [ ] Keep every workout on the phone. No account.
- [ ] Feel like a native iPhone app first, then a native Android app, then a Chromium web app.
- [ ] Be developable before a pad arrives, through a simulator that emits real FTMS bytes.

The shopping context matters. The owner is comparing budget walking pads. Many are advertised as FitShow or Kinomap compatible. At least one HomeFitnessCode pad's own app speaks FTMS. Kinomap compatibility is a hint, not proof: Kinomap also speaks vendor protocols. A pad is in scope for the first release only when it exposes Fitness Machine Service `0x1826`.

## 2. Non-goals

- Outdoor GPS runs, route maps, or step counting from the phone.
- A separate heart-rate strap. If the pad includes heart rate inside Treadmill Data, store it. Do not pair a second sensor.
- Invented calories. Show expended energy only when the pad sends it.
- Accounts, cloud backup, social features, or analytics.
- An Apple Watch or Wear OS app. The phone Lock Screen and Dynamic Island are in scope later; a watch app is not.
- iPad split view. Do not block it, and do not design for it.
- Remote control of belt speed or incline in the first release. It is a later, opt-in, safety-gated feature.
- FitShow `FFF0`, KingSmith, or other proprietary protocols in the first release.
- Promising that iOS will keep logging with the screen locked. See [Background behaviour](#13-background-behaviour).

## 3. User stories

- As the owner, I want today's kilometres on one screen when I open ftmsLedger, so I do not open a vendor app to know what I walked.
- As the owner, I want the session to begin when the belt starts and to end when the belt has stayed stopped, so a walk is logged even if I forget to tap start.
- As the owner, I want a short pause (tying a shoe, picking up a glass) to pause the moving clock without splitting the walk into two sessions.
- As the owner, I want a Bluetooth blip to reconnect into the same session, without double-counting the metres already walked and without losing them.
- As the owner, I want a correct distance even when the pad reports distance only while the belt is moving, or resets its counter every time the belt stops.
- As the owner, I want incline recorded and shown when the pad has it, and I want no fake "0%" on a pad that has no incline.
- As the owner, I want week, month, and all-time kilometres, plus a simple goal and streak, so the daily number has a context.
- As the owner, I want to export one walk or the whole history as CSV or JSON through the system share sheet.
- As the owner, I eventually want the same walk in Apple Health or Health Connect, written once, as a walk.
- As the owner, I do not want an account, and I do not want the app to upload anything unless I later opt in to Strava.
- As the owner, before the pad arrives, I want a simulated pad so I can see the live screen and the totals behave.
- As the owner, if a future pad allows it and I explicitly turn it on, I want the phone to set speed or incline, clamped to the range the pad advertises.

## 4. Feature phases

Each item is one of: **MVP**, **Next**, or **Later**. MVP is the logging loop on a real FTMS pad, plus the simulator needed to build it. Next is the rest of the personal logbook. Later needs either a platform capability with sharp edges (Health, background, control) or a pad that is not FTMS.

### 4.1 MVP

- [ ] Development build on Expo SDK 58 (Expo Go cannot load the BLE native module).
- [ ] Scan, connect, remember, and reconnect to a peripheral that exposes service `0x1826`.
- [ ] Read Fitness Machine Feature (`0x2ACC`) and, when present, Supported Speed Range (`0x2AD4`) and Supported Inclination Range (`0x2AD5`).
- [ ] Subscribe to Treadmill Data (`0x2ACD`). Also subscribe to Fitness Machine Status (`0x2ADA`) and Training Status (`0x2AD3`) when they exist.
- [ ] Live session screen: distance (km), speed (km/h), moving time, elapsed time, incline when known, connection state.
- [ ] Auto start and auto stop from belt speed, with status opcodes used when the pad sends them.
- [ ] Manual start, pause, and stop.
- [ ] Session history and a session detail screen.
- [ ] Totals for today, this week, this month, and all time, updated while a walk is in progress.
- [ ] Distance accumulator that handles counter resets, reconnects, and speed integration when device distance is missing or frozen.
- [ ] Simulated FTMS peripheral that feeds the same parser, for development without hardware.
- [ ] Local SQLite database, opened and migrated on launch.
- [ ] iPhone, Android, and web. Web connects only where Web Bluetooth exists. History and the simulator work in any browser.
- [ ] Permission prompts and a clear denied / Bluetooth-off empty state.
- [ ] Keep the screen awake during a session.

### 4.2 Next

- [ ] One daily kilometre goal and one weekly kilometre goal.
- [ ] Streak: consecutive local days that contain a kept walk. Computed, not stored as source of truth.
- [ ] CSV and JSON export of one session or of all kept sessions, then the system share sheet.
- [ ] Discard prompt for an accidental nudge (default: under 20 m and under 45 s of moving time).
- [ ] Edit a saved distance when a pad counter was wrong. Mark the session as edited.
- [ ] Session notes.
- [ ] Speed sparkline for one session, from stored samples.
- [ ] iOS Live Activity: Lock Screen banner and Dynamic Island, with a stale date so a frozen number does not look live.
- [ ] Haptics on start, pause, stop, and goal completion, following the system haptics setting.

### 4.3 Later

- [ ] Apple Health and Health Connect write of a completed walk. Write-only. Retry-safe.
- [ ] Optional Strava upload. The only networked feature. Off until the owner connects an account.
- [ ] Optional speed and incline control through the Fitness Machine Control Point (`0x2AD9`), off by default, with a safety interlock.
- [ ] Android foreground service so a walk can continue with the screen off.
- [ ] Best-effort iOS background restore. Documented as best-effort, not as a guarantee.
- [ ] FitShow service `FFF0` read path, only after a purchased pad lacks `0x1826` and nRF Connect shows `FFF0`.
- [ ] Other proprietary families (WalkingPad `FE00`, KingSmith `0x1234`) only under the same rule: a pad on the desk needs it, and the GATT table has been recorded.

### 4.4 Explicitly out of every phase unless a new plan says otherwise

- [ ] Reading the owner's existing Health data.
- [ ] Estimating calories from weight.
- [ ] Social, leaderboards, or ads.
- [ ] Controlling the belt without an on-screen confirmation and a supported-range clamp.

## 5. Stack

### 5.1 Why Expo SDK 58 and a development build

SDK 58 is the current Expo SDK. It targets the React Native 0.88 New Architecture, stable Expo Router native tabs, and `@expo/ui` as a first-class way to put SwiftUI and Jetpack Compose on screen.

BLE needs a native module. Expo Go does not include one. The app runs as a development build (`expo-dev-client`, `npx expo run:ios`, `npx expo run:android`, or EAS). The simulator mode still needs that build on a phone, because it lives in the same binary. Web can run from `npx expo start --web` without a dev client.

### 5.2 Why `@sfourdrinier/react-native-ble-plx`

The native central is `@sfourdrinier/react-native-ble-plx` on the 3.9.x line.

- It is the maintained fork of `react-native-ble-plx`, built for Expo SDK 57+ and React Native 0.86+, which covers SDK 58 / RN 0.88.
- It is a TurboModule. The old `dotintent/react-native-ble-plx` package is the wrong line for this SDK.
- It ships an Expo config plugin (Continuous Native Generation) for Bluetooth permission strings, iOS background mode, and Android scan/connect permissions. Native projects are generated, not hand-edited.
- The API matches what FTMS needs: scan, connect, request MTU, discover services, read, notify, and write-with-response plus indicate for the Control Point.
- 3.9.3 adds `getRestoredState()` for a later iOS restore attempt. Leave restoration off in the MVP plugin config (`iosEnableRestoration: false`) so the binary does not claim background behaviour it does not honour yet.

The 4.x "Unified BLE Manager" charter describes a future `/web` subpath. That subpath is not the dependency. Web Bluetooth is a separate, small transport in this repo so a 4.x alpha cannot destabilise phone builds.

Reject, for this app:

- Expo Go, because the native module will not load.
- A from-scratch Core Bluetooth / Android BLE module, because the fork already covers scan, connect, notify, and the Expo plugin.
- `react-native-health` (the unmaintained package) for the Health phase.

### 5.3 Why `@expo/ui`

`@expo/ui` is not a themed replica of iOS painted in React Native views. On iOS the components are SwiftUI. On Android they are Jetpack Compose. The universal entry (`@expo/ui`) is the default, so one component tree runs on iOS, Android, and web and still resolves to the native control.

Use the platform entry only when the universal control is missing a behaviour the screen needs:

- `@expo/ui/swift-ui` inside `Host` for an iOS-only control (for example a SwiftUI `Gauge`, or an SF Symbol `contentTransition`).
- `@expo/ui/jetpack-compose` for the Android equivalent.

Live Activities and widgets are restricted to Expo UI components. Building the in-app screens with the same kit keeps the Lock Screen layout from becoming a second design system.

Where the universal set has a gap on web, build the screen with the same structure and universal controls, and only then fall back to a web-specific branch. A screen that exists on iPhone must have a web counterpart, even if the web counterpart cannot talk to a pad.

### 5.4 Other libraries

| Need | Library | Phase |
| --- | --- | --- |
| File routes and native tab bar | `expo-router`, `expo-router/native-tabs` | MVP |
| Database | `expo-sqlite` | MVP |
| Screen awake during a walk | `expo-keep-awake` | MVP |
| Haptics | `expo-haptics` | Next |
| Write a CSV/JSON file | `expo-file-system` | Next |
| Share sheet | `expo-sharing` | Next |
| Live Activity | `expo-widgets` | Next |
| HealthKit and Health Connect | Evaluate `@appeeky/expo-healthkit` against SDK 58 at the start of that phase. It speaks both stores through one API and already requires a dev build. If it does not build on SDK 58, write a thin Expo module for the two write calls this app needs. Do not take a dependency on abandoned Health packages. | Later |
| Strava token | `expo-secure-store` | Later |

No Redux. Session state is a plain class the UI subscribes to with `useSyncExternalStore`.

React components are written as `const Name: FunctionComponent<Props> = ...`.

## 6. Screens and navigation

Primary navigation is a three-tab bar. A walk is a temporary task, so it is a full-screen cover, not a tab. Pairing is a sheet. This matches the Human Interface Guidelines: the tab bar is for persistent top-level sections, a sheet is for a short task that returns to the same place, and a full-screen cover is for an immersive task with its own ending.

```
Tabs (NativeTabs)
  Today          SF Symbol house.fill        Material home
  History        SF Symbol clock.fill        Material schedule
  Settings       SF Symbol gearshape.fill    Material settings

Sheet
  Connect a pad

Full-screen cover
  Live session

Push
  Session detail
  Goal editor (Next)

Sheet
  Export (Next)
```

Tab roots use a large title that collapses on scroll. Pushed screens use an inline title and a back button. The live session hides the tab bar and does not compete with a large navigation title; the distance is the title.

SF Symbols used in the interface, always with a text label when the symbol is the only content of a button:

| Symbol | Use |
| --- | --- |
| `figure.walk` | Walk, Live Activity |
| `speedometer` | Speed |
| `mountain.2.fill` | Incline |
| `clock` | Time |
| `bluetooth` | Connection |
| `square.and.arrow.up` | Export |
| `antenna.radiowaves.left.and.right` | Scanning |

### 6.1 Today

The tab root. Large title "Today".

- Hero: today's kilometres, two decimal places (`1.25 km`).
- Secondary row: moving time today, and this week's kilometres.
- A segmented control (native `Picker`) for Day, Week, Month, All time. Changing it swaps the hero and the breakdown. The tab title stays "Today" so the tab's job stays obvious; the segment is the range.
- If a walk is in progress, the hero is that walk and the primary button is "Resume".
- Primary button otherwise: "Start walk" if a pad is remembered, "Connect a pad" if not.
- Goal and streak sit below the hero once those features exist. Until then the space is empty, not a disabled placeholder full of explanation.
- Pull to refresh is unnecessary. Data is local. Do not add it.

HIG notes:

- [ ] Semantic system colours. No hard-coded greys that break dark mode.
- [ ] Text uses the system text styles so Dynamic Type, including accessibility sizes, changes the hero. At the largest sizes the hero may wrap; it must not truncate to an ellipsis.
- [ ] The primary button is a real button, at least 44 by 44 points, with a title. Not a tappable card with no button trait.
- [ ] VoiceOver order: range, distance, time, week summary, primary action.
- [ ] Empty state, before any pad: one sentence ("Connect a walking pad to start logging kilometres."), the connect button, and a secondary button for the simulator. No illustration that delays the button.
- [ ] Bluetooth off and permission denied are different empty states, each with the single next step (open Settings, or turn Bluetooth on).

### 6.2 Connect a pad

A sheet, not a new tab. Title "Connect a pad". Dismiss by drag or a Cancel button.

- List of discovered peripherals: advertised name, or "Unnamed pad" plus a short id, plus RSSI as a qualitative label (Strong, OK, Weak) rather than a raw number as the only cue.
- A connecting row shows a native progress indicator and the name.
- On success the sheet closes and Today shows the connected pad.
- "Show other devices" is a second section, off until tapped, for pads that do not advertise `0x1826` (see [Scan strategy](#84-scan-and-connect)).

HIG notes:

- [ ] Explain why Bluetooth is needed in the sheet body, in one line, before the system prompt has been accepted. The iOS purpose string is a single sentence; the sheet can be slightly more specific.
- [ ] Do not show a custom alert that duplicates the system permission alert.
- [ ] Sorting is by signal, then name. Do not reorder the list every second; batch visual updates.
- [ ] Announce new results politely to VoiceOver. Do not re-announce the whole list on each advertisement.

### 6.3 Live session

Full-screen cover. Opened from Today when a walk starts, or when the owner taps Resume.

- Distance in km, the largest text on the screen.
- Speed in km/h with one decimal place.
- Moving time and elapsed time.
- Incline in percent with one decimal place, only when the session has received a real inclination. Hidden when unknown. `0.0%` only when the pad reported zero.
- Connection: "Connected", "Reconnecting", or "Pad out of range", each with a symbol and a word. Colour is not the only signal.
- Primary controls: Pause and Stop while moving; Resume and Stop while paused. Stop always confirms if the walk is long enough to keep, and offers Discard when it looks like a nudge.
- A small line of source text in the details disclosure, not in the hero: "Distance from pad" or "Distance estimated from speed".

HIG notes:

- [ ] `expo-keep-awake` is active for the lifetime of this screen and released when the session ends.
- [ ] Respect Reduce Motion. The distance updates in place. No counting animation that ignores the setting.
- [ ] Haptics (Next): a light impact on start and pause, a notification success on a clean stop, a warning on safety-key stop. No haptics on every BLE notification.
- [ ] Controls stay reachable with one thumb. The distance occupies the top; the buttons sit in the bottom safe area.
- [ ] VoiceOver: when distance changes by 0.01 km, announce the new distance. Do not announce every speed tick.
- [ ] Dark mode uses the same semantic colours. The screen is readable in a dim room under a desk.

### 6.4 History

Tab root. Large title "History". Inset grouped list, sections by local day ("Today", "Yesterday", then the date).

- Row: distance, moving time, pad name, a walk symbol. Incline icon only if that session has incline.
- Tap pushes Session detail.
- Swipe actions, once Next ships: Delete, Export. Delete confirms.
- Empty state: "Walks you finish will show up here."

HIG notes:

- [ ] Grouped list style, not a custom card stack.
- [ ] Day headers use the locale's date format.
- [ ] Destructive delete uses the system destructive style and a confirmation dialog.

### 6.5 Session detail

Pushed. Inline title is the day and time the walk started.

- Distance, moving time, elapsed time, average speed, max speed.
- Incline average and max when known.
- Heart rate average only when the pad sent it.
- Distance source, and "Edited" if the owner corrected it.
- Notes (Next).
- Share (Next) presents the export sheet for this session.

No map. There is no GPS track.

### 6.6 Goals

Not a tab. Reached from Today once Next ships. A form with the system grouped style: daily kilometres, weekly kilometres, and a plain-language streak rule ("A day counts when a kept walk exists. A nudge you discard does not.").

### 6.7 Settings

Tab root. Large title "Settings". Grouped form.

- Connected pad, with Forget.
- Auto start, and the stop debounce (a stepper in seconds).
- Units: kilometres. Present as the current unit, not as a hidden assumption. A miles option is out of scope until asked; the control can be a single disabled-looking value "Kilometres" so the choice is visible.
- Include practice walks in totals. Default off.
- Export all (Next).
- Apple Health / Health Connect row (Later), off, with status.
- Strava row (Later), off.
- Belt control row (Later), off, with an explanation.
- Simulator (dev builds, and a "Practice walk" entry so the owner can try the UI before hardware).
- About: version, and a short privacy line ("Walks stay on this device.").

### 6.8 Web layout

The same three routes. On a browser without Web Bluetooth (Safari on iPhone, Safari on Mac, Firefox), Today explains that connecting a pad needs the iPhone or Android app, or Chrome, Edge, or Opera on a computer. History, totals, export, and the simulator still work.

On Chromium, "Connect a pad" calls `navigator.bluetooth.requestDevice` directly from the click handler. The browser chooser replaces the scan list. There is no continuous scan on the web.

### 6.9 Dynamic Island and Lock Screen (Next)

Built with `expo-widgets` `createLiveActivity` and Expo UI SwiftUI views. Started when a session becomes active. Updated from the foreground. `staleDate` is about 15 seconds after the last sample, so a dropped connection does not keep a fresh-looking number.

| Presentation | Content |
| --- | --- |
| Minimal | `figure.walk` |
| Compact leading | `figure.walk` |
| Compact trailing | distance in km |
| Expanded | distance, speed, moving time, incline if known |
| Lock Screen banner | the same expanded content |

Tapping the activity opens the session URL and the app routes to the live screen. Android has no Dynamic Island. The equivalent, in the Later background phase, is the foreground-service notification.

## 7. FTMS protocol

Normative sources:

- Bluetooth SIG Fitness Machine Service specification, revision FTMS.TS.p6 (2024-10-08). Field behaviour, control point, and status opcodes below follow that text.
- Bluetooth SIG GATT Specification Supplement entry `org.bluetooth.characteristic.treadmill_data`, as published in the SIG assigned-numbers repository (read 25 November 2026, commit `805932b`). Field widths and units follow that YAML, with one deliberate exception for pace width noted in [Pace width](#75-pace-width).

All multi-byte integers are little-endian. The client parses with a cursor. Fixed offsets are wrong as soon as any optional field is absent.

### 7.1 Services and characteristics

| Name | UUID | Use in ftmsLedger |
| --- | --- | --- |
| Fitness Machine Service | `0x1826` | Required. A pad without it is not an MVP pad. |
| Fitness Machine Feature | `0x2ACC` | Read once after connect. Two little-endian `uint32` masks. |
| Treadmill Data | `0x2ACD` | Notify. The live sample stream. Also readable; a read at disconnect is a nice-to-have summary, not the log. |
| Training Status | `0x2AD3` | Read and notify when present. |
| Supported Speed Range | `0x2AD4` | Read when present. Mandatory in the spec if speed target setting is supported. |
| Supported Inclination Range | `0x2AD5` | Read when present. Mandatory in the spec if inclination target setting is supported. |
| Fitness Machine Control Point | `0x2AD9` | Write and indicate. Later, and only for control. Enable indications before any write. |
| Fitness Machine Status | `0x2ADA` | Notify when present. Spec requires it if the Control Point exists. |
| Device Information | `0x180A` | Optional read of manufacturer and model, to label generic names such as "FTMS". |

16-bit UUIDs are the Bluetooth base UUID `0000XXXX-0000-1000-8000-00805F9B34FB`.

### 7.2 Fitness Machine Feature (`0x2ACC`)

Eight bytes. First `uint32` is machine features. Second `uint32` is target-setting features. Bits the app cares about:

| Mask | Bit | Meaning |
| --- | --- | --- |
| Features | 0 | Average speed |
| Features | 2 | Total distance |
| Features | 3 | Inclination |
| Features | 4 | Elevation gain |
| Features | 9 | Expended energy |
| Features | 10 | Heart rate |
| Features | 12 | Elapsed time |
| Features | 13 | Remaining time |
| Target | 0 | Speed target setting. Required before any speed control. |
| Target | 1 | Inclination target setting. Required before any incline control. |

Store both raw masks on the device row. Absence of a feature bit means the live screen must not expect that field. A pad may still send a field the feature bit forgot; the parser trusts the Treadmill Data flags for each notification, and the feature bits decide which controls to offer.

### 7.3 Treadmill Data flags (`0x2ACD`)

The first two bytes are a `uint16` flag field. Bit 0 is inverted relative to every other bit: `0` means instantaneous speed is present.

| Bit | Name | When 1, unless noted |
| --- | --- | --- |
| 0 | More Data | When **0**, Instantaneous Speed is present. When **1**, it is omitted (this notification continues a split record). |
| 1 | Average Speed present | Average Speed |
| 2 | Total Distance present | Total Distance |
| 3 | Inclination and Ramp Angle Setting present | Inclination and Ramp Angle Setting |
| 4 | Elevation Gain present | Positive Elevation Gain and Negative Elevation Gain |
| 5 | Instantaneous Pace present | Instantaneous Pace |
| 6 | Average Pace present | Average Pace |
| 7 | Expended Energy present | Total Energy, Energy per Hour, and Energy per Minute |
| 8 | Heart Rate present | Heart Rate |
| 9 | Metabolic Equivalent present | Metabolic Equivalent |
| 10 | Elapsed Time present | Elapsed Time |
| 11 | Remaining Time present | Remaining Time |
| 12 | Force on Belt and Power Output present | Force on Belt and Power Output |
| 13–15 | Reserved | Reject the notification if a reserved bit is set and the cursor would run off the buffer. Log the raw bytes. |

Fields are consumed in the order below, and only when their bit says they are present. "Not available" sentinels are skipped, not stored as zero.

| Order | Field | Type | Unit |
| --- | --- | --- | --- |
| 1 | Instantaneous Speed | `uint16` | 0.01 km/h |
| 2 | Average Speed | `uint16` | 0.01 km/h |
| 3 | Total Distance | `uint24` | 1 metre, since the pad's training session started |
| 4 | Inclination | `sint16` | 0.1 percent. Positive is uphill. `0x7FFF` means not available. |
| 5 | Ramp Angle Setting | `sint16` | 0.1 degree. `0x7FFF` means not available. |
| 6 | Positive Elevation Gain | `uint16` | 0.1 metre |
| 7 | Negative Elevation Gain | `uint16` | 0.1 metre |
| 8 | Instantaneous Pace | see below | see below |
| 9 | Average Pace | see below | see below |
| 10 | Total Energy | `uint16` | kilocalorie. `0xFFFF` means not available. |
| 11 | Energy per Hour | `uint16` | kilocalorie. `0xFFFF` means not available. |
| 12 | Energy per Minute | `uint8` | kilocalorie. `0xFF` means not available. |
| 13 | Heart Rate | `uint8` | beats per minute |
| 14 | Metabolic Equivalent | `uint8` | 0.1 MET |
| 15 | Elapsed Time | `uint16` | seconds since the pad's training session started. Wraps at 65535. |
| 16 | Remaining Time | `uint16` | seconds |
| 17 | Force on Belt | `sint16` | newton. `0x7FFF` means not available. |
| 18 | Power Output | `sint16` | watt. `0x7FFF` means not available. |

`uint24` distance is three bytes, low byte first. Maximum representable value is 16,777,215 m. That is large enough that overflow is not a product concern; a backwards jump is a reset, not an overflow.

The spec recommends notifications about once per second. The interval is not configurable. The app must not assume a 1.000 s period.

Request a larger ATT MTU after connect (185, or the platform maximum). A default payload of 20 bytes cannot hold every field at once. Many budget pads send a short subset (flags, speed, distance, maybe incline, elapsed time) that fits in 20 bytes. Both shapes must parse.

### 7.4 Split notifications

Section 4.19 of the service spec allows one logical record to arrive as several notifications when it does not fit the MTU. The More Data bit marks the split: the first notification of a record has More Data = 0 and includes instantaneous speed; a continuation has More Data = 1 and omits instantaneous speed. When a record is split, elapsed time is required on each piece so the client can tell which record a continuation belongs to.

Reassembly:

- [ ] A notification with More Data = 0 starts a new record and carries speed.
- [ ] A notification with More Data = 1 appends fields to the open record.
- [ ] If no record is open, drop the continuation.
- [ ] Match on elapsed time when both pieces have it. If elapsed time disagrees, discard the open partial and keep the notification that contains speed.
- [ ] A partial record older than 2 seconds is discarded.
- [ ] The accumulator sees only complete records.

### 7.5 Pace width

Speed, distance, incline, elevation, and the flags are the same in the FTMS service spec and in the current SIG YAML. Pace is not.

The YAML read on 25 November 2026 types Instantaneous Pace and Average Pace as `uint16`, described as time per 500 metres, in seconds. Older FTMS implementations, and the field layouts budget firmware is likely to have copied, use a `uint8` pace in 0.1 km/min. Elapsed time sits after pace. If the pace bit is set and the width is wrong, elapsed time, heart rate, and everything after pace shift by a byte. Distance and incline sit before pace, so they stay valid either way.

Parser rule:

- [ ] Parse speed, distance, incline, and elevation before pace. Those fields never depend on the pace guess.
- [ ] Try `uint16` pace first, then `uint8`, and keep the width for which the remaining flagged fields consume the buffer exactly.
- [ ] If both widths fit (the buffer is ambiguous), prefer the width already locked for this connection. Until one locks, prefer `uint8` on the first session of an unknown pad, because that is the layout seen on the class of pad being shopped for, and record the choice on the device row once a later notification disambiguates it.
- [ ] Session duration does not depend on this guess. The phone's monotonic clock is the source of truth for moving time and elapsed time. Device elapsed time is a cross-check.

### 7.6 Ranges

Supported Speed Range (`0x2AD4`), three `uint16` values, each 0.01 km/h: minimum, maximum, minimum increment.

Supported Inclination Range (`0x2AD5`), three `sint16` values, each 0.1 percent: minimum, maximum, minimum increment.

Read them for the device profile and, later, to clamp control commands. Do not invent a range if the read fails.

### 7.7 Training Status (`0x2AD3`)

Flags `uint8`, then status `uint8`, then an optional UTF-8 string when flag bit 0 is set.

| Value | Meaning |
| --- | --- |
| `0x00` | Other |
| `0x01` | Idle |
| `0x02` | Warming Up |
| `0x03` | Low intensity interval |
| `0x04` | High intensity interval |
| `0x05` | Recovery interval |
| `0x0B` | Cool Down |
| `0x0D` | Manual mode (quick start) |
| `0x0E` | Pre-workout |
| `0x0F` | Post-workout |

Other defined values (isometric, heart-rate control, fitness test, watt control, speed outside control region) are stored and otherwise ignored. Idle, pre-workout, and post-workout mean "not walking". Manual mode and the interval values mean "walking" when speed agrees. Training Status alone never starts a session: a pad can sit in Manual mode with the belt stopped.

### 7.8 Fitness Machine Status (`0x2ADA`)

One opcode byte plus a parameter. Notifications the session engine acts on:

| Opcode | Meaning | App action |
| --- | --- | --- |
| `0x02` | Stopped or paused by the user. Parameter `uint8`: `0x01` stop, `0x02` pause. | Pause or stop the session. |
| `0x03` | Stopped by the safety key. | Stop immediately. No debounce. |
| `0x04` | Started or resumed by the user. | Start or resume, if speed follows within a few seconds. |
| `0xFF` | Control permission lost. | Relevant only in the control phase. |

Other opcodes (`0x01` reset, `0x05` target speed changed, `0x06` target incline changed, and the rest of table 4.26) are logged on the sample or a debug ring and do not change session state, except that a target-incline change can update the displayed incline target. It is not a measured incline.

Many pads never notify this characteristic. Speed is the signal that must work alone.

### 7.9 Control Point (`0x2AD9`), Later only

Write with response, and enable indications before the first write. The spec returns an error if indications are not enabled. One procedure at a time; a second write while the first is open gets "Procedure Already In Progress".

| Opcode | Payload | Use |
| --- | --- | --- |
| `0x00` | none | Request Control. Must succeed before any other command. |
| `0x01` | none | Reset. Do not send this casually. It zeros the pad's elapsed time. |
| `0x02` | `uint16` speed, 0.01 km/h | Set target speed. |
| `0x03` | `sint16` incline, 0.1 percent | Set target inclination. |
| `0x07` | none | Start or resume. |
| `0x08` | `uint8`: `0x01` stop, `0x02` pause | Stop or pause. |

The indication is `0x80`, then the request opcode, then a result code:

| Code | Meaning |
| --- | --- |
| `0x01` | Success |
| `0x02` | Opcode not supported |
| `0x03` | Invalid parameter |
| `0x04` | Operation failed |
| `0x05` | Control not permitted |

Safety rules for the phase that adds control:

- [ ] The feature is off unless the owner turns it on in Settings and confirms a warning that the belt can move.
- [ ] Never write the Control Point during a normal logging connection.
- [ ] Request Control immediately before a command, and again after `0xFF` Control Permission Lost. Permission also ends when the connection drops or when the client sends Reset.
- [ ] Clamp speed and incline to the supported range and to the minimum increment. If the range read failed, refuse the command.
- [ ] Wait for the indication. A write response is not success.
- [ ] Timeout the wait at 3 seconds and show the failure. Do not retry a speed command automatically.
- [ ] The on-screen control is a slider or stepper that sends on release, not on every tick.
- [ ] Stop (`0x08 0x01`) is always available while control mode is on, including if a set-speed command fails.

## 8. Session engine

The engine is a pure reducer over samples plus a clock. BLE, SQLite, and React sit outside it. That is what makes the reset and reconnect cases unit-testable.

### 8.1 What a walk is

A session opens when the belt is considered moving, and closes when it is stopped. The phone clock defines time:

- Moving time accumulates only while the state is moving.
- Elapsed time runs from the first start through the final stop, and keeps running through pauses, so the owner can see that a 30-minute visit included a break.
- Device elapsed time is stored on samples as a cross-check. It is not the session clock. It resets when the pad thinks a new training session started, and it wraps at 65535 seconds.

### 8.2 Auto start and stop

Precedence, highest first:

1. Safety key (`0x2ADA` opcode `0x03`): stop now.
2. Status stop (`0x02` / `0x01`) or pause (`0x02` / `0x02`): stop or pause now.
3. Status start (`0x04`): arm a start. The session actually opens when the next samples show motion, so a status blip with a stopped belt does not log an empty walk.
4. Speed, which always works even if (2) and (3) never arrive.

Speed defaults, all configurable in Settings:

| Threshold | Default |
| --- | --- |
| Moving at or above | 0.4 km/h |
| Stopped at or below | 0.1 km/h |
| Samples required to start | 2 consecutive moving samples |
| Pause debounce | 20 seconds continuously stopped |
| Auto-finish after pause | 2 minutes still paused, or the owner taps Stop |

A pause keeps the same session. Speed rising above the moving threshold resumes it and clears the pause timer. Finishing writes the session and returns to Today.

The owner can tap Start early (the screen opens, moving time stays at zero until the belt moves) and can tap Stop at any time.

### 8.3 Distance

Internal distance is an integer number of metres plus a remainder, so slow speeds do not quantise to zero. Display kilometres with two decimal places (0.01 km = 10 m). The remainder is still stored.

On each complete sample the engine takes `dt` from the monotonic clock since the previous accepted sample, clamped to 0–3000 ms. A notification gap longer than 3 seconds does not invent a straight-line distance for the hole; those metres are recovered only if the pad's own counter advanced across the hole by a plausible amount.

Prefer the pad's Total Distance when it is behaving. Integrate speed only to fill gaps.

Let `device` be the `uint24` metres, when the flag is set.

- First sample of a connection that contains `device`: set the baseline to that value and add 0. The spec's counter is "since the pad's training session started", which may already be non-zero if ftmsLedger connected late. Metres walked before connect are not guessed. This is an accepted limit.
- `device` greater than the baseline by a plausible step: add the delta. Plausible means the step is at most `max(50 m, 1.5 × speed × dt)` using the maximum of the reported speed and the last speed. A larger jump is ignored and the source switches to integration for that interval.
- `device` equal to the baseline while speed says the belt is moving, for more than 3 seconds: the counter is frozen or the pad only emits distance while it feels like it. Integrate speed until `device` advances again.
- When a frozen counter later jumps, compare the jump with the metres just integrated. If they agree within 20 percent, replace the integrated stretch with the pad's delta so the pad stays authoritative. If the pad jumps by a much smaller or larger amount, keep the integrated metres and mark the session source as mixed.
- `device` less than the baseline: the counter reset (belt stop, new training session, or a reconnect). Add `device` if it is a small fresh counter (under the plausibility ceiling above). Do not subtract. Set the baseline to `device`.
- `device` absent: integrate speed for this `dt`.

Speed integration, in integers:

```
delta_millimetres = speed_centikmh * dt_ms / 36
```

because `0.01 km/h × dt_ms / 1000 × 1000 m/km / 3600 s/h`, scaled to millimetres, simplifies to `speed_centikmh * dt_ms / 36`. Carry the division remainder into the next sample. Add to the session metres when the millimetre accumulator crosses 1000.

`distance_source` on the session is `device`, `integrated`, or `mixed`.

Worked cases the tests must cover:

| Case | Expected |
| --- | --- |
| Steady counter, 1.00 km/h, one notification per second, for 60 s | 16 or 17 m (1 km/h is 0.278 m/s). Source `device` if the counter moved, else `integrated`. |
| Counter present only on moving samples, frozen while stopped | No extra metres during the stop. Source stays `device` if the counter resumes from the same value. |
| Counter resets to 0 every time the belt stops, two walking bouts | Session metres are the sum of both bouts. |
| Counter resets to 0 on reconnect, session still open | Metres before the drop are kept. New deltas add on top. |
| Counter frozen at 400 m while speed is 3 km/h | Integration takes over within 3 s. A later catch-up jump near the integrated amount replaces that stretch. |
| Single sample jumps by 2 km | Jump ignored. No 2 km credited. |
| Connect mid-walk, first distance already 800 m | Those 800 m are the baseline. They are not added. |
| Incline flag never set | `incline_available` is false. UI hides incline. |
| Incline `0x7FFF` | Treated as unknown for that sample, not as a huge percent. |
| Incline `0` | Shown as 0.0%. |

Average incline is the time-weighted mean of samples where incline is known. Average speed is session metres divided by moving time, and the displayed "average from the pad" is kept separately when the pad sends Average Speed, so a disagreement is visible in the detail disclosure.

### 8.4 Scan and connect

Budget pads often omit `0x1826` from the advertising packet and only reveal it after connection. A scan filtered solely on the service UUID misses those pads.

- [ ] First pass: scan for peripherals advertising `0x1826`.
- [ ] The visible list also includes strong advertisers whose name is non-empty, behind "Show other devices", so a pad advertising only a name can be chosen.
- [ ] On connect, discover services. If `0x1826` is missing, disconnect, tell the owner this pad did not expose the standard walking service, and store the service UUIDs that were seen (for example `FFF0`) on a local rejected-device note. Do not speak a vendor protocol.
- [ ] Request the larger MTU, then read `0x2ACC`, then the range characteristics that exist, then enable notifications on `0x2ACD` and the optional status characteristics.
- [ ] One central connection. The copy should say to disconnect the vendor app first. Most of these pads accept a single client.
- [ ] Remember the last pad: on iOS the peripheral identifier is stable per app install; on Android the address may be a random private address until the device is bonded, and many pads will not bond. Also store the advertised name. Reconnect tries the saved identifier first, then a short scan for the saved name.
- [ ] On the web there is no scan loop. `requestDevice({ filters: [{ services: [0x1826] }] })` runs inside the click. Where `navigator.bluetooth.getDevices` exists, offer the previously permitted device without a new chooser.

If the app is killed mid-walk, the session row is left `active`. On next launch, if the same pad is back within 15 minutes, offer Resume; otherwise close the session with the metres already saved and the end time set to the last sample. Never silently attach tomorrow morning's walk to yesterday's row.

### 8.5 Samples

Persist a sample at most once per second, and also whenever incline changes or the belt transitions between moving and stopped. A one-hour walk is about 3600 rows, which SQLite handles easily, and it is enough for a sparkline. Do not store every duplicate notification.

## 9. Data model and local storage

`expo-sqlite`, WAL mode, one database file `ftmsledger.db` in the app's document directory. All timestamps are Unix milliseconds. All distances are integer metres. Speeds are integer centi-km/h (1.00 km/h is `100`). Incline is integer deci-percent (1.5% is `15`). No floating point in the database.

Day, week, and month totals are queries, not a second stored ledger. A day is the phone's local calendar day. A week starts on Monday (ISO). Month is the local calendar month. All-time is every kept session. The active session is included in Today so the hero moves during a walk. Simulator sessions are excluded unless Settings says otherwise.

### 9.1 Tables

```sql
-- schema_migrations
-- version INTEGER PRIMARY KEY
-- applied_at INTEGER NOT NULL

-- devices
-- id TEXT PRIMARY KEY                -- app uuid, stable
-- platform_id TEXT                   -- CBPeripheral UUID or Android address; may change
-- name TEXT
-- last_seen_at INTEGER
-- feature_mask INTEGER NOT NULL      -- first uint32
-- target_mask INTEGER NOT NULL       -- second uint32
-- pace_width INTEGER                 -- 1 or 2 once disambiguated, else NULL
-- speed_min_centi INTEGER
-- speed_max_centi INTEGER
-- speed_step_centi INTEGER
-- incline_min_deci INTEGER
-- incline_max_deci INTEGER
-- incline_step_deci INTEGER
-- created_at INTEGER NOT NULL

-- sessions
-- id TEXT PRIMARY KEY
-- device_id TEXT REFERENCES devices(id)
-- status TEXT NOT NULL               -- 'active' | 'paused' | 'completed' | 'discarded'
-- source TEXT NOT NULL               -- 'ble' | 'simulator'
-- started_at INTEGER NOT NULL
-- ended_at INTEGER
-- distance_m INTEGER NOT NULL
-- moving_time_ms INTEGER NOT NULL
-- elapsed_time_ms INTEGER NOT NULL
-- max_speed_centi INTEGER
-- incline_available INTEGER NOT NULL -- 0 or 1
-- avg_incline_deci INTEGER
-- max_incline_deci INTEGER
-- distance_source TEXT NOT NULL      -- 'device' | 'integrated' | 'mixed'
-- edited INTEGER NOT NULL            -- 0 or 1
-- notes TEXT
-- health_sync TEXT NOT NULL          -- 'off' | 'pending' | 'synced' | 'failed'
-- health_external_id TEXT            -- sample or workout id, to keep retries idempotent
-- strava_sync TEXT NOT NULL          -- 'off' | 'pending' | 'synced' | 'failed'
-- strava_activity_id TEXT
-- created_at INTEGER NOT NULL
-- updated_at INTEGER NOT NULL

-- samples
-- id INTEGER PRIMARY KEY AUTOINCREMENT
-- session_id TEXT NOT NULL REFERENCES sessions(id)
-- mono_ms INTEGER NOT NULL           -- ms from session start, monotonic
-- speed_centi INTEGER
-- distance_m INTEGER NOT NULL        -- session cumulative
-- incline_deci INTEGER               -- NULL if unknown
-- heart_rate INTEGER
-- device_distance_m INTEGER          -- raw uint24, NULL if absent
-- device_elapsed_s INTEGER
-- flags INTEGER NOT NULL

-- settings
-- key TEXT PRIMARY KEY
-- value TEXT NOT NULL
```

Indexes: `sessions(started_at)`, `sessions(status)`, `samples(session_id, mono_ms)`.

Totals query, conceptually: sum `distance_m` and `moving_time_ms` over sessions whose `status` is `completed` or `active`, whose `source` is included, and whose local day (or week, or month) matches the range. Discarded sessions stay in the database so an accidental discard can be undone from History for a while, but they are absent from totals. A later cleanup can purge discarded rows older than 30 days.

### 9.2 Migrations

- [ ] `PRAGMA user_version` is the only schema version. `schema_migrations` records when each version was applied.
- [ ] Version 1 is the tables above, created inside a transaction.
- [ ] Each later change is a new file, `src/db/migrations/00N.sql`, applied in order. Never edit a migration that has shipped.
- [ ] The migrator runs before any repository call. A failure blocks launch with a short message and does not delete the file.
- [ ] Web uses the same SQL. If `expo-sqlite` on web needs `configureWeb()` and a `wa-sqlite` build, that setup is part of the MVP web milestone, not a fork of the schema.

### 9.3 Export shape (Next)

CSV, one row per session, header included:

`id,started_at_iso,ended_at_iso,distance_km,moving_time_s,elapsed_time_s,avg_speed_kmh,max_speed_kmh,avg_incline_percent,max_incline_percent,distance_source,source,notes`

JSON is an array of the same objects plus a `samples` array when exporting a single session. Dates are ISO 8601 in the local offset. Distance in the file is decimal kilometres with three places, produced from integer metres, so a round trip does not gain a metre.

Write the file with `expo-file-system`, then open the share sheet with `expo-sharing`. On web, trigger a download; call the Web Share API only when `Sharing.isAvailableAsync()` (or `navigator.canShare`) says so. Sharing a local file URI is not supported on web, so the web path shares bytes, not a file URI.

## 10. Architecture and folders

The Expo Router tree and the domain code stay apart. The domain code does not import React or a BLE library.

```
app/
  _layout.tsx                 root stack
  (tabs)/
    _layout.tsx               NativeTabs
    index.tsx                 Today
    history.tsx
    settings.tsx
  session/[id].tsx            live session and, with a query, the detail
  devices.tsx                 connect sheet
  goals.tsx                   Next
src/
  ftms/
    uuids.ts
    parseTreadmillData.ts     cursor parser, pure
    parseFeature.ts
    parseStatus.ts
    parseRanges.ts
    controlPoint.ts           encode/decode only; no writes
    types.ts
  session/
    engine.ts                 reducer: sample in, state out
    distance.ts               baseline, reset, frozen, integrate
    detect.ts                 start, pause, stop
  ble/
    transport.ts              interface
    plxTransport.ts           @sfourdrinier/react-native-ble-plx
    webBluetoothTransport.ts  navigator.bluetooth
    simulatorTransport.ts     emits Uint8Array notifications
    ftmsClient.ts             connect, subscribe, map bytes to samples
  db/
    client.ts
    migrations/
    sessions.ts
    devices.ts
    totals.ts
  export/                     Next
  health/                     Later
  components/                 @expo/ui only
```

`FtmsTransport` is the seam:

- scan or request a device
- connect and disconnect
- read a characteristic
- subscribe to notifications
- write a characteristic (control phase)

`FtmsClient` depends on the transport and on the parsers. The session engine depends only on parsed samples. Screens depend on the engine's snapshot.

The simulator transport builds the same bytes the parser reads, including a flags field. It does not push pre-parsed objects into the engine. Scenarios, selectable from Settings in a dev build:

- [ ] Steady 3.0 km/h, distance counting, no incline.
- [ ] Same, with incline rising from 0% to 5%.
- [ ] Distance omitted; speed present. Forces integration.
- [ ] Distance frozen while speed stays high, then a catch-up jump.
- [ ] Counter reset to 0 mid-walk.
- [ ] Disconnect for 5 seconds and reconnect with the counter restarted.
- [ ] Split record: More Data 0, then More Data 1.
- [ ] Both pace widths, with elapsed time after pace, to lock the disambiguation test.
- [ ] Safety-key status opcode.

Practice-walk sessions use `source = 'simulator'`.

## 11. Permissions

Copy is specific. "Bluetooth is used to connect to your walking pad and read distance, speed, and time." Do not mention location. The app does not use location.

### 11.1 iOS

Set through the Expo config, not a hand-edited `Info.plist` (native projects are generated):

- [ ] `NSBluetoothAlwaysUsageDescription`: "ftmsLedger uses Bluetooth to read distance, speed, and time from your walking pad."
- [ ] Do not add a location usage string in the MVP. Adding one causes a location prompt the app does not need.
- [ ] Health, when that phase starts: `NSHealthUpdateUsageDescription` for writing walks, and share usage only if a future feature reads Health. Update-only for the planned write.
- [ ] Bluetooth background mode stays off until the background milestone. The plugin's `iosEnableRestoration` stays false.

The system prompt appears when the manager is first created, which should be the moment the owner taps Connect, not at first launch. Today is usable before Bluetooth is granted (history, simulator).

### 11.2 Android

The BLE library's minimum is Android 8 (API 24). Compile and target SDK follow the Expo SDK 58 / library default (the 3.9.x line compiles against target SDK 36). Do not lower `minSdk` below 24.

| Permission | When |
| --- | --- |
| `BLUETOOTH_SCAN` | Android 12+. Declare `neverForLocation`. The scan is not used to infer where the owner is. |
| `BLUETOOTH_CONNECT` | Android 12+. Required to connect and to discover GATT. |
| `ACCESS_FINE_LOCATION` | Android 6 through 11 only, because those versions require it for a BLE scan. Do not request it on Android 12+. |
| `POST_NOTIFICATIONS` | Later, for the foreground-service notification. |
| `FOREGROUND_SERVICE` and `FOREGROUND_SERVICE_CONNECTED_DEVICE` | Later, for screen-off logging. |

Request scan and connect together, from the Connect button, with a one-line rationale in the sheet before the system dialog. If the owner denies, the sheet shows how to enable Bluetooth in system settings and does not loop the dialog.

`neverForLocation` remains correct even though the scan cannot always filter on `0x1826`: the flag means the app will not derive location from scan results, not that a service filter is required.

### 11.3 Web

Web Bluetooth requires a secure context (HTTPS, or localhost). `requestDevice` must run in the user-gesture turn. There is no permission the app can pre-request. iOS Safari does not implement Web Bluetooth; iPhone use is the native app.

## 12. Background behaviour

MVP assumption: the phone is awake and ftmsLedger is on screen for the walk. `expo-keep-awake` holds the display. That matches an under-desk pad sitting in front of the phone.

iOS limits, which the UI must not paper over:

- A JavaScript timer does not keep running once iOS suspends the app.
- `bluetooth-central` background mode can deliver a pending connection and some notifications to a native central, but the React Native JavaScript runtime is not a reliable place to turn those into SQLite writes while suspended.
- Live Activities update when the app updates them. They do not run the BLE stack. `staleDate` is how the Island admits the number has stopped refreshing.
- The honest MVP copy, in the session screen's detail disclosure: "Keep ftmsLedger open during the walk."

Later iOS work, if the foreground flow is solid:

- [ ] Enable the restoration subspec and `getRestoredState()`.
- [ ] On restore, pull any samples the native layer retained and fold them through the same engine.
- [ ] If the fork's background delivery still drops packets while the JS runtime is suspended, stop there and keep the "leave the app open" guidance. Do not add a custom native buffer unless a real pad shows the loss.

Android is more tractable and still explicit:

- [ ] A connected-device foreground service with a persistent notification is required for reliable BLE once the screen is off or the owner leaves the app. That notification shows distance, speed, and time, and its tap returns to the session.
- [ ] The service starts when a session starts and stops when the session ends. No always-on Bluetooth.

Web: the tab is the session. Closing the tab, or the browser discarding it, ends the connection. There is no background BLE. The engine still flushes the session to SQLite (or the web equivalent) on `visibilitychange` and `pagehide`.

## 13. Testing

### 13.1 Parser and engine unit tests

Jest, no device, no emulator. Fixtures are hex byte strings.

- [ ] Each flag bit adds the right width, and a missing bit does not consume a byte.
- [ ] Little-endian `uint24` distance, including values above 65535.
- [ ] Incline sign: positive, negative, zero, and `0x7FFF`.
- [ ] Speed `0x0064` is 1.00 km/h. Speed `0x03E8` is 10.00 km/h.
- [ ] Short buffer with a flag that asks for more bytes returns a parse error and does not throw into the session.
- [ ] Split notifications reassemble; a stray continuation is dropped.
- [ ] Both pace widths, with elapsed time after the pace field, decode to the same elapsed time.
- [ ] Every row of the distance table in [Distance](#83-distance).
- [ ] Start requires two moving samples. One spike does not open a session.
- [ ] Twenty seconds stopped pauses. Speed resumes the same session. Two minutes paused finishes it.
- [ ] Safety key finishes immediately.
- [ ] Remainder carry: 1.00 km/h for 3600 s of 1 s steps lands on 1000 m ± 1 m.

Golden fixtures from a real pad, once one is on the desk, are checked in beside the synthetic ones. A firmware surprise should fail a test, not the owner's monthly total.

### 13.2 Simulator

The scenarios in [Architecture](#10-architecture-and-folders) are the manual and automated stand-in for hardware. At least the steady, reset, frozen, and reconnect scenarios run as engine tests by feeding simulator bytes through the real parser.

### 13.3 Real-device checklist (nRF Connect, then the app)

Do this once per pad model, before trusting it. Use nRF Connect on the phone. Disconnect the vendor app first.

- [ ] Pad powered, phone Bluetooth on. Note the advertised name and whether `0x1826` appears in the advertisement.
- [ ] Connect. Confirm primary service `0x1826`.
- [ ] Read `0x2ACC`. Write down both 32-bit masks.
- [ ] Enable notify on `0x2ACD`. Start the belt. Save five to ten notifications as hex.
- [ ] Stop the belt. Save the next notifications. Record whether distance keeps counting, freezes, disappears, or resets to zero.
- [ ] Read `0x2AD4` and `0x2AD5` if present.
- [ ] Read `0x2AD3`. Note idle versus manual.
- [ ] Enable notify on `0x2ADA` if present. Start and stop from the pad's own button. Record opcodes.
- [ ] Do not write `0x2AD9` on this first pass.
- [ ] If `0x1826` is absent, write down every service UUID, especially `FFF0` / `FFF1` / `FFF2`, `FE00`, and `1234`. Stop. That pad is outside the MVP. The UUIDs decide whether a later fallback is worth it.
- [ ] Paste the hex into parser fixtures. Confirm the app's live distance stays within one percent of the pad console over a 10-minute walk, or that the difference is explained by the documented baseline rule.
- [ ] Walk away with the phone until the link drops, come back, and confirm the session resumes without a doubled total.
- [ ] Repeat with the vendor app force-quit, and confirm the pad accepts the new connection.

### 13.4 App checks before a phase is done

- [ ] iPhone: Dynamic Type at the largest accessibility size, dark mode, VoiceOver on Today and the live session, Reduce Motion on.
- [ ] Android: a 12+ device (scan without a location prompt) and, if one is available, an Android 11 device (location prompt only on connect).
- [ ] Web: Chrome connect flow on localhost, plus Safari showing the unsupported-Bluetooth empty state and still opening history.
- [ ] Kill the app mid-session and relaunch. Resume or close behaves as specified.
- [ ] A simulator walk does not change the all-time total while "include practice walks" is off.

## 14. Privacy

- [ ] No account. No sign-in screen. No server.
- [ ] The database stays in the app sandbox. Uninstalling the app removes it. Say so in Settings → About.
- [ ] No analytics, no crash reporter, and no advertising SDK in the MVP. If a crash reporter is added later, it is opt-in and it does not receive workout rows, Bluetooth identifiers, or health data.
- [ ] Bluetooth traffic is between the phone and the pad. The app does not forward notifications anywhere.
- [ ] Export happens only when the owner taps Share. The share sheet is the operating system's. The app does not choose a destination.
- [ ] Apple Health and Health Connect writes happen only after an explicit toggle, and only for walks completed after the toggle. Existing walks are not bulk-uploaded unless the owner asks.
- [ ] Strava is the exception to "no account", and it is off. The token lives in `expo-secure-store`. Disconnect deletes the token. Failed uploads stay on the phone.
- [ ] Logs that might contain raw GATT bytes exist only in dev builds.

## 15. Commit conventions

[Conventional Commits](https://www.conventionalcommits.org/). One concern per commit.

```
<type>(<scope>): <imperative summary>
```

Types: `feat`, `fix`, `docs`, `test`, `refactor`, `perf`, `chore`, `build`, `ci`.

Scopes: `ftms`, `ble`, `session`, `ui`, `db`, `export`, `health`, `strava`, `sim`, `web`.

Examples:

```
feat(ftms): parse treadmill data flags and uint24 distance
fix(session): keep metres when the pad counter resets
test(session): cover frozen distance and speed integration
docs: correct the pace-width note
```

The summary names ftmsLedger only when the subject would otherwise be ambiguous. User-visible copy inside the diff still says `ftmsLedger`.

Do not mix a protocol fix and a screen restyle in one commit. A migration is its own `feat(db)` commit, and it only adds a new version.

## 16. Milestones

Each milestone is done when its acceptance list is true. There is no calendar on these.

### M0 — Plan

- [x] This document is on `main`.
- [ ] The owner has read the open questions in [section 17](#17-open-questions-and-risks) and the defaults stand, or they have been edited.

### M1 — Parser, engine, simulator bytes

No screens required.

- [ ] Unit tests in [section 13.1](#131-parser-and-engine-unit-tests) pass.
- [ ] The simulator scenarios produce bytes that travel through the parser into the engine.
- [ ] A one-hour steady walk at 3 km/h yields 3000 m ± 1 m.

### M2 — Dev build, database, Today, practice walk

- [ ] `npx expo run:ios` produces a dev build named ftmsLedger. Expo Go is not part of the check.
- [ ] Migrations create version 1.
- [ ] A practice walk updates the live screen and, with the setting on, Today's total. With the setting off, the all-time total is unchanged.
- [ ] Dark mode and a larger Dynamic Type size remain readable on Today and the live session.

### M3 — One real FTMS pad on iPhone

- [ ] The nRF Connect checklist is filled in for that pad, and the hex is a fixture.
- [ ] Scan, connect, reconnect, auto start, auto stop, and history all work against that pad.
- [ ] A 10-minute comparison against the pad console is within the rule in the checklist.
- [ ] Forgetting the pad and denying Bluetooth both land on a clear empty state.

### M4 — Android and web parity for the logging loop

- [ ] The same walk flow works on Android 12+ without a location permission prompt.
- [ ] Chrome on a computer can connect to the same pad, log a session, and show it in history.
- [ ] Safari shows the unsupported message and still shows history entered on that browser.

### M5 — Logbook

- [ ] Goals, streaks, CSV/JSON export, share sheet, discard prompt, distance edit, notes, sparkline, haptics, and the Live Activity match [section 4.2](#42-next).
- [ ] An edited distance is what the total and the export use.
- [ ] The Live Activity goes stale after a dropped connection instead of pretending.

### M6 — Health

- [ ] A completed walk writes one walking workout with distance and duration to Apple Health, and the Android equivalent to Health Connect.
- [ ] Repeating the sync does not duplicate the workout.
- [ ] Turning the toggle off writes nothing further. Already written workouts are left alone.
- [ ] Simulator sessions are never written.

### M7 — Only if a pad or a request requires it

- [ ] Control: the safety checklist in [section 7.9](#79-control-point-0x2ad9-later-only) is true, and a command outside the supported range is refused.
- [ ] Background: Android screen-off logging keeps a 10-minute walk. iOS either restores samples through the same engine or the in-app copy still says to leave the app open.
- [ ] FitShow `FFF0`, or another vendor protocol, has its own short plan addendum and its own fixtures. It is not slipped into the FTMS parser.

## 17. Open questions and risks

### 17.1 Open questions

Defaults are in effect until the owner changes them. They are listed so the choice is visible.

| Question | Default |
| --- | --- |
| Which pad will actually be purchased? | Unknown. The app supports a pad that exposes `0x1826`. A HomeFitnessCode model whose vendor app uses FTMS is a good candidate and still gets the nRF Connect checklist. "Kinomap compatible" or "FitShow compatible" is not sufficient. |
| Week starts on which day? | Monday. |
| What counts as a streak day? | Any local day with a kept session. No minimum kilometre. A discarded nudge does not count. |
| Daily goal? | Unset until the owner types one. No invented target. |
| Miles? | Kilometres only. |
| Should the phone move the belt? | No, until a later phase is explicitly requested. |
| Is screen-on logging acceptable for the first version? | Yes. |
| Resume window after the app is killed? | 15 minutes, then the session is closed with the metres already stored. |
| Minimum kept walk? | Prompt to discard under 20 m and under 45 s of moving time. |

### 17.2 Risks

- **Proprietary pads.** A cheap pad may speak only FitShow `FFF0` (service `FFF0`, notify `FFF1`, write `FFF2`, framed as `02 CMD … FCS 03`), or a KingSmith / WalkingPad private service. Those pads will not work in the MVP. The connect flow records the GATT table and stops, which is the input for a later fallback. Do not start that fallback "just in case" before a real rejected pad is on the desk. FitShow's own material also says the module can speak FTMS for apps such as Kinomap; some units will expose both, in which case ftmsLedger uses `0x1826` and ignores `FFF0`.
- **Advertisement without the service UUID.** Handled by the two-step scan. Risk remains that a noisy environment makes "Show other devices" unpleasant. The list is sorted by signal and does not auto-connect.
- **Distance semantics.** The spec says total distance since the training session started. Firmware often resets on every belt stop, omits the field while idle, or reports a lifetime odometer. The accumulator treats the counter as a delta source with a baseline, which is right for all three, and the tests lock the behaviour. A lifetime odometer connected mid-life must not dump years of metres into Today; the baseline rule is what prevents that.
- **Pace width.** A wrong guess shifts elapsed time. It must not shift stored distance. The phone clock is authoritative for duration. See [Pace width](#75-pace-width).
- **Single connection.** The vendor app holding the link looks like "ftmsLedger cannot see the pad". The empty state says to close the other app.
- **iOS background.** Easy to over-promise. The plan does not. Copy and the Live Activity stale date are part of the feature, not polish.
- **Web Bluetooth gaps.** No iOS Safari, no Firefox, no background, chooser instead of a scan list, and a connection that dies with the tab. The native app is the product. Web is a second client for a desk with Chrome.
- **`@expo/ui` gaps.** A control missing from the universal set can tempt a custom React Native stand-in. Prefer a platform file (`swift-ui` / `jetpack-compose`) and a universal web equivalent. Keep Live Activities on Expo UI, because that runtime cannot host arbitrary React Native views.
- **Health library versus SDK 58.** `@appeeky/expo-healthkit` is the candidate because it covers HealthKit and Health Connect and uses the Expo modules API. It may lag an SDK. The fallback is a small Expo module that writes one workout type, not a general Health SDK.
- **Control safety.** A bad write can start a belt unexpectedly. The control phase is gated, clamped, and never on the logging connection. Reset (`0x01`) is not exposed in the UI.
- **Random Android addresses.** Reconnect by saved id, then by name. A renamed pad is connected again from the sheet like a new pad, and its history stays on the old device row. Merging two device rows can wait until it actually happens.
- **New Architecture.** Stay on `@sfourdrinier/react-native-ble-plx` 3.9.x. Do not add the upstream dotintent package and do not adopt the 4.x alpha for the phone build.

## 18. References

- Bluetooth Fitness Machine Service, FTMS.TS.p6 (2024-10-08).
- Bluetooth GATT Specification Supplement, `org.bluetooth.characteristic.treadmill_data` (assigned numbers as of 25 November 2026).
- Expo SDK 58 changelog and the SDK 58 docs for `@expo/ui`, Expo Router native tabs, `expo-sqlite`, `expo-widgets`, and `expo-sharing`.
- `@sfourdrinier/react-native-ble-plx` 3.9.x README, Expo plugin notes, and compatibility table (Expo SDK 57+, React Native 0.86+).
- Apple Human Interface Guidelines: navigation (tab bars, sheets, modality), typography and Dynamic Type, colour and dark mode, accessibility, haptics, and Live Activities.
- Web Bluetooth: `requestDevice` filters, secure context, user gesture. Not available in iOS Safari.
