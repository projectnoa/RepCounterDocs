# 05 · Phone ↔ watch sync

[← The workout run](04-workout-run.md) · [README](../README.md) · Next: [HealthKit →](06-healthkit.md)

The apps talk over two channels:

| Channel | Carries | Code |
| --- | --- | --- |
| **WatchConnectivity** (`WCSession`) | App-level messages: the workout catalogue, settings, recorded runs, "end the workout" | `Shared/WatchSync.swift`, both `ConnectivityManager`s |
| **HealthKit session mirroring** | Live workout data while a session runs: statistics, elapsed time, water | `MirroredPayload` in `WatchSync.swift`, `WorkoutManager` ([06](06-healthkit.md)) |

This page covers WatchConnectivity.

## WatchConnectivity refresher

A `WCSession` offers several ways to send, with very different guarantees:

| API | Delivered when | Kept if the other app isn't running? | Notes |
| --- | --- | --- | --- |
| `sendMessage` | Now, if the counterpart is **reachable** | No | Fast, and can wake the iPhone app in the background |
| `updateApplicationContext` | Next time the counterpart runs | Only the **latest** dictionary | Each call replaces the previous context entirely. An identical dictionary is not resent. |
| `transferUserInfo` | Eventually, in order | Yes, all of them, queued | For data that must not be lost |

Delegate callbacks arrive on a **background queue**. Payloads must be property-list types
(`String`, `Data`, numbers, dates, arrays and dictionaries of them).

## The contract: `SyncMessage`

`Shared/WatchSync.swift` is compiled into both apps, so there is one definition of every
message:

```swift
enum SyncMessage: Equatable, Sendable {
    case endWorkout                      // watch → phone
    case workouts([WorkoutSnapshot])     // phone → watch
    case settings(RunSettings)           // phone → watch
    case workoutRun(WorkoutRunRecord)    // watch → phone
    case liveRun(LiveRunState)           // both ways, during a phone-started run
}
```

Each message has a **delivery policy**, and the channel sends it accordingly:

| Message | `delivery` | How it is sent | Why |
| --- | --- | --- | --- |
| `endWorkout` | `.liveOrWhenLaunched` | `sendMessage` if reachable, otherwise the watch's application context | The phone should end its run now, or as soon as it runs |
| `workouts` | `.latestState` | Merged into the phone's application context | Only the latest catalogue matters |
| `settings` | `.latestState` | Merged into the phone's application context | Only the latest settings matter |
| `workoutRun` | `.guaranteed` | `transferUserInfo` | A run must never be lost, and each one counts |
| `liveRun` | `.liveState` | Merged into the application context, and `sendMessage` too when reachable | Should arrive at once, but a watch that comes up after the phone started still needs it |

`payload(id:)` builds the property-list dictionary: keys `endWorkout`, `workouts`,
`settings`, `workoutRun`, `liveRun`, with values encoded as JSON `Data` using `.sortedKeys`.
Non-state messages also carry a random `id`, because `updateApplicationContext` drops a
context identical to the last one. Without the id, a second `endWorkout` sent while the
phone is away would never arrive. State messages carry **no** id, so an unchanged catalogue
is an identical context and isn't delivered again. `.sortedKeys` keeps the bytes stable for
the same reason.

`SyncMessage.messages(in:)` decodes a received dictionary into every message it holds. It
ignores unknown keys and malformed values, so an older or newer counterpart can't crash
it. Payloads from older app versions still decode, for example `["endWorkout": true]`
without an id.

## Live runs: the phone's cards on the watch

When the phone starts a run, HealthKit launches the watch app to run the workout session
([06](06-healthkit.md)). The watch then shows **the same cards** as the phone, on the same
one, and either device can page them and log reps and weight:

- **`LiveRunState`** (`Shared/WatchSync.swift`): the run's id, workout id, the phone's
  layout (`showsExercisePreviews`) and *Time workout* switch, the active card (its index in
  `RunPlan`), each card's `CardInput` (reps, weight in kg), `isEnded`, `updatedAt` and
  `origin`. A timed exercise's time isn't synced: each device measures its own card.
- **`LiveRunSync`**: one side's copy. Local changes update the state at once and are sent
  250 ms later, so turning a wheel sends where it stops rather than every tick. `receive`
  takes the counterpart's state when it's the same run and newer (last writer wins; the
  whole state is small), and reports what changed (`LiveRunChange`). `end()` sends at
  once.
- **Phone** (as little as possible changes there): `WorkoutControlView` owns a
  `LiveRunLink` and passes it down as `\.liveRunLink`. The carousel reports moves and
  follows `requestedCard`; `ExerciseCardView` reports what's logged and shows
  `watchInputs[card]`. `ConnectivityManager.watchLiveRun` holds the latest state from the
  watch. Without a watch app nothing is sent (`SyncChannel` checks), so the run is exactly
  as before.
- **Watch**: `ConnectivityManager.liveRun`, `showLiveCard(_:)` and `logLive(_:onCard:)`.
  `RootView` shows `LiveRunView` while `showExerciseMetrics` is on and the state is current,
  and `MetricsView` otherwise (no state yet, an older phone app, or a workout missing from
  the watch's catalogue). `LiveRunLayout` builds the cards from `RunPlan` with the phone's
  options, intro and rests included, so indices match. The watch doesn't record the run;
  the phone does. Its summary card adds up the synced inputs (`RunSummarySnapshot`).
- The run ends with the phone's session (`showExerciseMetrics = false` clears the state) or
  the watch's End, which also sends `endWorkout`.

## The layers

```
ConnectivityManager (WCSessionDelegate, @MainActor)    what the app does with messages
        │ send(_:) / deliver(_:)
SyncChannel (@MainActor)                               delivery rules, state merging
        │
SyncTransport (protocol)  ←  WCSessionTransport         WCSession, or LoopbackTransport in tests
```

- **`SyncTransport`** is the few `WCSession` operations the apps use: `isCounterpartInstalled`,
  `isReachable`, `sendMessage`, `updateApplicationContext`, `transferUserInfo`.
  `WCSessionTransport` is the real one. Tests use `LoopbackTransport`, which pushes every
  payload through `PropertyListSerialization`, so a payload WatchConnectivity couldn't
  carry fails the test ([11](11-testing.md)).
- **`SyncChannel`** applies the delivery rules and does nothing when the counterpart app
  isn't installed. The phone's application context has to hold **both** the catalogue
  and the settings, and WatchConnectivity keeps only one context. So the channel keeps
  every piece of state sent so far and **merges** each new state message into it before
  calling `updateApplicationContext`.
- **`ConnectivityManager`** activates the session and is its delegate. Every delegate
  method (`didReceiveMessage`, `didReceiveApplicationContext`, `didReceiveUserInfo`, and
  the reply-handler variant) funnels into one `nonisolated deliver(_:)`. That decodes on
  the background queue, then hops to the main actor to handle the messages in order. The
  reply-handler variant always replies, so a sender that asked for a reply never times out.

## Phone side (`RepCounter/Core/ConnectivityManager.swift`)

**Publishing the catalogue.** `publishWorkouts(force:)` builds
`WorkoutSnapshot.all(in: context)` and sends it unless it equals the last one sent. It
runs:

- when the session activates, and when the watch app is newly installed (both with
  `force: true`);
- after any save of the view context that inserts, updates or deletes a `Workout`,
  `Circuit`, `Round`, `Exercise`, `Target` or `RoundMetric` (`changesCatalogue(_:)`).
  `RoundMetric` is in the list because records travel with the catalogue, so a run that
  sets a new record republishes. A run that changes no record produces an identical
  snapshot and sends nothing.

**Publishing settings.** `publishSettings(force:)` sends `RunSettings`, which today is
just `showsExercisePreviews`. It runs on activation, when the watch app is installed, and
whenever the toggle in Settings changes.

**Receiving.**

- `endWorkout` increments `endWorkoutRequests`. `WorkoutControlView` observes the counter
  and ends the phone's run as if **End** were pressed. It is a counter, not a flag, so
  every request is observed.
- `workoutRun` goes to `WorkoutRunImporter.save`, then the History is backed up to
  Firestore.

## Watch side (`RepCounter Watch App/Core/ConnectivityManager.swift`)

On activation it handles `session.receivedApplicationContext` straight away, because the
latest catalogue and settings may have arrived while the app wasn't running.

**Receiving.**

- `workouts`: first merges the records the snapshots carry into the stored records
  (`PersonalRecords.store(merging:in:)`). Then it applies the catalogue with
  `WorkoutCatalogImporter`, unless a run is in progress. In that case it keeps the latest
  one in `pendingWorkouts` and applies it when `isRunInProgress` goes false.
- `settings`: stores `RunSettings` in `UserDefaults`: `showExercisePreview`, and the
  phone's weight unit (`weightUnit`, when it sends one). The next
  run's layout reads it. A run already on screen is unchanged.

**Sending.** `send(.workoutRun(record))` when a run finishes, and
`sendEndWorkoutMessage()` from the live metrics screen's End button.

## The transfer models

### `WorkoutSnapshot` (phone → watch)

A self-contained copy of one workout: name, info, icon, difficulty, favourite, targets,
**owner and `assignedBy`** (so the watch's list groups a coach's, the user's and the
catalogue's workouts as the phone does), and circuits with their rounds. Each round
carries its precise weight, whether it's timed, and its note; its exercise carries its id,
name, image and target, plus the exercise's **record** (most reps), **`lastTime`** (the
best set of the last run that included it) and **`bestSet`**. The ids are the phone's ids,
which is what lets a run recorded on the watch be matched back to the phone's circuits and
rounds. Every field added after the first version is optional, so either app still reads
an older counterpart.

- `WorkoutSnapshot.all(in:)` snapshots every listed workout, catalogue and user alike, and
  attaches `PersonalRecords.historicalBest` and `PersonalRecords.benchmarks`. Archived
  workouts are left out, so archiving one on the phone deletes it from the watch.
- `records` collects the per-exercise records, and `benchmarks` last time and the best set.
  The watch keeps both in `UserDefaults` (`personalRecords`, `personalRecords.benchmarks`)
  and updates them after its own runs. Its exercise cards read them as the phone's do: the
  ring fills to last time's reps, and the status reads "Last time: 60 kg × 8".
- `ignoringRecords` is the same snapshot with records and benchmarks cleared, for
  comparing structure only.

### `WorkoutCatalogImporter` (on the watch)

`apply(_:to:)` makes the watch's store match the snapshots:

- Workouts the phone no longer has are deleted, with their circuits and rounds.
- New workouts are created with the phone's id.
- Existing workouts are **skipped if unchanged**. The comparison is against the snapshot
  `ignoringRecords`, because records are stored separately and a new record alone must not
  rebuild a workout.
- Changed workouts get their fields updated and their circuits and rounds **rebuilt** from
  the snapshot. Rebuilding is safe on the watch because it keeps no History that could point
  at the old objects.
- Exercises and targets are found by id, or created.

### `WorkoutRunRecord` (watch → phone)

A run as the watch recorded it: `id`, `workoutID`, `completedAt`, `duration`, heart
rate, calories, and `sets`. Each set holds `circuitID`, order, calories, duration and
`rounds`. Each round holds `roundID`, order, reps, weight, duration and rest.

### `WorkoutRunImporter` (on the phone)

`save(_:in:)` turns a record into the same History graph a phone run produces:

- Saved **once**: a record whose id is already a History is ignored. That covers
  `transferUserInfo` delivering a record twice.
- Dropped if its workout no longer exists on the phone.
- Sets and rounds whose circuit or round can't be found are skipped. The rest are kept.
- `run` is numbered with `History.nextRun(of:)`, and `createdAt` is the run's
  `completedAt`.
- On a save error the context is rolled back.

## Adding a new message

1. Add a case to `SyncMessage` and give it a `delivery`.
2. Add a `Key`, and encode it in `payload(id:)` and decode it in `messages(in:)`. Keep
   decoding tolerant.
3. Handle it in **both** `ConnectivityManager.handle(_:)` switches. The compiler makes you
   decide what the other side does with it, even if that is `break`.
4. If it's state the phone publishes, publish it on activation and when the watch app is
   installed, as `publishWorkouts` and `publishSettings` do.
5. Add contract tests (`SyncContractTests`) and end-to-end tests on both sides
   (`PhoneWatchSyncTests`, `WatchPhoneSyncTests`).
