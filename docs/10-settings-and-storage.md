# 10 · Settings and local storage

[← Firebase](09-firebase.md) · [README](../README.md) · Next: [Testing →](11-testing.md)

## Persisted preferences

Everything except the Core Data store lives in `UserDefaults.standard`, usually through
SwiftUI's `@AppStorage`. The key strings are the stored names, so renaming one resets
everyone's setting.

| Key | Type / default | App | Set by | Read by |
| --- | --- | --- | --- | --- |
| `showExercisePreview` (`RunSettings.showExercisePreviewKey`) | `Bool`, `true` | Phone | Settings → *Show exercise previews* | `WorkoutConfig.showExercisePreview`, which becomes `RunPlan.Options.showsExercisePreviews` |
| `showExercisePreview` | `Bool`, `true` | Watch | The phone, via `SyncMessage.settings` | `RunSettings.stored()` when a watch run is laid out |
| `weightUnit` (`WeightUnit.storageKey`) | `WeightUnit` raw value, `"kg"`/`"lb"`; the region's usual unit until chosen | Phone | Settings → *Weight unit* | `WeightUnit.current`, everywhere a weight is shown or picked. UI tests always get kg |
| `showSpeechRecognitionOption` | `Bool`, `true` | Phone | Settings → *Ask to listen for count* | Whether the detail screen offers *Listen for count* |
| `hasSeenIntroGuide` (`IntroGuideView.seenKey`) | `Bool`, `false` | Phone | Finishing or skipping the first-launch slideshow | `FirstLaunchCover`, which shows the slideshow before login until it's seen |
| `tips.resetOnNextLaunch` | `Bool` | Phone | Settings → *Show Tips Again* before iOS 26 | `AppTips.configure()`, which clears TipKit's store before configuring it |
| `isFirstTimeLaunch` | `Bool`, `true` | Phone | It is the `isPresented` binding of Home's first-launch cover (`LoginView`, or `WelcomeBackView` when signed in), so dismissing the cover sets it to `false` | Home asks for HealthKit once it turns `false` |
| `personalRecords` | `[String: Int]` (exercise UUID → reps) | Watch | `PersonalRecords.store(merging:)`: catalogue from the phone and the watch's own runs | `PersonalRecords.stored()` at the start of each watch run |

`@SceneStorage("selectedTab")` restores the selected tab per scene. It is not a
preference.

### The watch mirrors the phone's settings

The watch has no settings screen. **Show exercise previews** is set on the phone and sent
to the watch as `RunSettings` ([05](05-watch-sync.md)). It goes out when the session
activates, when the watch app is installed, and whenever the toggle changes. So both
apps lay out runs the same way. To sync another setting:

1. Add a property to `RunSettings` in `Shared/WatchSync.swift`. Give `stored(in:)` a
   default so older payloads still decode, and write it in `store(in:)`.
2. Read it where the watch needs it.
3. If the phone toggle writes through `@AppStorage`, add
   `.onChange { connectivityManager.publishSettings() }` beside it, as `SettingsView` does.

## Per-run options: `WorkoutConfig`

`WorkoutConfig` is an `ObservableObject` created by each workout's detail screen and
passed to the run with `.environmentObject`. Its values last only for that screen. They are
**not** saved:

| Property | Phone default | Watch default | Effect |
| --- | --- | --- | --- |
| `timeWorkout` | `false` | `true` | Cards show a countdown and advance on their own ([04](04-workout-run.md#3-timers-and-time-workout)) |
| `listenForCount` | `false` | — | Voice counting on exercise cards ([07](07-voice-counting.md)) |

The phone's `WorkoutConfig` also exposes `showExercisePreview` and
`showSpeechRecognitionOption` through `@AppStorage`, so the detail screen reads the saved
settings from one place.

## Settings screen (phone)

`Views/Settings/SettingsView.swift`:

- **Show exercise previews:** see above.
- **Weight unit:** kilograms or pounds. Weights are stored in kg (`weightKg`), so switching
  only changes how they're shown and the picker steps (2.5 kg or 5 lb).
- **Ask to listen for count:** see above.
- **Help:** *How RepCounter Works* replays the intro slideshow. *Show Tips Again* makes
  every TipKit tip eligible again (`AppTips.resetAll()`): at once on iOS 26 and later, on
  the next launch before that, since `resetEligibility()` is iOS 26-only and
  `Tips.resetDatastore()` has to run before TipKit is configured.
- **Reset data:** after confirmation, `UserDataReset.eraseWorkoutData` deletes the
  signed-in account's Firestore backup first (`CloudSync.deleteBackup()`), then the device
  (`DataStore.resetCoreData` batch-deletes every entity, the catalogue is reloaded, and the
  `coaching.seen`/`personalRecords` defaults are cleared), then publishes an empty progress
  summary. `ConnectivityManager.refreshApp` rebuilds `RootView`. The account, profile and
  connections stay. If the backup can't be deleted (offline), nothing is erased and an
  alert says so: erasing only the device would let the next sync restore everything.
- App version and contact links.

## Files on disk

| What | Where |
| --- | --- |
| Core Data store | App Group container `group.buildthestack.repcounter`, `CoreData/RepCounter.sqlite` ([03](03-data-model.md)) |
| Seed catalogue | App bundle, `targets.json`, `muscleGroups.json`, `equipments.json`, `exercises.json`, `workouts.json` |
| Home greeting messages | App bundle, `messages.json` (`getRandomStringForToday()` picks one per day, from a generator seeded with the date) |
| Exercise animations | App bundle, `Apngs/<Target>/<image>.apng` with stills in `Stils/` |

## Tips (TipKit)

`Views/Tips/AppTips.swift`. `AppTips.configure()` runs at launch (`RepCounterApp.init`)
with `.displayFrequency(.immediate)`; under XCUITest it calls
`Tips.hideAllTipsForTesting()` first, so no popover covers what a test taps. Each tip is
invalidated when its feature is used:

| Tip | Where | Style | Notes |
| --- | --- | --- | --- |
| `StartWorkoutTip` | Workout detail, Start | Popover | Sets `hasStartedWorkout` |
| `ListenForCountTip` | Workout detail, Listen for count | Inline | Only after a first start, so the detail screen never shows two tips |
| `RunCardsTip` | The active exercise card's wheels | Popover (below) | Leaving the card sets `hasMovedOn` |
| `EndWorkoutTip` | Run, End | Popover | Only once `hasMovedOn`, so it never overlaps the card tip |
| `CreateWorkoutTip` | Top of the Workouts list, signed in | Inline | A popover on the tab's toolbar button didn't show reliably |
| `BuildWorkoutTip` | Top of the workout editor | Inline (list row) | Put away on save |
| `AddCoachTip` | Profile, above People | Inline | Put away when People opens |
| `ShareCodeTip` | People, My Code | Popover | |
| `DeleteRunTip` | History, above the workouts | Inline | Put away on delete |

Inline tips use `.inlineTip(_:)` (or, in the editor's list, a row shown from
`shouldDisplayUpdates`) rather than a bare `TipView`: an ineligible `TipView` still takes
a stack's spacing, or a blank list row.
