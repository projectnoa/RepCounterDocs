# 06 · HealthKit and live metrics

[← Phone ↔ watch sync](05-watch-sync.md) · [README](../README.md) · Next: [Voice rep counting →](07-voice-counting.md)

HealthKit is used for two things:

1. **Today's activity** on the Home screens: calories, steps, distance and exercise time
   (`HealthManager`).
2. **Live workout metrics** during a run: heart rate, active energy and elapsed time.
   These come from an `HKWorkoutSession` that runs **on the watch** and is **mirrored** to
   the phone (`WorkoutManager`).

## `HealthManager` (both apps)

`Shared/HealthManager.swift`, `@MainActor`, owns the app's `HKHealthStore`.

- `typesToRead`: exercise time, walking and running distance, step count, activity summary.
- `refreshData()` runs four `HKStatisticsQuery`s from the start of today and publishes
  `todayCaloriesCount`, `todayStepCount`, `todayDistance` and `todayExerciseTime`. The
  query callbacks arrive on a HealthKit queue and hop to the main queue before publishing.
- The phone asks for access with SwiftUI's `.healthDataAccessRequest` modifier: on Home
  after the first-launch login, and on a workout's **Start** (with `WorkoutManager`'s
  types). The watch calls `requestAuthorization()`.

## `WorkoutManager` (both apps)

`Shared/WorkoutManager.swift` has the shared parts. `WorkoutManager+watchOS.swift` and
`WorkoutManager+iOS.swift` have the platform parts.

Published state that the UI observes: `sessionState`, `heartRate`, `activeEnergy`, `water`,
`elapsedTimeInterval`, and on the watch the finished `workout` (`HKWorkout`).
`peakHeartRate` and `averageHeartRate` are plain properties, read when a run is recorded.

`typesToShare`: workouts and dietary water. `typesToRead`: heart rate, active energy,
dietary water, workouts.

### Starting a session

**From the phone** (the usual path):

1. `WorkoutDetailView` → **Start** → HealthKit authorisation.
2. `WorkoutManager.startWatchWorkout(workoutType: .traditionalStrengthTraining, with:)`,
   a `static` method, calls `HKHealthStore.startWatchApp(toHandle:)`. The watch app launches, or wakes, and
   the system calls the watch `AppDelegate.handle(_ workoutConfiguration:)`.
3. The watch shows the phone's cards (`LiveRunView`), or `MetricsView` until they arrive
   (`showExerciseMetrics = true`), and calls
   `startWorkout(workoutConfiguration:with:)`. That creates the `HKWorkoutSession` and
   its `HKLiveWorkoutBuilder`, calls **`startMirroringToCompanionDevice()`**, and starts
   the activity and collection.
4. On the phone, `retrieveRemoteSession(with:)` installs
   `healthStore.workoutSessionMirroringStartHandler`. **`RepCounterApp.init` calls it at
   launch.** The system can launch the app in the background just to deliver a mirrored
   session. A handler installed by a screen (it used to be Home) is missing whenever the
   app restores to another tab, and the run then gets no live metrics. When the mirrored
   session arrives, the phone adopts it as its `session` and becomes its delegate.

**From the watch:** `WorkoutControlView` starts the same session directly when a watch run
begins. It is still mirrored, so a phone that is running sees it: Home shows *Active
Workout → Continue*, which opens `MirroringWorkoutView`.

The phone never creates a session of its own. A phone-only run has no live metrics.

### Data flowing over the mirrored session

`sendToRemoteWorkoutSession(data:)` carries raw `Data`. The formats are defined in
`MirroredPayload`:

| Direction | Payload | Encoding |
| --- | --- | --- |
| watch → phone | `.statistics([HKStatistics])` | `NSKeyedArchiver` (secure coding) |
| watch → phone | `.elapsedTime(WorkoutElapsedTime)` | JSON `{timeInterval, date}` |
| phone → watch | `.water(HKQuantity)` | `NSKeyedArchiver` |

The data has no type tag, because the format predates the enum and older builds must still
be read. `MirroredPayload(data:)` tries each format in turn: elapsed time is JSON, water is
an archived `HKQuantity`, statistics an archived array. Anything else is `nil` and
ignored.

- **Statistics:** on every `workoutBuilder(_:didCollectDataOf:)` the watch updates its own
  published values (`updateForStatistics`) and sends the batch to the phone. The phone
  applies the same `updateForStatistics`.
- **Elapsed time:** on every session state change the watch sends the builder's elapsed
  time and the date it was measured. While the session is running, the phone adds the time
  since then, so its clock doesn't lag.
- **Water:** the phone's `MirroringWorkoutView` has a water button. It sends 1 fl oz. The
  watch adds a `dietaryWater` sample to the builder.

### Session state changes and `AsyncStream`

HealthKit calls `workoutSession(_:didChangeTo:from:date:)` on a background queue, and the
handling is `async`: ending the builder and finishing the workout. Swift doesn't run
separate `Task`s in the order they were created, so a `Task` per callback could process
*stopped* before *running*. Instead:

```swift
let asynStreamTuple = AsyncStream.makeStream(of: SessionSateChange.self,
                                             bufferingPolicy: .bufferingNewest(1))
// init:
Task { for await value in asynStreamTuple.stream { await consumeSessionStateChange(value) } }
// delegate (nonisolated):
asynStreamTuple.continuation.yield(SessionSateChange(newState: toState, date: date))
```

A single consumer processes changes one at a time, in order. On the watch, when the state
becomes `.stopped` it calls `endCollection`, `finishWorkout` and `session.end()`, then
publishes the `HKWorkout` for `SummaryView`.

### Ending

- **Phone End button**: the phone calls
  `session?.stopActivity(with:)` on the mirrored session. That stops the watch's session,
  and watch `MetricsView` closes when it sees `.ended`.
- **Watch End on the live metrics screen** (`MetricsView.exit`): stops the activity and
  sends `SyncMessage.endWorkout`. The phone ends its run as if **End** were pressed
  ([05](05-watch-sync.md)).
- **Watch run End** (`ControlView`): records the run as ended early, stops the activity,
  and dismisses.
- **Watch run reaching its summary card:** the run is recorded and sent, then the activity
  is stopped. The builder finishes in the background while the user leaves with **Done**.
  `consumeSessionStateChange` keeps its own references to the session and builder for the
  whole finish, so leaving (which calls `resetWorkout()`) can't stop the session from being
  ended. It publishes the `HKWorkout` only if that builder is still current, so a finished
  run's totals never show up on a later run's summary. A watch run also calls
  `resetWorkout()` before starting, for the same reason.
- `resetWorkout()` clears the session, builder and all published values before a new
  session.

## Delegates and isolation

`WorkoutManager` is `@MainActor`, but HealthKit calls its `HKWorkoutSessionDelegate` and
`HKLiveWorkoutBuilderDelegate` methods on its own serial queues. So those methods are
declared `nonisolated`. They either yield to the stream, which is safe because the stream
tuple is a `let` constant, or they wrap their work in `Task { @MainActor in … }`. See
[08 · Concurrency](08-concurrency.md#delegates-that-call-back-on-other-queues).
