# 02 · Architecture

[← Getting started](01-getting-started.md) · [README](../README.md) · Next: [Data model →](03-data-model.md)

## The shape of the app

RepCounter is two apps that share a model and a contract:

```
            ┌──────────────────────── iPhone ─────────────────────────┐
            │  SwiftUI views ─ view models ─ managers (@MainActor)     │
            │        │                         │         │            │
            │   Core Data store           Firebase    HealthKit       │
            │   (App Group)          (Auth/RC/Firestore) (mirrored)   │
            └────────▲───────────────────────────────────▲────────────┘
                     │ WatchConnectivity                  │ HKWorkoutSession
                     │ (SyncMessage: catalogue,           │ mirroring
                     │  settings, runs, end workout)      │ (MirroredPayload:
            ┌────────▼───────────────────────────────────▼─ statistics,
            │  SwiftUI views ─ view models ─ managers      elapsed time, water)
            │   Core Data store (own copy)   HealthKit (owns the session)│
            └──────────────────────── Watch ──────────────────────────┘
```

- **The phone is the source of truth** for the catalogue, the user's own workouts,
  History, plans and schedules. It backs up workouts and History to Firestore.
- **The watch keeps a copy** of the workouts, rebuilt from what the phone sends. It keeps
  no History of its own. A run on the watch is sent to the phone as a record and saved
  there.
- **Two channels connect them:** WatchConnectivity for app-level messages
  ([05](05-watch-sync.md)), and HealthKit session mirroring for live workout data
  ([06](06-healthkit.md)).

## Targets and what they compile

| Target | Sources |
| --- | --- |
| `RepCounter` | `RepCounter/**`, `Shared/**`, most of `RepCounter/Models/**` |
| `RepCounter Watch App` | `RepCounter Watch App/**`, `Shared/**`, the model files and some custom views from `RepCounter/` (for example `TimerView`, `GlassWindowView`, `DragDownStampView`, and the entity extensions) |
| `RepCounterTests` | Phone unit tests, hosted by the phone app (`@testable import RepCounter`) |
| `RepCounter Watch AppTests` | Watch unit tests, hosted by the watch app, and `SyncTestSupport.swift`, which the phone tests also compile |
| `RepCounterUITests`, `RepCounter Watch AppUITests` | XCUITest suites |

Files are listed in the project explicitly (no folder-synchronised groups). **A new file
must be added to each target that needs it.** When phone and watch share a file,
platform differences live behind `#if os(watchOS)` (see `WorkoutManager.swift`) or in
per-platform extension files (`WorkoutManager+iOS.swift`, `WorkoutManager+watchOS.swift`).

### What lives in `Shared/`

`Shared/` holds the code both apps must agree on. A change here is a change to both apps
at once, which is the point:

| File | Why it is shared |
| --- | --- |
| `WatchSync.swift` | The message contract. What one side sends is what the other reads. |
| `WorkoutTransfer.swift` | The wire formats (`WorkoutSnapshot`, `WorkoutRunRecord`), the rules for splitting calories and time across sets (`SetSplits`), run numbering (`History.nextRun`), and both importers |
| `RunPlan.swift` | The card sequence of a run, so phone and watch lay out a workout identically |
| `PersonalRecords.swift` | The "last record" rules |
| `Weight.swift` | `WeightUnit` (kg/lb display and pickers) and `SetPerformance` (how sets compare) |
| `WorkoutManager*.swift`, `HealthManager.swift` | HealthKit |
| `PersistenceController.swift` | The Core Data stack |
| `UITestConfiguration.swift`, `UITestSeed.swift`, `AccessibilityIdentifiers.swift` | Test hooks used by both apps' UI tests |

## Managers: long-lived state

The apps use a handful of `@MainActor` `ObservableObject` managers, created once and
handed to views through the environment:

| Manager | Phone | Watch | Responsibility |
| --- | --- | --- | --- |
| `AuthManager` | ✓ | | Firebase user and `AuthState` (`signedOut`, `authenticated` = anonymous, `signedIn`) |
| `RemoteConfigProvider` | ✓ | | Fetches catalogue JSON and publishes `freshDataAvailable` |
| `HealthManager` | ✓ | ✓ | `HKHealthStore`, authorisation, today's calories, steps, distance and exercise time |
| `WorkoutManager` | ✓ | ✓ | The HealthKit workout session and its live statistics (see [06](06-healthkit.md)) |
| `ConnectivityManager` | ✓ | ✓ | The WatchConnectivity session and `SyncChannel`, and what each app does with incoming messages |

Where they are created:

- **Phone:** `RepCounterApp.init` creates them as `@StateObject`s and injects them with
  `.environmentObject(_:)`. `AppDelegate` (via `@UIApplicationDelegateAdaptor`) only
  activates `WCSession` early.
- **Watch:** `AppDelegate` (via `@WKApplicationDelegateAdaptor`) owns them. That is because
  HealthKit can launch the watch app with a workout to run (`handle(_ workoutConfiguration:)`)
  before any view exists.

The managers are not singletons. Anything that needs one is given it, which makes
them easy to replace in tests. For example, `ConnectivityManager` takes a transport, a
session, a context and a `UserDefaults`. The only `static let shared` instances are
stateless helpers or process-wide resources: `PersistenceController.shared` (the store),
`FirestoreWorkoutStore.shared` (the default `WorkoutCloudStore`),
`AppleSignInManager.shared` and `GoogleSignInManager.shared` (the sign-in flows), and
`Logger.shared` (`os.Logger`).

### Two ways to read `WorkoutManager`

`WorkoutManager` publishes heart rate and energy several times a minute. A view that
declares `@EnvironmentObject var workoutManager` redraws on every update. For the run
carousel that would rebuild every card. So there are two access paths:

- `@EnvironmentObject var workoutManager: WorkoutManager` for views that **display** live
  values (`VitalsMetricsView`, watch `MetricsView`).
- `@Environment(\.workoutManagerReader) private var workoutManager`, an `@Entry`
  environment value in `Shared/WorkoutManager+Environment.swift`, for views that only
  **read** it when something happens, such as leaving a card or ending a run. Reading
  through the environment value does not subscribe the view to changes.

Both apps inject both: `.environmentObject(workoutManager)` and
`.environment(\.workoutManagerReader, workoutManager)`.

**No view above the run may observe `WorkoutManager`.** That includes the workout detail
screens, which push or present the run. A redraw there re-runs the closure that builds
the run. The phone once built the run's view model inside `navigationDestination`, so
every heart-rate update from a paired watch replaced it: the carousel flickered, every
rep wheel went back to 0, and the rounds recorded so far were lost. On the watch, the
redraws snapped the rep wheel back while it was being turned. Now:

- The detail screens don't hold the manager at all. The HealthKit types they request and
  `startWatchWorkout` are `static` on `WorkoutManager`, and the watch detail screen reads
  it through `workoutManagerReader` only to reset it after the summary.
- The run screen owns its view model: `@StateObject` on the phone's `WorkoutControlView`,
  and `@State` on the watch's, created once per run in `init`.
- Carousel cards take their identity from their index (`CarouselItem.id`), so even laying
  the same run out again keeps each card's state.

`RunWorkoutUITests.testLiveMetricsDoNotResetTheRepCount` and
`WatchWorkoutUITests.testLiveMetricsDoNotResetTheRepCount` guard this, using the
`--uitesting-live-metrics` launch argument ([11](11-testing.md)).

## View models

Most screens pair a `View` with a nested `ViewModel` (`WorkoutDetailView.ViewModel`,
`WorkoutCardCarouselView.ViewModel`, …). There are three kinds:

- **Plain structs** built from a Core Data object. Most row and card view models are like
  this: they copy what the view shows (title, image, counts) so the view does not reach
  into managed objects.
- **`ObservableObject` classes** where the view model is edited in place and the view
  must redraw, such as the workout editor's row models (`CircuitItemView.ViewModel`,
  `RoundItemView.ViewModel`) and the editor itself.
- **`@MainActor` classes with behaviour**, such as `WorkoutCardCarouselView.ViewModel` on
  the phone, which records the run, and `WatchRunRecorder` on the watch.

Views hold view models in `@StateObject` or `@State` when they own them. For example, the
watch `WorkoutControlView` keeps its carousel view model in `@State`, so a redraw does not
reset the run's recording. When a view is handed a view model, it takes it as a plain
property.

## Navigation

### Phone

`RootView` switches between three `ScreenView`s:

- `.splashScreen`: `SplashView` loads the catalogue, then switches to `.mainView`.
- `.mainView`: a `TabView` with **Home** and **Workouts**. On iOS 26 this uses the `Tab`
  API with `.sidebarAdaptable` and the Workouts tab in the search role; before iOS 26
  it uses classic `.tabItem`s. Each tab has its own `NavigationStack`.
- `.metricsView`: `MirroringWorkoutView`, the live metrics of a workout running on the
  watch. You get there from the *Active Workout → Continue* button on Home.

From Home, full-screen covers open Schedule, History, the exercise list, Settings, login
and the user profile. From the Workouts tab, `WorkoutListView` pushes
`WorkoutDetailView`, which pushes `WorkoutControlView` (the run) and presents
`WorkoutEditView` for user workouts.

`ConnectivityManager.refreshApp` is toggled after *Reset data* in Settings. `RootView`
changes the tab view's `.id` in response, which rebuilds the whole tree against the reset
store.

### Watch

`RootView` shows:

- While `ConnectivityManager.showExerciseMetrics` is true (the phone started a workout):
  `LiveRunView`, the phone's cards kept in step with it, or `MetricsView`, live metrics
  only, until the run's state arrives ([05](05-watch-sync.md#live-runs-the-phones-cards-on-the-watch)).
- Otherwise a vertically paged `TabView` of `HomeView` (today's totals) and
  `WorkoutListView`. Picking a workout pushes `WorkoutDetailView`, whose **Start** presents
  `WorkoutControlView`: a paged view with the controls page (End) and the run carousel.

## A run, end to end

```
WorkoutDetailView ──Start──▶ HealthKit authorisation sheet (healthDataAccessRequest)
   │                              │
   │                              ├─▶ WorkoutControlView ─▶ WorkoutCardCarouselView (phone run)
   │                              └─▶ WorkoutManager.startWatchWorkout
   │                                     └─ HKHealthStore.startWatchApp(toHandle:)
   │                                           └─ watch AppDelegate.handle(_:) starts the
   │                                              HKWorkoutSession and mirrors it back
   ▼
Summary reached / End pressed ─▶ ViewModel.finishRun ─▶ History saved ─▶ Firestore
```

A run started on the watch instead goes watch `WorkoutDetailView` → `WorkoutControlView` →
`WatchRunRecorder` → `SyncMessage.workoutRun` → phone `WorkoutRunImporter` → History. The
details are in [04](04-workout-run.md) and [05](05-watch-sync.md).

## Where to put new code

- A service shared across screens is a manager, owned by the app and injected.
- A rule the phone and watch must agree on goes in `Shared/` as a pure function or type,
  with tests on both sides. Examples: `RunPlan`, `SetSplits`, `PersonalRecords`.
- Anything that touches Firestore goes behind a small protocol like `WorkoutCloudStore`,
  so it can be tested with a spy and skipped under test.
- Every interactive element a UI test needs gets an `AID` identifier in
  `Shared/AccessibilityIdentifiers.swift`. Then run `Scripts/sync-accessibility-identifiers.sh`
  (see [11](11-testing.md)).
