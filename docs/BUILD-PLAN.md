# Build plan: workouts + a correct clock

Fork of `piyush140104/galaxy-health-bridge` at `dv-brunoh/galaxy-health-bridge`,
branch `workouts-and-timebase`.

Target: **steps and workouts recorded on the watch land in Apple Health**, with no
Android phone, no subscription, no cloud.

---

## What already works

Read this before writing anything. Most of the pipeline is done.

| Piece | Where | State |
| --- | --- | --- |
| Steps / kcal / distance / floors from Health Services | `wearos/health/HealthReader.kt` | works |
| Local store + BLE cursor/stream contract | `wearos/data/SampleStore.kt`, `wearos/ble/Protocol.kt` | works |
| GATT peripheral, MTU packing, notify streaming | `wearos/ble/GattServer.kt` | works |
| Core Bluetooth central, sync state machine | `ios/Services/BLEClient.swift` | works |
| HealthKit writes + uid dedupe | `ios/Services/HealthKitManager.swift` | works for quantities |
| Workout **display** in the dashboard | `ios/Features/Dashboard/HealthSummaryView.swift:503` | already reads `HKWorkout` |

So the gap is exactly one feature, not a rewrite: nothing on the watch starts or
records a workout, and nothing on the iPhone writes an `HKWorkout`.

`CanonicalSampleType.workout` already exists (`ios/Models/CanonicalSample.swift:29`)
but has no `healthKitWriteType` — `HKWorkout` is not an `HKSampleType` subclass you
can hand to `store.save()`. It needs its own write path.

---

## Three constraints that decide the design

### 1. Health Services only gives you workouts your own app is recording

Wear OS has no "read every past workout" API. `PassiveMonitoringClient` gives
steps/kcal/distance/floors 24/7 with no session; `ExerciseClient` streams only while
*your* app owns an active exercise session.

Consequence: **Samsung Health workouts will not appear.** You start workouts in this
app from now on. Samsung Health keeps counting steps independently (harmless — Health
Services' daily aggregates are the source of truth on the wire, and the dashboard
already hides Samsung Health's step source if you demote it in Health).

Also: `ExerciseClient.getCapabilitiesAsync().supportedExerciseTypes` is authoritative
for what this watch/firmware actually supports. Drive the picker from that, not a
hardcoded list.

### 2. The watch clock is wrong, and the current rebase destroys history

`HealthReader` stamps samples with `System.currentTimeMillis()`; an unpaired watch
drifts months. `BLESyncCoordinator.rebaseTimestamps` (`ios/Services/BLESyncCoordinator.swift:133`)
then rewrites any sample more than an hour off to `now`, preserving only duration.

For daily totals that is survivable. For a workout recorded Tuesday and synced
Thursday it is not: the workout moves to Thursday, and its start/end collapse to the
length of the sync.

Fix first, before any workout code: add a `TIME` characteristic. iPhone writes its
epoch-ms on connect; watch keeps an offset and stamps samples with
`now + offset`. Rebase stays as a fallback for pre-fix leftovers only.

### 3. The watch drops the BLE link when its screen sleeps

The fork's own roadmap says so. A workout screen keeps the app up, and
`ExerciseUpdateCallback` is backed by the foreground service already declared in the
manifest, so a live workout naturally holds the connection. Two additions:

* hold a partial wake lock for the workout's duration (the service already owns one;
  extend rather than add a second)
* keep the workout screen on while a session is active

Sync still needs the iPhone app foregrounded. That is unchanged by this work and is
the accepted cost of the project.

---

## Phases

Each phase is independently shippable and testable on the watch.

### Phase 0 — toolchain and device access

Not optional, and this Mac has none of it yet: no Xcode, no JDK, no Android SDK, no
`adb`, 15 GB free disk.

1. Free disk. Xcode wants ~20 GB installed and refuses to download below ~40 GB
   free; Android SDK + Gradle cache adds ~10 GB.
2. `brew install openjdk@17`, `brew install --cask android-commandlinetools`,
   `brew install xcodegen`. `sdkmanager "platform-tools" "platforms;android-35"
   "build-tools;35.0.0"`.
3. Install Xcode from the App Store, `sudo xcodebuild -license accept`.
4. Watch: long-press the Wear OS logo on the pairing screen → Settings → About →
   tap Software version 5–7× → Developer mode. Note the **Wear OS version and API
   level** here, it gates `ExerciseType` coverage.
5. `adb pair <ip>:<port>` with the pairing code, `adb connect <ip>:5555`.
   Wi-Fi and the Mac must be on the same network.
6. `adb install -r app-debug.apk`, then run the provisioned-flags trick from
   `docs/INSTALL_WEAROS.md` and disable the two setup wizard packages.

Proof: `adb shell pm list packages | grep galaxyhealthbridge` from the Mac.

### Phase 1 — clock

* `Protocol.kt`: `TIME_UUID = e2a00005-…`, write-only, 8-byte LE epoch ms.
* `GattServer.kt`: on write, store offset in `SharedPreferences`.
* New `TimeBase.kt`: `fun now(): Long`; replace the four `System.currentTimeMillis()`
  call sites in `HealthReader.kt` and the one in `GattServer.kt` status payload.
* `BLEClient.swift`: write the time char immediately after the status read, before
  the cursor write.
* Keep `rebaseTimestamps`; tighten its window to 15 minutes so corrected samples
  never get moved.

Proof: watch with a deliberately wrong clock; run a sync; sample timestamps in the
watch's Room DB match the Mac's clock within a second.

### Phase 2 — workout capture on the watch

New files under `wearos/workout/`:

* `WorkoutTypes.kt` — supported `ExerciseType` list filtered by
  `getCapabilitiesAsync()`, and the wire activity token per type.
* `WorkoutController.kt` — `ExerciseClient` wrapper: `start(exerciseType)`,
  `pause()`, `resume()`, `stop()`. Feeds an in-memory session buffer.
* `WorkoutStore` — writes `workout` rows plus per-bucket `hr` / `dist` / `cal`
  samples tagged with the parent workout uid.

`Protocol.kt` gains `act` (string) and `parent` (string?) on `Sample`, and the
workout summary fields in a `meta` blob (`total_kcal`, `total_dist_m`, `total_steps`,
`avg_hr`, `max_hr`).

`WorkoutController`:
* `ExerciseConfig` with `ExerciseType`, `isAutoPauseAndResumeEnabled = true`,
  data types `HEART_RATE_BPM`, `CALORIES`, `DISTANCE`, `STEPS`, `SPEED`
* `ExerciseUpdateCallback`; on each update persist the delta buckets, ignore
  `latestMetrics` inside ambient batches (Health Services batches in ambient — read
  deltas, not latest, or you lose data)
* on `isEnded`, write the summary row and stop

Manifest additions: `READ_HEART_RATE`, `ACCESS_FINE_LOCATION`,
`BODY_SENSORS_BACKGROUND`. Fine location is genuinely required for `DISTANCE`/`SPEED`
on several types.

UI: `MainActivity` gets an activity picker, then a live screen
(duration / HR / distance / kcal, pause, hold-to-finish).

### Phase 3 — HealthKit workout writes

* `CanonicalSampleType.workout`: add `workoutActivityType(from:)` mapping and mark it
  non-quantity so `save()` skips it.
* New `WorkoutWriter.swift`: on a workout frame, open `HKWorkoutBuilder`, buffer the
  child samples by `parent`, `addSamples`, then `addMetadata` +
  `finishWorkout()`. Dedupe by deleting any existing workout whose metadata
  `GHBClientUid` matches before rebuilding.
* `BLESyncCoordinator.toCanonical` learns `act`; grouping by `parent` happens there.
* Request `.workoutType()` share auth, not just read (already in `readTypes`,
  needs adding to `writeTypes`).

Activity mapping — `HKWorkoutActivityType`, unmapped types degrade to `.other` plus
`HKMetadataKeyWorkoutBrandName` = Samsung's label:

| Wire token | HealthKit |
| --- | --- |
| `walking` | `.walking` |
| `running` | `.running` |
| `hiking` | `.hiking` |
| `cycling` | `.cycling` |
| `indoor_cycling` | `.cycling` |
| `swimming_pool` / `swimming_open_water` | `.swimming` |
| `rowing_machine` | `.rowing` |
| `elliptical` | `.elliptical` |
| `stair_climbing` / `stair_climbing_machine` | `.stairClimbing` |
| `strength_training` | `.traditionalStrengthTraining` |
| `hiit` | `.highIntensityIntervalTraining` |
| `yoga` | `.yoga` |
| `dance` | `.dance` |
| `other_workout` | `.other` |

Every one of these needs a negative test: total kcal on the workout equals the sum
of the child `cal` samples, and a re-sync does not duplicate the workout.

### Phase 4 — signing

Free Apple ID profile expires every 7 days. Order of attempts:

1. Plain Xcode install with a personal team. Seven-day reinstall, zero extra
   infrastructure.
2. If the weekly reinstall is the thing that kills it: SideStore from the Mac's
   pairing file, or AltStore with the Mac on the same Wi-Fi.

A $99/yr account buys a year per signature and removes this phase entirely. Note the
comparison honestly: Merge Pro is $59.99/yr, so the paid account only pays off if
what you are buying is control and workouts rather than money.

---

## Test strategy

* Kotlin unit tests for the mapping and the frame packer (existing pattern:
  `wearos/app/src/test/.../RequestPayloadTest.kt`). No device needed.
* Swift unit tests for `rebaseTimestamps` at the new window and for
  `workoutActivityType(from:)` exhaustiveness — a missing case must be a compile
  error, so use an exhaustive `switch` over an enum, not a dictionary.
* One end-to-end run per phase on the watch: wrong-clock sync, 3-minute walk,
  10-minute run, a re-sync, and a Health app check that the workout shows once with
  the right type and duration.

## Risks

* **`ExerciseType` availability on Watch 4.** Samsung's Health Services provider may
  expose a thin subset. If a wanted type is unavailable, the picker simply will not
  offer it — no code change, but expect fewer types than the table above.
* **Plausible-looking wrong data is the worst outcome.** A workout with a wrong type
  or a doubled calorie total is worse than no workout. The mapping test and the
  kcal-sum test are the guard.
* **7-day signing** will expire mid-project. Rebuild rather than assume a stale
  install.
* **Wear OS version drift on Watch 4.** Confirm the API level in Phase 0 before
  writing `ExerciseClient` code; a Wear OS 6 / API 36 device changes availability
  semantics and the sleep-stage path.
