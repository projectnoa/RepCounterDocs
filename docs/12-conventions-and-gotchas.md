# 12 · Conventions and gotchas

[← Testing](11-testing.md) · [README](../README.md) · Next: [Workout generation →](13-workout-generation.md)

## Code conventions

- **Naming:** PascalCase types, camelCase members. View models are nested types named
  `ViewModel`, and their JSON shapes are nested types named `Model`.
- **State:** `@State private var` for view-local state, `let` for constants, `@StateObject`
  for view models a view owns.
- **Formatting:** 4-space indentation, one blank line between members, imports at the top.
- **No force unwraps** in new code. Prefer `guard let` with an early return, or a logged
  fallback (`printLog("…", error:)`).
- **Comments** explain *why*: the constraint, the invariant, the thing that went wrong
  before. The code already says *what*.
- **Async:** use async/await and `AsyncStream`. Don't add new Combine pipelines. See
  [08](08-concurrency.md).
- **Tests:** Swift Testing for unit tests, XCUITest for UI tests
  ([11](11-testing.md)).
- **Logging:** `printLog(_:error:)` (silent in previews) or `Logger.shared` (`os.Logger`).

## Gotchas

### Project and build

- **Files are added to targets by hand.** The project doesn't use folder-synchronised
  groups. A new file used by both apps must be added to both targets. So must a new test
  helper shared by both test targets, as `SyncTestSupport.swift` is.
- **The UI tests seed from a frozen catalogue**, `Preview Content/uitest-*.json`, not the
  shipping data. Update the app's exercises and workouts freely. Edit the fixture only
  together with the tests that name its contents ([11](11-testing.md#fixtures)).
- **The phone UI tests use a generated copy of the accessibility identifiers.** Run
  `Scripts/sync-accessibility-identifiers.sh` after editing
  `Shared/AccessibilityIdentifiers.swift`.
- **`Notification` is a Core Data entity.** Inside the app module, `Notification` means the
  entity. Write `Foundation.Notification` for the `NotificationCenter` type.
- **The App Group is required.** Without the `group.buildthestack.repcounter` entitlement,
  the store can't open, and `PersistenceController` stops the app with a clear message.
- **Linked ahead of use:** the phone target links `FirebaseMessaging` and `FirebaseStorage`
  for planned features (messaging and photo upload). Nothing calls them yet.

### Core Data

- **Use the sorted getters.** `getCircuits()`, `getRounds()`, `getRoundMetrics()` and the
  rest return arrays in order. The raw `NSSet` relationships are unordered.
- **Model validation ranges are enforced on save.** `Circuit.setCount` 1–8,
  `duration`/`restTime`/`restartTime` 10–600 s, `Round.repCount` 1–99, `duration` 10–600 s,
  `weight` 0–500, `Workout.info` at least 6 characters, and more. A value out of range makes
  `save()` throw. The editor's controls and validation (title and info longer than 6
  characters, at least one focus, at least one exercise) keep user input inside these
  ranges.
- **Never hard-delete what History uses.** A workout with runs is archived
  (`Workout.archive()`), and a circuit or round with metrics is detached. Deleting any of
  them would cut History's link to its exercises, because `Round.metrics` and friends only
  nullify ([03](03-data-model.md#deleting-and-archiving)).
- **List workouts through the listed predicates.** Any new fetch of workouts to show or run
  must use `Workout.listed` (or its user and catalogue variants). Otherwise archived
  workouts reappear.
- **Change the model by adding a version.** Editing a shipped model version makes
  existing stores unreadable ([03](03-data-model.md#model-versions)).
- **Keep catalogue ids stable.** `CatalogueSync` matches by id. A changed id reads as "delete
  the old, insert a new one".
- **Main actor only.** All Core Data code runs on the main actor against the view context
  or a main-queue child. Never hand a managed object to another actor
  ([08](08-concurrency.md#core-data-and-actors)).

### The run

- **The summary must be the last card**, and no other card may have index `count - 1`.
  Reaching that index is what finishes a run on both apps.
- **Card index = position.** `RunPlan` output is enumerated, and the index is the page tag.
  Don't insert cards anywhere except `RunPlan.cards`.
- **Phone cards report on deactivation.** A round is recorded when its card is *left*. The
  two-phase activation in the phone carousel makes the leaving card report before the next
  one activates. And `finishWorkout` waits a second before recording, so the last card has
  reported.
- **Nothing above the run may observe `WorkoutManager`.** Live metrics arrive several times
  a second with a watch paired. A detail screen that observes the manager redraws the run
  on every update, which resets the cards' rep wheels. Use
  `@Environment(\.workoutManagerReader)` in views that only read the manager when
  something happens, and keep `@EnvironmentObject` for leaf views that display live values
  ([02](02-architecture.md#two-ways-to-read-workoutmanager)).
- **Keep live values out of views that draw glass.** A view that observes live metrics and
  also applies `glassEffect` redraws its glass with every update. During a swipe that
  overloads it ("glassEffect() tried to update multiple times per frame") and can crash
  the app. Draw the glass in a view that observes nothing, and put the changing readout in
  a child view that does, as `VitalsMetricsView` and its `VitalsReadout` do.
- **The run screen owns the run's view model** (`@StateObject`/`@State`, created once in
  `init`). Never build it inline in a `navigationDestination` or a cover's content closure:
  those re-run on every redraw of the presenting view.
- **Timers use dates, and time's-up only advances the active card**
  (`nextCard(with:)` checks `index == activeCard`).
- **The pickers are intentional:** the rep wheel goes from 0 to 99 and the weight wheel from
  2.5 to 300 kg in steps of 2.5 (5 to 660 lb in steps of 5). Weights are stored in kg;
  show them with `WeightUnit.current` (`format` "20 kg", `compactFormat` "20kg"), never a
  hard-coded "kg". The UI tests find the weight wheel by its value ("25kg", always kg under
  test; `XCUIElement.weightWheel`), read "15 kg" on History rows, and the rest card's length
  from its timer's label (`AID.RunWorkout.restTimer`, "Rest timer, 15 seconds"), since the
  clock's digits change every second.

### Watch and sync

- **The phone's application context carries two things,** the catalogue and the settings.
  Always send state through `SyncChannel`, which merges them. Calling
  `updateApplicationContext` directly would drop one.
- **Catalogue updates wait during a watch run** (`isRunInProgress` and
  `pendingWorkouts`). Applying one rebuilds the circuits and rounds a run is using.
- **The watch rebuilds workouts rather than diffing them.** That is only safe because the
  watch keeps no History. If the watch ever stores metrics, the importer must update in
  place, as `CatalogueSync` does.
- **Records don't rebuild workouts.** The watch compares against `ignoringRecords`.
  Otherwise every phone run would rebuild a workout on the watch.
- **In UI-test mode the watch has no `WCSession`** (`ConnectivityManager(session: nil)`),
  or the paired phone simulator's catalogue would replace the test fixture.

### HealthKit

- **The watch owns the session, and the phone only mirrors it.** A phone-only run has no
  calories or heart rate, and that is by design.
- **Never make the user wait on HealthKit.** Finishing a workout (`endCollection`,
  `finishWorkout`) can take a while, or never complete (no permission, UI tests). Stop the
  session and let it finish in the background. Every screen that shows its result needs a
  way out that doesn't depend on it.
- **`MirroredPayload` has no type tag.** Decoding tries each format in turn. Add a new
  payload type only with a format the others can't be mistaken for, or with an explicit
  tag and a fallback for old data.

### Voice counting

- **iPhone only.** watchOS has no speech recognition API ([07](07-voice-counting.md)).
- **Permission is requested when the toggle is switched on.** If it is denied, the toggle
  goes back off and an alert explains why.

## Model entities waiting for features

`Achievement`, `Goal`, `Log`, `Message` and `Notification` exist in the model with no code
using them. `Plan` and `Session` have factories and tests but little UI. `Workout.assignedTo`
and `Exercise.owner` are unused attributes. They're kept so the store needs no migration
when those features are built. (Custom exercises use `Exercise.createdBy`, a `String` uid,
rather than `owner`, which is a `UUID`.)

## Where to start for common tasks

| Task | Start here |
| --- | --- |
| Change how a run is laid out | `Shared/RunPlan.swift`, then both `loadCards` and `WorkoutRunScript.swift` in the UI tests |
| Record something new per round or set | `WorkoutCardCarouselView.ViewModel` (phone), `WatchRunRecorder` and `WorkoutRunRecord` (watch), `WorkoutRunImporter` |
| Add a catalogue field | The entity's `Model`, `CatalogueSync`, the bundled JSON, and `WorkoutSnapshot` if the watch needs it |
| Send something new between the apps | [05 · Adding a new message](05-watch-sync.md#adding-a-new-message) |
| Add a setting the watch should follow | [10 · The watch mirrors the phone's settings](10-settings-and-storage.md#the-watch-mirrors-the-phones-settings) |
| Back something new up to Firestore | Extend `WorkoutCloudStore` or add a sibling protocol, and gate it on `skipsCloudSync` |
| Change what Generate Workout makes | `RepCounter/Core/WorkoutGenerator.swift`: its instructions, the `@Generable` guides and `circuitCount(for:)` ([13](13-workout-generation.md)) |

## Previews

Every preview, in both apps, reads the same fixtures (`RepCounter/Core/PreviewContent.swift`,
compiled into both targets). They're deliberately **not** in "Preview Content": Xcode leaves
that folder out of archives, but `#Preview` bodies still compile in Release, so archiving
failed with "Cannot find 'PreviewFixtures' in scope".

- **`PreviewFixtures`** builds, once and on first use, a small but real slice of the
  catalogue (names, images, muscle groups and equipment as the seed JSON has them) into the
  in-memory preview store: *Push Day* (the user's own: barbell, dumbbell, bodyweight and a
  timed hold, with notes), *Leg Day* (catalogue, favourite), *Coach's Pull Day* (assigned
  by a coach), *Quick Core* (never run), nine weeks of runs that get stronger week by week,
  the latest with feedback, and a weekly schedule.
- Use the **named fixtures**, never array indices: `PreviewFixtures.pushDay`, `.history`,
  `.round(.weighted | .bodyweight | .timed)`, `.exercise(_:)`, `.circuit`,
  `.exerciseCard(_:)`, `.records`, `.runSummary`, `.liveRun`, and on the phone
  `.progressSummary`, `.historyViewModel`, `.cloudRun`, `.runFeedback`, `.profileUser`.
- End every preview with **`.previewEnvironment()`** (`timeWorkout:` for run cards): it
  injects the store and every object the views read from the environment. On the phone it
  also configures Firebase from the bundled plist, since previews don't run the app's
  launch and `AuthManager` and `RemoteConfigProvider` read Firebase as they're made.
- Give a preview its own state with `@Previewable @State`.
- The watch simulator's empty store is seeded with the same fixtures
  (`PreviewFixtures.populate(into:)`) until the phone sends its workouts.
- **Watch previews with a navigation bar crash on the watchOS 27 preview runtime.** A
  private UIKit function behind the new bar and scroll-edge ("pocket") effect switches on
  `userInterfaceIdiom` and traps on a value the preview host reports (symbolicated from
  the crash: an unhandled idiom falls through to `brk #1`). It's in UIKit, not the app, and
  the app itself runs fine. So watch previews don't wrap views in a `NavigationStack` they
  don't need. Screens that build their own (`HomeView`, `RootView`, `WorkoutListView`,
  `MetricsView`, `LiveRunView`) crash there until Apple fixes it: preview them on a watchOS
  11.5 simulator, which Xcode's canvas device menu offers.
- Screens don't ask for HealthKit permission in previews, which can't present the sheet.

