# 11 · Testing

[← Settings and storage](10-settings-and-storage.md) · [README](../README.md) · Next: [Conventions and gotchas →](12-conventions-and-gotchas.md)

## The four test targets

| Target | Framework | Host | Run with |
| --- | --- | --- | --- |
| `RepCounterTests` | Swift Testing (`import Testing`, `@Suite`, `@Test`, `#expect`) | The phone app (`@testable import RepCounter`) | `RepCounter` scheme, iPhone 17 simulator |
| `RepCounter Watch AppTests` | Swift Testing | The watch app (`@testable import RepCounter_Watch_App`) | `RepCounter Watch App` scheme, Apple Watch Ultra 4 (49mm) simulator |
| `RepCounterUITests` | XCUITest | — | `RepCounter` scheme |
| `RepCounter Watch AppUITests` | XCUITest | — | `RepCounter Watch App` scheme |

All four compile in Swift 6 mode. Unit tests are `@MainActor` suites where they touch Core
Data or the managers.

> Always check the destination first. The UI tests drive whatever device is selected,
> including a real iPhone.

## Swift Testing refresher

```swift
@MainActor
@Suite("Workout run recording")
struct WorkoutRunRecordingTests {
    let stack = CoreDataTestStack()          // a fresh value per test, so a fresh store

    @Test("Only reaching the summary card finishes the workout", arguments: [1, 2, 3])
    func onlyTheSummaryFinishesTheWorkout(finalCircuitSets: Int) { … }

    @Test func somethingThrowing() throws {
        let history = try #require(History.fetchAll(with: stack.context).first)
        #expect(history.run == 1, "The workout's first run")
    }
}
```

- A suite is a type. Swift Testing creates **a new instance for every test**, so stored
  properties are per-test setup, and there is no `setUp`/`tearDown`.
- `#expect(condition, "message")` records a failure and continues. `try #require(…)` unwraps or
  fails, then stops the test.
- `arguments:` runs a test once per value. `zip(a, b)` pairs arguments.
- Tests run **in parallel** by default, even within a suite. Anything global, such as
  `UserDefaults.standard`, `PersistenceController.shared` or `WCSession.default`, must
  not be mutated from tests. Inject a private instance instead, as below.
- `confirmation(expectedCount:)` checks that a callback fires, or doesn't.

## Test infrastructure

### `CoreDataTestStack` (`RepCounterTests/Support/`)

An isolated store per test: an `NSPersistentContainer` on a SQLite store at `/dev/null`.
It lives in memory, but unlike `NSInMemoryStoreType` it supports batch deletes. It has
factories for every entity (`makeWorkout`, `makeCircuit`, `makeRound`,
`makeHistory`, `makeCircuitMetric`, `makeRoundMetric`, …).

It **reuses the host app's loaded `NSManagedObjectModel`**. Loading the model a second time
would leave two entity descriptions claiming each `NSManagedObject` subclass, and every
`Workout(context:)` would crash with "Failed to find a unique match for an
NSEntityDescription". The watch tests have `WatchCoreDataTestStack` with the same design.

### `LoopbackTransport` (`RepCounter Watch AppTests/Support/SyncTestSupport.swift`)

Both unit test targets compile this file. It is a `SyncTransport` that stands in for
`WCSession`:

- Records every send (`sends: [.message, .applicationContext, .userInfo]`).
- Round-trips every payload through `PropertyListSerialization`, so a payload
  WatchConnectivity couldn't carry fails the test.
- Follows WatchConnectivity's queueing rules: the application context keeps only the latest
  value, drops one identical to the last, and delivers on `counterpartDidLaunch()`. User
  info is queued in order.
- `deliver(to: delegate)` calls a real `ConnectivityManager`'s `WCSessionDelegate` methods,
  and `deliver(to: channel)` feeds a bare `SyncChannel`.

With it, a sync test wires the **real** `ConnectivityManager` of one app to a stand-in for
the other, for example `PhoneWatchSyncTests`. Helpers `waitUntil { … }` and `settle()` wait
for the main-actor hop that follows every delivery.

### Injection points made for tests

| Type | Parameter | Tests pass |
| --- | --- | --- |
| `ConnectivityManager` (both) | `transport:`, `session:`, `context:`, `defaults:` | A loopback, `nil` (no `WCSession`), a test stack, a suite-named `UserDefaults` |
| `WorkoutEditView.ViewModel` | `viewContext:`, `cloud:` | A test stack, `WorkoutCloudSpy` |
| `WorkoutCardView.ViewModel.toggleFavorite` | `cloud:` | `WorkoutCloudSpy` |
| `PersonalRecords.stored/store`, `RunSettings.stored/store` | `defaults:` | `UserDefaults(suiteName: "<Suite>.<UUID>")` |
| `WatchRunRecorder` methods, `recordCircuit`, `finishRun` | `at: Date` | Fixed dates, so durations are exact |

### The app knows when it is hosting tests

`UITestConfiguration.isHostingUnitTests` is true when `XCTestConfigurationFilePath` is
set. The phone app then:

- shows `Color.clear` instead of `RootView`, so the splash screen can't seed or reset the
  shared store while tests run;
- skips the Remote Config fetch;
- creates `ConnectivityManager(session: nil)` and doesn't activate `WCSession`;
- skips all Firestore writes (`skipsCloudSync`).

## What the unit tests cover

**Phone** (`RepCounterTests`): entity factories, getters and defaults (`WorkoutTests`,
`ExerciseTests`, `CircuitAndRoundTests`, `HistoryTests`, `LookupEntityTests`,
`UUIDLookupTests`), Codable round trips (`CodableModelTests`), sorting and formatting,
the exercise filter's predicates, `DataStore` resets, `CatalogueSync`, the run recording
(`WorkoutRunRecordingTests`: carousel layout, the metric graph, per-set calories and
durations, revisits, ending early, run numbers, personal records), editing workouts in
place (`Workout editing`), the Firestore seam (`Workout cloud sync`), phone-side sync
(`PhoneWatchSyncTests`), and the transcription number table. The workout generator isn't
covered yet; its model-free steps can be ([13 · Testing it](13-workout-generation.md#testing-it)).

**Watch** (`RepCounter Watch AppTests`): the message contract (`SyncContractTests`), the
carousel layout (`WatchCarouselTests`), watch-side sync (`WatchPhoneSyncTests`), catalogue
import, run recording and records (`WatchRunRecordingTests`), and `WorkoutManager` state
and water handling (`WatchWorkoutSessionTests`).

## UI tests

### Launch arguments

Every UI test launches with `--uitesting` and optional switches, read by
`Shared/UITestConfiguration.swift`:

| Argument | Effect |
| --- | --- |
| `--uitesting` | Master switch. In-memory store seeded from the frozen test catalogue ([Fixtures](#fixtures)), no Remote Config, no Firebase Auth (signed in as `uitest-user`), no login cover, no HealthKit sheet, no Firestore writes. On the watch, no `WCSession` either, so the paired phone simulator can't overwrite fixtures. |
| `--uitesting-show-login` | Keep the first-launch login cover |
| `--uitesting-show-intro` | Show the first-launch intro slideshow, whatever an earlier run left in `UserDefaults`. Alone, finishing it goes home; with `--uitesting-show-login` the login screen follows. TipKit tips are always hidden under `--uitesting`. |
| `--uitesting-show-health-prompt` | Let the HealthKit sheet appear |
| `--uitesting-signed-out` | Start signed out |
| `--uitesting-empty-store` | No catalogue |
| `--uitesting-minimal-workout` | Add a one-round workout (`UITestSeed.insertMinimalWorkout`) |
| `--uitesting-long-workout` | Add a three-circuit, multi-set workout (`UITestSeed.insertLongWorkout`) |
| `--uitesting-watch-workout` | Watch: seed the fixture workout and open its detail screen |
| `--uitesting-watch-catalog` | Watch: seed the fixture, launch on Home |
| `--uitesting-watch-metrics` | Watch: launch on the live metrics screen, as if the phone had started a workout |
| `--uitesting-live-metrics` | Phone and watch: `UITestLiveMetrics` publishes changing heart rate and energy on the `WorkoutManager` once a second, about as often as a paired watch's session does. On the phone the vitals overlay shows them, as if a watch were paired. Use it to check that live updates don't disturb what's on screen. |
| `--uitesting-cloud` | Phone: turns on backup, people, coaching and profiles against the in-memory fakes in `UITestBackend` (off otherwise, as before). |
| `--uitesting-coaching` | Phone, implies `--uitesting-cloud`: seeds `UITestPeople`, a trainee the user coaches (with three runs of progress), a coach who assigned them a workout with a custom exercise, someone asking to follow them, and someone to follow with code `KAPTEN23`. |
| `--uitesting-custom-exercise-workout` | Phone: also seeds a user workout of one round of a custom exercise (`UITestSeed.insertCustomExerciseWorkout`). |

`UITestCase` (phone) and `WatchUITestCase` (watch) wrap launching and waiting, and add
helpers for scrolling, finding the current carousel card, and paging.
`WorkoutRunScript.swift` derives the card-by-card script a workout's run should follow,
mirroring `RunPlan`. It walks the carousel entering reps and weights, which is what the
long-run and builder tests use.

### No network under test

Under `--uitesting` the phone app never starts Firebase (`FirebaseApp.configure()`, App
Check and Remote Config's listener are skipped), so nothing a UI test does can reach the
network: code that still called Firebase would crash rather than go online. Instead,
`AppBackend` (`RepCounter/Core/UITestBackend.swift`) hands every manager an in-memory fake
(`UITestConnectionRemote`, `UITestProfileRemote`, `UITestCoachingRemote`,
`UITestBackupStore`, `UITestMessagingRemote`) and a fake account, `UITestAccount`.
`AuthManager` signs that account in and out: email sign-in accepts only
`uitest@repcounter.app` / `uitest-password`, and deleting the account empties the fakes.
The fakes start empty on each launch and are compiled in Debug builds only. **New code
that reaches Firebase must go through `AppBackend`.**

### Accessibility identifiers

Every element a UI test touches has an identifier from `AID` in
`Shared/AccessibilityIdentifiers.swift`, grouped by screen (`AID.WorkoutDetail.start`,
`AID.RunWorkout.repPicker`, `AID.WatchRun.end`, …).

- The **watch** UI test target compiles `Shared/AccessibilityIdentifiers.swift` directly.
- The **phone** UI test target compiles a generated mirror,
  `RepCounterUITests/AccessibilityIdentifiers.swift`. **After changing any identifier,
  run `Scripts/sync-accessibility-identifiers.sh`.** Don't edit the mirror by hand.
- Containers that hold identified children need `.accessibilityElement(children: .contain)`
  before their own `.accessibilityIdentifier`. Otherwise the container's identifier
  replaces its children's.

### Fixtures

**The UI tests never see the app's real catalogue.** Under `--uitesting`, `DataStore.load`
seeds from a frozen test catalogue: `RepCounter/Preview Content/uitest-targets.json`,
`uitest-muscleGroups.json`, `uitest-equipments.json`, `uitest-exercises.json` and
`uitest-workouts.json`. That's a snapshot of the catalogue the tests were written against.
It ignores Remote Config, including values cached from earlier runs, and ignores the
shipping `Models/Data/*.json`. So updating the app's exercises and workouts, or
publishing a new catalogue, never breaks a UI test.

- The fixture lives in Preview Content, a development-assets folder, so Xcode leaves it
  out of App Store archives. It's copied into Debug builds only.
- **Edit the fixture only together with the tests that name its contents.** Names the UI
  tests assert against are in `SeedData` (`UITestCase.swift`) and in the `UITestSeed`
  fixtures.
- `CatalogueSyncTests.uiTestCatalogueSeeds` checks that the fixture still decodes with the
  current models and still has every workout and exercise the tests name. If a model change
  breaks it, that unit test fails first, clearly.
- The unit tests that read the **shipping** JSON (`FileJSONDecoderTests`,
  `bundledCatalogueIsIdempotent`) check only that it's valid: it decodes, ids are unique,
  workouts reference known exercises, and seeding is idempotent. They never check names or
  counts, so they keep checking real data as it changes.

`Shared/UITestSeed.swift` builds the fixture workouts on top of the catalogue: the phone
ones from its exercises, and the watch one with its own exercises, since the watch has no
catalogue. The UI tests hard-code the same names, reps and weights (`SeedData` and
`WatchSeedData`), so keep both in step.

## Running tests

- Xcode: pick the scheme and the simulator, then ⌘U, or run one test from the diamond in the
  gutter.
- Command line, building once and then running selectively:

  ```sh
  xcodebuild build-for-testing -project RepCounter.xcodeproj -scheme RepCounter \
    -destination 'platform=iOS Simulator,name=iPhone 17'
  xcodebuild test-without-building -project RepCounter.xcodeproj -scheme RepCounter \
    -destination 'platform=iOS Simulator,name=iPhone 17' \
    -only-testing:RepCounterTests/WorkoutRunRecordingTests
  ```

  The watch uses `-scheme "RepCounter Watch App"` and
  `-destination 'platform=watchOS Simulator,name=Apple Watch Ultra 4 (49mm)'`.
