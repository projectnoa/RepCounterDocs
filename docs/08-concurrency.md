# 08 · Concurrency in this codebase

[← Voice rep counting](07-voice-counting.md) · [README](../README.md) · Next: [Firebase →](09-firebase.md)

Every target, apps and tests alike, compiles in the **Swift 6 language mode**. There, data
races are compile-time errors rather than warnings. This page is a refresher on the pieces
of Swift concurrency you'll meet here, each shown with the project's own code, plus the
rules this project follows.

## The model in brief

- **`async`/`await`:** an `async` function can suspend at each `await` without blocking
  its thread. Code after an `await` may resume later, possibly on a different thread,
  unless it is isolated to an actor.
- **`Task { … }`:** starts asynchronous work from synchronous code. A `Task` created
  inside actor-isolated code **inherits that isolation**. A `Task` inside a `@MainActor`
  method runs on the main actor. `Task.detached` doesn't inherit, and this project doesn't
  use it.
- **Actors:** reference types that protect their mutable state by running one piece of
  their code at a time. From outside you `await` their methods.
- **Global actors:** `@MainActor` is the actor for the main thread. Marking a type
  `@MainActor` isolates all its members to it.
- **`Sendable`:** a type that is safe to share across isolation domains, meaning value
  types of `Sendable` parts, immutable classes, actors, and `@Sendable` closures. Swift 6
  checks every value that crosses from one isolation domain to another.
- **`nonisolated`:** opts a member of an isolated type out of that isolation. It can then
  be called from any thread, but it can't touch isolated state synchronously.

## Rule 1: UI, state and Core Data live on the main actor

Almost everything that holds state is `@MainActor`:

- Managers: `AuthManager`, `RemoteConfigProvider`, `HealthManager`, `WorkoutManager`, both
  `ConnectivityManager`s.
- Behaviour-rich view models: `WorkoutCardCarouselView.ViewModel`,
  `WorkoutEditView.ViewModel`, the watch carousel's `ViewModel`.
- Services and rules that touch Core Data: `PersistenceController`, `CatalogueSync`,
  `WorkoutRunImporter`, `RunPlan.cards`, `PersonalRecords`, `WatchRunRecorder`,
  `SyncChannel`, `WorkoutCloudStore`.

SwiftUI views are main-actor isolated already, so they call all of these synchronously.
The rare exceptions are pure value types that cross threads, such as
`SyncMessage`, `WorkoutSnapshot`, `WorkoutRunRecord` and `RunSettings`. They are
`Sendable` structs and enums.

### Core Data and actors

`NSManagedObjectContext` has its own threading rule: use a context only on its queue. The
view context's queue is the main queue, which is the main actor. So:

- All Core Data work uses the **view context**, or a **main-queue** child of it, from
  `@MainActor` code. `WorkoutEditView` once used a private-queue background context from
  the main thread. That violates the rule even though it compiles, and it was replaced by a
  main-queue scratch context.
- **Don't pass managed objects across actors.** `NSManagedObject` is not `Sendable`. To
  hand data to another domain, copy it into a value type (`WorkoutSnapshot`,
  `Workout.Model`) or pass the `NSManagedObjectID`.
- If you ever need background Core Data work, create a `newBackgroundContext()` and do
  everything inside `context.perform { … }`. Move results back as `NSManagedObjectID`s.

`DataStore` is declared as an `actor`, but nearly all its methods are `@MainActor` or
`nonisolated`, because they take the view context. It is effectively a namespace, not a
source of isolation. Don't add actor-isolated state to it without thinking about who calls
it.

## Rule 2: callbacks from other queues go through `nonisolated`

### Delegates that call back on other queues

WatchConnectivity, HealthKit and Speech call their delegates and handlers on their own
background queues. A `@MainActor` class that conforms to such a delegate protocol declares
those methods `nonisolated`, then moves to the main actor explicitly:

```swift
// ConnectivityManager (both apps)
nonisolated func session(_ session: WCSession, didReceiveMessage message: [String: Any]) {
    deliver(message)
}

nonisolated private func deliver(_ payload: [String: Any]) {
    let messages = SyncMessage.messages(in: payload)   // decode here: [String: Any] isn't Sendable
    guard messages.isEmpty == false else { return }

    Task { @MainActor in                               // [SyncMessage] is Sendable
        self.channel.receive(messages)
    }
}
```

The payload dictionary is decoded into `Sendable` messages **before** the hop, because
`[String: Any]` can't cross into the main actor in Swift 6.

The same shape appears in `WorkoutManager`: `workoutBuilder(_:didCollectDataOf:)` and
`workoutSession(_:didReceiveDataFromRemoteWorkoutSession:)` both do
`Task { @MainActor in … }`. It also appears in `SpeechRecognizer`'s `nonisolated`
recognition handler.

### `MainActor.assumeIsolated`

When a callback is guaranteed to run on the main thread but the compiler can't prove it,
`MainActor.assumeIsolated { … }` asserts it at runtime and gives you main-actor access
without a hop. The phone `ConnectivityManager` uses it in its `NSManagedObjectContextDidSave`
observer, registered with `queue: .main`. The notification is examined **before** that
call, in a `nonisolated static func changesCatalogue(_:)`, because `Notification` isn't
`Sendable` and may not be passed into the isolated closure.

## Rule 3: ordering is not free: use `AsyncStream` for serial work

Separate `Task`s are not guaranteed to run in the order they were created. When events
must be handled one at a time, in order, and the handling is `async`, feed them into an
`AsyncStream` and consume it from a single task. `WorkoutManager` does this for session
state changes ([06](06-healthkit.md#session-state-changes-and-asyncstream)):

```swift
let asynStreamTuple = AsyncStream.makeStream(of: SessionSateChange.self,
                                             bufferingPolicy: .bufferingNewest(1))
```

`continuation.yield` is synchronous and safe from any thread. The tuple is a `let`, so it
can be used from `nonisolated` code.

## Rule 4: time comes from dates, not from ticks

Timers stop firing while the app is in the background, so counting ticks drifts. Durations
here are always differences between `Date`s:

- `TimerView` banks elapsed time from dates inside a `.task(id:)` loop that sleeps with
  `Task.sleep(for:)`. See [04](04-workout-run.md#3-timers-and-time-workout).
- Exercise cards measure time on the card from `activatedAt`.
- Set durations use `SetSplits.durations(starts:end:)`.

## SwiftUI and concurrency

- **`.task { … }` / `.task(id:) { … }`** run async work tied to a view's lifetime. The task
  is cancelled when the view disappears or `id` changes. Prefer them over `onAppear` plus
  an unstructured `Task`.
- **`@State`, `@StateObject`, `@Observable`:** view models that SwiftUI owns live in
  `@StateObject` (for `ObservableObject`) or `@State`. `TabSettings` in the phone `RootView`
  uses the `@Observable` macro. The rest of the app uses `ObservableObject` with
  `@Published`.
- **`@Entry`** declares custom environment values in one line, as
  `workoutManagerReader` does.

## Combine: where it remains

The project prefers async/await, and new code should not add Combine. What remains:

- `@Published` on `ObservableObject`s. That is how SwiftUI observes them, and it counts as
  Combine only technically.
- `onReceive(provider.$freshDataAvailable.dropFirst())` in `SplashView`, and
  `onReceive(connectivityManager.$refreshApp.dropFirst())` in `RootView`.
- `Timer.publish(every: 1, …)` in `VitalsMetricsView` (phone) and `MetricsView` (watch),
  for the live clock. These could move to `TimelineView(.periodic(from:by:))` or a
  `.task` loop like `TimerView`'s.

## Third-party code and `@preconcurrency`

`AuthManager` imports FirebaseAuth with `@preconcurrency`. Firebase's types predate
`Sendable` annotations, so this silences warnings about them crossing isolation. Use
`@preconcurrency` only for modules you don't own.

## Checklist for new code

- A new stateful type that views use is `@MainActor`.
- A delegate method from an Apple framework on a `@MainActor` type is `nonisolated`, and
  it hops with `Task { @MainActor in }` after converting inputs to `Sendable` values.
- Nothing `NSManagedObject` crosses an actor boundary.
- Ordered async events go through one `AsyncStream` consumer.
- Durations come from `Date` arithmetic.
- No `nonisolated(unsafe)` and no `@unchecked Sendable`. The codebase has none; keep it
  that way unless there is a documented reason.
