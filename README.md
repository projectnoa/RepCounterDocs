# RepCounter

RepCounter is an iPhone and Apple Watch app for bodyweight and strength workouts. You pick
or build a workout of circuits and rounds. The app then walks you through it one card at
a time: a timer on each card, a rep wheel to log what you did, and your last record to
beat. Every run is saved to History. With an Apple Watch you also get live heart rate and
calories. The watch can run workouts on its own and send the results back to the phone.

The app is built with SwiftUI, Core Data, HealthKit, WatchConnectivity, the Speech
framework and Firebase (Auth, Remote Config and Firestore). Both apps compile in the
Swift 6 language mode with complete concurrency checking.

## At a glance

| | iPhone app | Watch app |
| --- | --- | --- |
| Target | `RepCounter` | `RepCounter Watch App` |
| Minimum OS | iOS 17 | watchOS 10 |
| Role | Owns the catalogue, the user's workouts, History and the cloud copy | Runs workouts with live HealthKit metrics |
| Data | Core Data store in the App Group container | Its own Core Data store, filled from the phone |
| Runs a workout | Yes, with or without a watch | Yes, and sends the run to the phone |
| Voice rep counting | Yes (Speech framework) | No (watchOS has no speech recognition) |
| HealthKit session | Mirrors the watch's session | Owns the session |

## Quick start

1. Open `RepCounter.xcodeproj` in Xcode. It is currently built with Xcode 27 and the
   iOS/watchOS 27 SDKs. The code uses iOS 26 APIs (Liquid Glass, the new `Tab` API)
   behind availability checks, so an SDK of 26 or later is required.
2. Choose the **RepCounter** scheme and an **iPhone 17** simulator. The scheme also
   builds and embeds the watch app.
3. Build and run. On first launch the splash screen seeds the catalogue, from Firebase
   Remote Config or, failing that, from the JSON bundled in `RepCounter/Models/Data`.
4. To try the watch, run the **RepCounter Watch App** scheme on the **Apple Watch Ultra 4
   (49mm)** simulator paired with the iPhone 17 simulator.

[Getting started](docs/01-getting-started.md) covers signing, Firebase, the App Group and
the simulators in detail.

## Documentation

Start with the architecture page, then read whichever topic you're working on.

| Page | What it covers |
| --- | --- |
| [01 · Getting started](docs/01-getting-started.md) | Requirements, setup, schemes, simulators, first run, project layout |
| [02 · Architecture](docs/02-architecture.md) | Targets, the `Shared` folder, app lifecycle, navigation, managers and how they reach views |
| [03 · Data model and persistence](docs/03-data-model.md) | Core Data entities and relationships, the store, catalogue vs user data, `CatalogueSync`, editing workouts |
| [04 · The workout run](docs/04-workout-run.md) | `RunPlan`, the card carousels, timers, recording a run, calories and durations per set, ending early, personal records |
| [05 · Phone ↔ watch sync](docs/05-watch-sync.md) | The WatchConnectivity contract, delivery rules, what each side sends, the catalogue and run importers |
| [06 · HealthKit and live metrics](docs/06-healthkit.md) | Workout sessions, mirroring to the phone, statistics, water, today's activity |
| [07 · Voice rep counting](docs/07-voice-counting.md) | `SpeechRecognizer`, `TranscriptionEngine`, permissions, why it is phone-only |
| [08 · Concurrency in this codebase](docs/08-concurrency.md) | A refresher on Swift concurrency (actors, `@MainActor`, `Sendable`, `nonisolated`, `AsyncStream`) as this project uses it, plus Core Data threading and where Combine remains |
| [09 · Firebase and cloud services](docs/09-firebase.md) | Auth (Apple, Google, anonymous), Remote Config, Firestore, the `WorkoutCloudStore` seam |
| [10 · Settings and local storage](docs/10-settings-and-storage.md) | Every `UserDefaults` / `@AppStorage` key, per-run options, what syncs to the watch |
| [11 · Testing](docs/11-testing.md) | Test targets, Swift Testing, the Core Data test stack, the WatchConnectivity loopback, UI test launch arguments |
| [12 · Conventions and gotchas](docs/12-conventions-and-gotchas.md) | Code style, recurring patterns, known intricacies and pitfalls, unused parts of the model |
| [13 · Workout generation](docs/13-workout-generation.md) | Drafting a workout's plan with Apple's on-device model (FoundationModels): availability, choosing exercises, the `@Generable` types, checking the answer |

## Repository layout

```
RepCounter/                    iPhone app
  Core/                        Services: auth, Remote Config, data store, catalogue sync,
                               WatchConnectivity, speech, workout generation
  Models/                      Core Data classes, their helpers and Codable models
    CoreData/                  Generated NSManagedObject subclasses
    Data/                      Bundled catalogue JSON (seed data) and home messages
    Helpers/                   Sort descriptors and formatters
  Views/                       SwiftUI screens, grouped by feature
RepCounter Watch App/          Watch app (Core/ for services, Views/ for screens)
Shared/                        Code compiled into both apps: sync contract, run plan,
                               transfer models, personal records, HealthKit workout manager,
                               persistence, test hooks
RepCounter.xcdatamodeld/       The Core Data model, shared by both apps
RepCounterTests/               Phone unit tests (Swift Testing)
RepCounterUITests/             Phone UI tests (XCUITest)
RepCounter Watch AppTests/     Watch unit tests (Swift Testing)
RepCounter Watch AppUITests/   Watch UI tests (XCUITest)
Scripts/                       sync-accessibility-identifiers.sh
docs/                          This documentation
```

## The core flow in one paragraph

On launch the phone shows a splash screen. It brings the exercise and workout catalogue
up to date in Core Data ([03](docs/03-data-model.md)), then opens a tab view with Home and
Workouts. Picking a workout opens its detail screen, which has two per-run switches: *Time
workout* and *Listen for count*. **Start** asks for HealthKit access and opens the run
carousel ([04](docs/04-workout-run.md)). The same call starts the watch app, whose
HealthKit session the phone then mirrors for live metrics ([06](docs/06-healthkit.md)).
Cards are laid out by the shared `RunPlan`. As you move through them the view model
records each set and round. Reaching the summary, or ending early, saves a `History`
entry and backs it up to Firestore ([09](docs/09-firebase.md)). The phone keeps the watch's
copy of the workouts, records and settings up to date. A run started on the watch comes
back to the phone as a `WorkoutRunRecord`, which the phone saves as History too
([05](docs/05-watch-sync.md)).
