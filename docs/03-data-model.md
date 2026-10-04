# 03 · Data model and persistence

[← Architecture](02-architecture.md) · [README](../README.md) · Next: [The workout run →](04-workout-run.md)

## Core Data in one minute (refresher)

- An **`NSPersistentContainer`** wraps the model (`RepCounter.xcdatamodeld`), the store
  file (SQLite) and a **view context** (`viewContext`). The view context is a main-queue
  `NSManagedObjectContext` that SwiftUI reads through `@Environment(\.managedObjectContext)`
  and `@FetchRequest`.
- **Managed objects** (`Workout`, `Round`, …) belong to exactly one context and may only
  be touched on that context's queue. For the view context, that means the main actor.
  Passing a managed object to another thread or actor is a bug, even if it compiles.
- **Child contexts** (`context.parent = viewContext`) are scratch pads: `save()` on a child
  pushes its changes into the parent, and only the parent's `save()` writes to disk.
  Throwing the child away discards everything in it. The workout editor uses one for new
  workouts, so *Cancel* leaves no trace.
- **Relationship delete rules** decide what happens to related objects on delete:
  *Cascade* deletes them too, and *Nullify* just clears the link.

This project keeps all Core Data work on the **main actor**, against the view context or a
main-queue child of it. Nothing runs on background contexts. See
[08 · Concurrency](08-concurrency.md#core-data-and-actors).

## The stack: `PersistenceController`

`Shared/PersistenceController.swift`, `@MainActor`:

- `PersistenceController.shared` is the app's store. Under XCUITest
  (`UITestConfiguration.usesInMemoryStore`) it is an in-memory store instead, so tests never
  touch real data.
- The SQLite file lives in the **App Group container** `group.buildthestack.repcounter`, at
  `CoreData/RepCounter.sqlite`, with `completeUntilFirstUserAuthentication` file
  protection so saves also work while the device is locked after first unlock.
- The view context has `automaticallyMergesChangesFromParent = true` and the
  `mergeByPropertyObjectTrump` merge policy.
- `PersistenceController.preview` is an empty in-memory store for SwiftUI previews, which
  `PreviewFixtures` fills the first time a preview asks ([12](12-conventions-and-gotchas.md#previews)).
- In-memory stores use a SQLite store at `/dev/null` rather than `NSInMemoryStoreType`,
  because only SQLite supports `NSBatchDeleteRequest` (used by *Reset data*).

The watch uses the same class and model, in its own store on the watch.

## Entities

```
Target ◀──── Exercise ────▶ MuscleGroup, Equipment (+ optional), Tag
  ▲             ▲
  │             │ exercise
Workout ──circuits (Cascade)──▶ Circuit ──rounds (Cascade)──▶ Round
  │  ▲                            ▲                              ▲
  │  │ workout                    │ circuit                      │ round
  │  └──────── History ──circuitMetrics (Cascade)──▶ CircuitMetric ──roundMetrics──▶ RoundMetric
  │
  └── schedules ▶ Schedule ◀── Plan ──▶ Session ◀── workout
```

### Catalogue entities

| Entity | Key attributes | Notes |
| --- | --- | --- |
| `Target` | `id: Int16`, `name`, `icon`, `image` | A workout focus (Chest, Legs, …). Integer ids from the catalogue. |
| `MuscleGroup` | `id: Int16`, `name`, `icon`, `info` | |
| `Equipment` | `id: Int16`, `name`, `icon`, `info` | `Exercise` has both `equipments` and `optionalEquipments`. |
| `Exercise` | `id: UUID`, `name`, `alternateName`, `info`, `instructions`, `image`, `bodySegment`, `difficulty`, `isFavorite`, `isTimed`, `createdBy: String?`, `isInLibrary`, `symbol` | `image` names an animated PNG in the bundle (`<image>.apng`, with a still in `Stils/`). `rounds` cascades. A **custom exercise** has `createdBy` (its creator's uid), no `image`, and an SF Symbol in `symbol`; see [Custom exercises](#custom-exercises). |
| `Tag` | `id: UUID`, `name` | |

### Workout structure

| Entity | Key attributes | Notes |
| --- | --- | --- |
| `Workout` | `id: UUID`, `name`, `info`, `icon`, `difficulty`, `isFavorite`, `isArchived`, `owner: String?` | `owner == nil` means a **catalogue workout**. An owner (the Firebase uid) means a **user workout**: editable, and backed up to Firestore. An **archived** workout is kept only for its History (see [Deleting and archiving](#deleting-and-archiving)). |
| `Circuit` | `id`, `order`, `setCount` (1–8), `restTime`, `restartTime`, `duration` (10–600 s), `note` (≤ 280) | A block of rounds repeated `setCount` times. `restTime` separates rounds, and `restartTime` comes before the next circuit. `note` is guidance, such as a coach's, shown on the circuit card. |
| `Round` | `id`, `order`, `repCount` (1–99), `duration` (10–600 s), `weightKg` (0–1000, precise), `weight` (whole kg, legacy), `note` (≤ 280) | One exercise within a circuit; the same exercise may appear more than once. `duration` falls back to the circuit's when it is 0 (`RunPlan.duration(of:in:)`). Use `weightInKg`, which writes both weights and reads `weight` when `weightKg` is 0 (stores from before version 4). For a timed exercise (`Exercise.isTimed`) `duration` is the target and reps don't apply. |

### What was done: History

| Entity | Key attributes | Notes |
| --- | --- | --- |
| `History` | `id`, `run`, `duration`, `caloriesBurned`, `averageHeartRate`, `peakHeartRate`, `createdAt`, `effort` (0–10, 0 = not rated), `note` (≤ 500) | One saved run. `run` numbers a workout's runs from 1 (`History.nextRun(of:)`). `effort` and `note` are the user's feedback (`setFeedback(effort:note:)`, `EffortRating`). Deleting a History cascades to its sets and rounds (`HistoryDeletion`). |
| `CircuitMetric` | `order`, `duration`, `caloriesBurned` | One set, meaning one pass through a circuit. Links back to its `Circuit`. |
| `RoundMetric` | `order`, `repCount`, `weight`, `duration`, `restTime` | One round as performed. Links back to its `Round`, and so to the `Exercise`. |

Metrics point back at the plan (`CircuitMetric.circuit`, `RoundMetric.round`). That link
is how History shows an exercise's name and image, and how personal records find the
exercise (`RoundMetric.round.exercise.id`). **Code that deletes workouts, circuits or rounds
must keep this in mind.** See [Deleting and archiving](#deleting-and-archiving).

### Planning

| Entity | Notes |
| --- | --- |
| `Schedule` | A workout on a `specificDate`, or every `dayOfWeek` when `repeatEvent` is set. Shown in *Today workouts* on Home and in `ScheduleView`, whose picker offers the user's, their coaches' and the catalogue's workouts. `Workout.schedules` cascades, so a deleted workout (by the user, or a coach taking one away) leaves the schedule with it. |
| `Plan`, `Session` | A training plan and its ordered sessions. Modelled, but with little UI so far. |

### Unused entities

`Achievement`, `Goal`, `Log`, `Message` and `Notification` are in the model but nothing
reads or writes them yet. They are kept so the store needs no migration when features
arrive. `Notification` is also the name of a Foundation type. Because of this entity,
code in the app module must write `Foundation.Notification` when it means the
`NotificationCenter` type.

## Deleting and archiving

History must keep resolving after the plan it recorded changes. The delete rules and the
code that deletes things work together:

| Relationship | Rule | So that |
| --- | --- | --- |
| `Workout.circuits`, `Circuit.rounds` | Cascade | A workout's plan goes with it |
| `History.circuitMetrics`, `CircuitMetric.roundMetrics` | Cascade | Deleting a History leaves no orphaned sets or rounds |
| `Round.metrics`, `Circuit.metrics`, `Workout.histories` | Nullify | Deleting plan objects never deletes History, but it does cut History's link to them, which is why the code below avoids deleting what History uses |

- **Workouts with History are archived, not deleted.** `Workout.archive()` sets
  `isArchived`, clears the favourite and deletes the workout's schedules. The editor's
  **Delete** does this for a user workout that has runs, and hard-deletes one that
  doesn't. Either way it removes the Firestore copy. `CatalogueSync` archives a dropped
  catalogue workout that has runs, and unarchives it if the catalogue brings it back.
- **Archived workouts are hidden everywhere a workout can be picked or run.** Lists and
  pickers use `Workout.listed`, `Workout.listedUserWorkouts`,
  `Workout.listedCatalogueWorkouts` and `Workout.listedCoachWorkouts`. Favourites combine `Workout.listed` with
  `isFavorite`. `WorkoutSnapshot.all` leaves them out, so the watch deletes its copies.
  History still shows them, and their runs still count towards personal records. **A new
  query that lists workouts must use one of these predicates.**
- **Circuits and rounds with History are detached, not deleted.** When the editor or
  `CatalogueSync` takes a circuit or round out of a plan, it deletes it only if no metric
  refers to it. Otherwise it clears the object's `workout` or `circuit`.
- **Personal records count saved runs only.** `PersonalRecords.historicalBest` reads round
  metrics whose `circuitMetric.history` exists. That ignores rounds of a run still in
  progress, and any left behind by older builds.

### Model versions

`RepCounter.xcdatamodeld` holds five versions. `RepCounter 5` is current (set in
`.xccurrentversion`, and as `currentVersion` in the project file).

- Version 2 added `Workout.isArchived` and made `CircuitMetric.roundMetrics` cascade.
- Version 3 added `Workout.assignedBy`: the coach's uid on a trainee's copy of a workout
  their coach made for them ([09](09-firebase.md#coaching-stage-4)).
- Version 4 added `Exercise.isTimed`, `Circuit.note`, `Round.note`, and precise
  `Round.weightKg`/`RoundMetric.weightKg` (Double kg, so pounds convert without rounding).
- Version 5 added `History.effort` and `History.note`: the user's feedback on a run.

The store migrates **lightweight** on first open: `NSPersistentContainer`'s default store
description migrates automatically and infers the mapping. `ModelMigrationTests` opens
stores written with versions 1 to 4 to prove it.

To change the model, **add a new version** (Editor → Add Model Version), make it current,
and change only that. Editing a version that has shipped makes existing stores
unreadable. Keep changes lightweight-compatible (add optional or defaulted attributes,
add entities, rename with a renaming identifier). Add a migration test like the existing
one.

## Entity code: three layers per entity

For `Workout`, for example:

1. `Models/CoreData/Workout+CoreDataClass.swift` and `+CoreDataProperties.swift` are the
   generated `NSManagedObject` subclass and its `@NSManaged` properties. Regenerate these
   rather than editing them by hand.
2. `Models/Workout.swift` holds the hand-written extension: `awakeFromInsert` (assigns
   `id` and `createdAt`), a `create(with:…)` factory with defaults, sorted getters
   (`getCircuits()`, `getRounds()`, all ordered by `Sorter` descriptors), display helpers
   (`getLength()`, `getDifficultyString()`) and `find(with:id:)` lookups.
3. `Workout.Model`, a `Codable` struct nested in the same file, is the JSON shape used by
   the bundled catalogue, Remote Config and Firestore. `Model.create(with:)` builds the
   entity, and `Model(with:)` reads one. `getJSONRepresentation()` encodes an entity to
   JSON.

Getters return sorted arrays rather than the raw `NSSet`. Always use `getCircuits()` and
`getRounds()` instead of `circuits`/`rounds` when order matters, which is almost always.

## Catalogue vs user data

The **catalogue** is the exercise library plus the workouts that ship with the app. It
comes from Firebase Remote Config, keys `target_list`, `muscle_group_list`,
`equipment_list`, `exercise_list` and `workout_list`, each holding JSON. The files in
`RepCounter/Models/Data/*.json` are the fallback.

**User data** is everything the user makes or records: their workouts (`owner != nil`),
favourites, History, schedules and plans.

### Loading the catalogue: `CatalogueSync`

`SplashView` calls `DataStore.load(with:provider:)` when Remote Config reports fresh
values or the store has no exercises. That calls `CatalogueSync.apply`, which **updates
in place, matched by id**:

- Targets, muscle groups, equipment and exercises are inserted or updated.
- Catalogue workouts are inserted or updated, circuits and rounds included. A workout's
  favourite flag belongs to the user, so it is kept. A newly inserted catalogue workout
  always starts unfavourited, whatever the JSON says.
- **User workouts are never touched.**
- Anything the new catalogue dropped is deleted **only if no History refers to it**.
  Otherwise a workout is archived, and a circuit or round is detached, so History still
  resolves. Exercises are pruned last, once the workouts that used them are gone.
- On a save error the context is rolled back rather than crashing.

Before this, the app wiped and reseeded the store, which deleted History and plans every
time Remote Config changed. The tests in `CatalogueSyncTests` pin the new behaviour.

**Reset data** (Settings) is the exception. `UserDataReset.eraseWorkoutData` deletes the
cloud backup, then `DataStore.resetCoreData` batch-deletes every entity and the catalogue
is reloaded ([10](10-settings-and-storage.md#settings-screen-phone)).

## Editing a workout: `WorkoutEditView`

The editor (`Views/Workout/Edit/`) is where user workouts are created and changed.

- `WorkoutEditView.ViewModel` makes a **main-queue scratch child** of the view context. A
  new workout is built in it, so cancelling throws it away. An existing workout is edited
  in its own context.
- The list is a flat array of `WorkoutBuilderItem`s: circuit headers
  (`CircuitItemView.ViewModel`), rounds (`RoundItemView.ViewModel`) and "+ Exercise"
  rows. Row view models hold **values only**, plus the id of the circuit or round they
  describe. Adding circuits or exercises inserts nothing into Core Data until **Save**.
- `WorkoutEditItemsView.ViewModel.save(on:context:)` applies the rows **in place**. A row
  whose id matches an existing circuit or round updates that object, so it keeps its
  identity and its History links. New rows create objects with the row's id. A circuit or
  round taken out of the plan is deleted, unless metrics refer to it: then it is only
  detached from the workout.
- Tapping a row opens its settings sheet: reps (not for a timed exercise), duration and
  weight in the chosen unit, plus a **note** (circuits have one too). Blank notes are saved
  as none (`String.nilIfBlank`). Swiping a row right, or its context menu, **duplicates**
  it below with the same settings, for the same exercise twice in a circuit.
- **Delete** archives a workout that has History and deletes one that doesn't
  ([above](#deleting-and-archiving)).
- After saving, the child's changes are pushed to the view context and saved. The
  workout is then mirrored to Firestore through `WorkoutCloudStore`
  ([09](09-firebase.md)). Deleting a workout also deletes its Firestore document.

## Custom exercises

Signed-in users can make exercises the catalogue doesn't have (`CustomExerciseEditView`,
from **+** on the exercise list or the workout editor's picker). They are ordinary
`Exercise` rows, so rounds, History, records and runs need nothing special:

- `createdBy` is the creator's uid; `nil` means a catalogue exercise
  (`Exercise.inCatalogue`). `CatalogueSync` never deletes a custom exercise.
- `isInLibrary` decides whether an exercise is listed. **A query that lists exercises to
  browse or pick must use `Exercise.inLibrary`** (the exercise list, the picker, and the
  workout generator do). A custom exercise that only came with someone else's workout,
  or that its creator removed from their list, isn't in the library.
- There is no "deleted" flag. `Exercise.pruneUnusedCustomExercises(in:)` deletes custom
  exercises out of the library that **no round uses**. Rounds with History are detached
  rather than deleted, so an exercise that was done stays with that History. Never delete
  an exercise that rounds use: `Exercise.rounds` cascades. The editor calls it after
  saving or deleting a workout, and `removeFromLibrary()` after taking one out of the list.
- Without an animation, `ExerciseArtworkView` shows the exercise's `symbol`. Each app
  has its own (`RepCounter/Views/Exercise/` and `RepCounter Watch App/Views/`); the watch
  gets the symbol in `WorkoutSnapshot.ExerciseSnapshot.symbol` ([05](05-watch-sync.md)).
- The creator's custom exercises are backed up to `users/{uid}/exercises`
  ([09](09-firebase.md)) and restored before workouts.
- **Sharing.** An `AssignedWorkout` carries the custom exercises its rounds use
  (`exercises`, built by `CoachingManager.assign` with `Exercise.customModels(usedIn:in:)`),
  whoever made them, so a workout can be passed on. `CoachWorkoutsImporter` adds them to the
  trainee's store **out of their library**, updates them when the coach changes them, never
  touches one in the trainee's library, and prunes them once the workout goes.
- **Save to My Exercises** (exercise details) calls `saveToLibrary(by:)`. One the person
  made goes back in their list. Anyone else's is **copied** with a new id and the person as
  `createdBy`, so the coach's later changes reach the workout's copy but not the saved one,
  and the copy can be passed on in the person's own workouts.

## Transfer models (not Core Data)

These `Codable`, `Sendable` value types in `Shared/WorkoutTransfer.swift` carry data
between the apps. They are covered in [05](05-watch-sync.md):

- `WorkoutSnapshot`: a workout as the watch needs it, with its circuits, rounds,
  exercises and each exercise's record.
- `WorkoutRunRecord`: a run the watch recorded, with sets and rounds.

## Querying from SwiftUI

Most lists use `@FetchRequest` directly in the view, with `NSPredicate` filters built by
the filter view models. See `ExerciseFilterView.ViewModel.getPredicate()`, which combines
target, muscle group, body segment, equipment and difficulty predicates with
`NSCompoundPredicate(andPredicateWithSubpredicates:)`. Keep predicates on attributes and
ids, for example `ANY muscleGroups.id IN %@`. Bridge Swift integers to `NSNumber` in
predicate arguments.

**Sorting.** The workout and exercise lists fetch by name, then arrange what they fetched
with the order chosen at the top of their filter sheet (`WorkoutFilterView.ViewModel.Sort`,
`ExerciseFilterView.ViewModel.Sort`). It's done in memory because the most useful orders
come from History, which a sort descriptor can't reach:

- Workouts: name (A–Z, Z–A), recently run, most run, newest first, easiest or hardest
  first. The workout picker (Schedule) uses the same sheet and orders.
- Exercises: name (A–Z, Z–A), recently done, most done (runs that included it), easiest or
  hardest first. Usage comes from `ExerciseFilterView.Usage.all(in:)`, worked out once
  per choice of order rather than on every keystroke of a search.

Ties, and never-run workouts or never-done exercises, go by name. As with History
(`HistoryFilterView`), the sort isn't counted as a filter, and **Clear** resets it to name.
