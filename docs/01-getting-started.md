# 01 · Getting started

[← README](../README.md) · Next: [Architecture →](02-architecture.md)

## Requirements

- **Xcode 27** (the project currently builds with Xcode 27.0 and the iOS/watchOS 27 SDKs).
  The code calls iOS 26 APIs, such as `glassEffect` and the `Tab` API with
  `.tabBarMinimizeBehavior`, behind `#available` checks, so any SDK from 26 up works.
- **Deployment targets:** iOS 17, watchOS 10.
- **Swift:** the app targets and all four test targets use the Swift 6 language mode with
  complete strict concurrency checking. See [08 · Concurrency](08-concurrency.md).
- **An Apple Developer team** for HealthKit, Sign in with Apple and App Group capabilities
  on a device. The simulator works without one for most flows.

## Swift packages

Xcode resolves these automatically on first open:

| Package | Used for |
| --- | --- |
| Firebase (Core, Auth, Firestore, Remote Config, Analytics) | Sign-in, catalogue delivery, cloud backup of workouts and History |
| GoogleSignIn / GoogleSignInSwift | Google sign-in button and flow |
| SDWebImage / SDWebImageSwiftUI | Animated PNG (`.apng`) exercise demonstrations (`AnimatedImage`) |
| WrappingHStack | Wrapping rows of equipment and muscle-group icons |

The phone target also links `FirebaseMessaging`, `FirebaseStorage` and `Speech.framework`.
Speech is used (see [07](07-voice-counting.md)). Messaging and Storage are linked ahead of
planned features, in-app messaging and photo upload, and nothing calls them yet.

## Configuration files

| File | Purpose |
| --- | --- |
| `RepCounter/GoogleService-Info.plist` | Firebase project configuration. `FirebaseApp.configure()` in `RepCounterApp.init` reads it. Replace it to point at a different Firebase project. |
| `RepCounter/RepCounter.entitlements` | Sign in with Apple, HealthKit (with background delivery), iCloud container, App Group `group.buildthestack.repcounter` |
| `RepCounter Watch App/RepCounter Watch App.entitlements` | HealthKit, iCloud container, the same App Group identifier |
| `RepCounter/Info.plist`, `RepCounter-Watch-App-Info.plist` | Usage strings for HealthKit, microphone and speech recognition are set as `INFOPLIST_KEY_*` build settings |

The Core Data store lives in the App Group container
(`PersistenceController`, see [03](03-data-model.md)). If the entitlement is missing, the app
stops with `App Group container not found` on first launch. Check signing first when you
see that.

## Schemes and destinations

| Scheme | Builds | Tests |
| --- | --- | --- |
| `RepCounter` | Phone app, and the embedded watch app | `RepCounterTests`, `RepCounterUITests` (test plan `RepCounter.xctestplan`) |
| `RepCounter Watch App` | Watch app | `RepCounter Watch AppTests`, `RepCounter Watch AppUITests` (test plan `RepCounter Watch App.xctestplan`) |

The simulators the project is developed against are **iPhone 17** and **Apple Watch Ultra
4 (49mm)** paired with it. Watch features that need the phone (catalogue sync, mirrored
sessions, runs sent back) only work when both simulators are booted as a pair.

> Check the run destination before you press ⌘U. With a physical iPhone selected, the UI
> tests will drive your own phone.

## First launch, step by step

1. `RepCounterApp.init` configures Firebase and creates the long-lived managers
   (`AuthManager`, `RemoteConfigProvider`, `HealthManager`, `WorkoutManager`,
   `ConnectivityManager`). It also starts a Remote Config fetch.
2. `RootView` shows `SplashView`. When Remote Config reports fresh values, or the store has
   no exercises yet, `DataStore.load` runs `CatalogueSync` to add and update the catalogue
   in place.
3. The tab view appears. The first time, Home presents a full-screen cover
   (`FirstLaunchCover`): the intro slideshow (`IntroGuideView`, 7 pages on starting and
   finishing a workout, making one, listening and coaches), then `LoginView`: Sign in with
   Apple, Google, or continue anonymously. The `isFirstTimeLaunch` flag controls the
   cover, and `hasSeenIntroGuide` the slideshow.
4. After the login cover is dismissed, Home asks for HealthKit access for today's activity
   rings and calls `WorkoutManager.retrieveRemoteSession`. That way the phone is ready to
   mirror a watch workout.

## Useful launch arguments

The app reads a set of `--uitesting…` arguments (see [11 · Testing](11-testing.md)). They
are handy for manual work too. Add them under *Edit Scheme → Run → Arguments*:

- `--uitesting` alone: an in-memory store seeded from the bundled JSON, no Remote Config,
  no Firebase Auth (you appear signed in), no HealthKit prompt, no Firestore writes.
- `--uitesting --uitesting-long-workout` also seeds a multi-circuit workout. It's useful for
  working on the run carousel.
- On the watch, `--uitesting --uitesting-watch-workout` opens a fixture workout directly.

## Project layout in more detail

```
RepCounter/
  RepCounterApp.swift          @main: creates managers, injects them into the environment
  Core/
    AppDelegate.swift          Activates WCSession at launch
    AuthManager.swift          Firebase Auth state, Apple/Google/anonymous sign-in, account deletion
    AppleSignInManager.swift   Sign in with Apple request + nonce
    GoogleSignInManager.swift  Google sign-in flow
    RemoteConfigProvider.swift Fetches the catalogue JSON from Remote Config
    DataStore.swift            Catalogue load, store reset, Firestore writes, WorkoutCloudStore
    CatalogueSync.swift        In-place upsert of the catalogue
    ConnectivityManager.swift  Phone side of WatchConnectivity
    SpeechRecognizer.swift     SFSpeechRecognizer + AVAudioEngine wrapper (an actor)
    TranscriptionEngine.swift  Turns transcripts into rep counts
  Models/                      Entity extensions (factories, getters) and Codable `Model`s
  Views/
    RootView.swift, SplashView.swift
    Home/                      Home tab: greeting, activity, today's schedule, actions, favourites
    Workout/                   List, filter, detail, editor, run (Activity/), summary
    Exercise/                  Exercise list, filter, detail, run cards
    Circuit/, Round/           Rows and settings sheets for the editor, circuit run card
    History/                   History list, all workouts with search and filters, per-run detail, trends, badges and records
    Schedule/                  Calendar and scheduling of workouts
    Auth/                      Login, welcome back, user profile
    Settings/                  Settings screen
    Custom/                    Reusable pieces: cards, glass, timers, rep ring, modifiers
RepCounter Watch App/
  RepCounterApp.swift          @main for the watch; the AppDelegate owns the managers
  Core/
    AppDelegate.swift          Handles workouts started from the phone
    ConnectivityManager.swift  Watch side of WatchConnectivity
    WatchRunRecorder.swift     Collects a run into a WorkoutRunRecord
  Views/                       Home, workout list and detail, run carousel and cards,
                               controls, live metrics, summary
Shared/
  WatchSync.swift              SyncMessage, SyncTransport, SyncChannel, MirroredPayload
  WorkoutTransfer.swift        WorkoutSnapshot, WorkoutRunRecord, SetSplits, importers
  RunPlan.swift                The card sequence of a run
  PersonalRecords.swift        Best reps per exercise
  WorkoutManager*.swift        HealthKit session, mirroring, statistics
  WorkoutManager+Environment.swift  workoutManagerReader environment value
  HealthManager.swift          HealthKit authorisation and today's totals
  PersistenceController.swift  The Core Data stack
  UITestConfiguration.swift, UITestSeed.swift  Test launch hooks and fixtures
  AccessibilityIdentifiers.swift  Identifiers shared with the UI tests (AID)
  Theme.swift, Global.swift, …  Small shared helpers
```
