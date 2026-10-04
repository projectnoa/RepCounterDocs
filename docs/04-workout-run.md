# 04 · The workout run

[← Data model](03-data-model.md) · [README](../README.md) · Next: [Phone ↔ watch sync →](05-watch-sync.md)

A **run** is one pass through a workout, card by card. Both apps show a run as a paged
carousel. Both lay out the same cards from the same plan. Both record what the user did in
the same shape: one History with a metric per set and a metric per round.

## Vocabulary

| Term | Meaning |
| --- | --- |
| **Circuit** | A block of rounds, repeated `setCount` times |
| **Set** | One pass through a circuit. Recorded as a `CircuitMetric`. |
| **Round** | One exercise within a circuit. Each performance of it is a `RoundMetric`. |
| **Card** | One page of the carousel |
| **Run** | One performance of the whole workout. Recorded as a `History`. |

## 1. The plan: `RunPlan`

`Shared/RunPlan.swift` turns a `Workout` into an ordered array of `PlannedCard`s:

```
[intro]                                       phone only
for each circuit c (numbered from 1):
  for set in 1...c.setCount:
    circuit(c, number, set)                   "Coming up: Circuit 2, Lap 1 of 3"
    for each round r in c:
      preview(r)       if showsExercisePreviews and set == 1
      exercise(r, number)
      rest(c.restTime)     between rounds          phone only
    rest(c.restartTime)    unless last circuit     phone only
summary                                       always last
```

`RunPlan.Options` controls the three variations: `showsExercisePreviews` (the *Show
exercise previews* setting), `includesIntro` and `includesRests` (true on the phone,
false on the watch). A circuit with `setCount == 0` contributes nothing.

Two invariants the carousels depend on:

1. **A card's index is its position in the array.** Cards are tagged with their index, and
   the carousel's selection is that index.
2. **The summary is always last**, at `count - 1`. Both carousels finish the run when the
   selection reaches `count - 1`, so no other card may ever have that index.
   `onlyTheSummaryFinishesTheWorkout` in the phone tests guards this.

Each app maps `PlannedCard`s to its own card view models in
`WorkoutCardCarouselView.ViewModel.loadCards` (one per app).

## 2. The cards

| Card | Phone view | Watch view | Timer (when *Time workout* is on) |
| --- | --- | --- | --- |
| intro | `WorkoutIntroCardView` | — | — |
| circuit | `CircuitPreviewCardView` | `CircuitPreviewCardView` | the circuit's `duration` |
| preview | `ExercisePreviewCardView` | `ExercisePreviewCardView` | the circuit's `duration` |
| exercise | `ExerciseCardView` (rep wheel or clock for a timed exercise, weight wheel, ring, note, record) | `ExerciseCardView` (rep wheel, ring, record) | the round's duration (`RunPlan.duration`) |
| rest | `RestCardView` | (not planned) | the rest length |
| summary | `WorkoutSummaryCardView` | `WorkoutSummaryCardView` | — |

The phone's exercise card also shows the listening indicator when *Listen for count* is
on ([07](07-voice-counting.md)). `RepCountCircleView` draws one arc per rep. Its scale is
the round's target reps (seconds for a timed exercise), on both phone and watch, so a full
ring means the planned set is done. Reps past it go round again, and reps past the record
(last time's set) turn yellow.

### Which card is "active" (phone)

A `TabView` with `.page` style keeps neighbouring pages alive. So a card can't assume it is
on screen just because it exists. Each phone card gets an `isActive` flag and reacts to
it changing:

- **Becoming active:** start the card's `RunnerState` (its timer), stamp `activatedAt`,
  and start listening if enabled.
- **Becoming inactive:** stop the timer, stop listening, and report the card's result
  through `onComplete`. That covers a round's reps and weight, or a circuit starting.

The carousel activates exercise cards in **two phases**
(`activeExerciseIndex = nil`, then the new index on the next run-loop turn). That way the
card being left always reports its result before the next one starts. Without it, paging
quickly could deliver the two changes in the wrong order.

The watch cards use `onAppear`/`onDisappear` to drive their `RunnerState`, and the watch
carousel reports enter/leave to `WatchRunRecorder`.

### The rest card (phone)

`RestCardView` recaps the run between exercises. `loadCards` builds each rest card's
view model from the plan with `WorkoutCardCarouselView.RestContext`:

- **Just done:** the exercise card before the rest and its set number. It shows the set
  (weight × reps, or time) against the best ever and against the same round and set of
  the workout's last run (`History.latest(of:)`), compared by `SetPerformance`.
- **Workout:** exercises done out of the run's total, and the volume lifted so far (reps
  when nothing was weighted) against the last run at the same point.
- A rest that comes right before a circuit card is a **set break** and reads *Set Complete*.

The next exercise is left to its preview card. The reps come live from the carousel's
`PersonalRecords` (`reps(forCard:)`, `totalReps(throughCard:)`), which the exercise cards
update as the wheel moves. A card whose wheel never moved counts as 0 reps.

`RestCardView` reads those values in its own `body`, where the environment object is
observed, and passes plain numbers to its panels. The pager lays rest cards out before
their exercise is done. Handed the `PersonalRecords` object itself, a panel's inputs never
change, so it keeps showing the reps from that first layout.

## 3. Timers and "Time workout"

`WorkoutConfig` holds the two per-run switches from the detail screen. One is created per
detail screen, and it is passed down with `.environmentObject`:

- `timeWorkout`: cards show a countdown (`TimerView`) and **advance on their own** when it
  runs out. Not on a **timed exercise**: it shows its own clock, counting up, and the
  workout's countdown would race it, so there it's hidden and doesn't run
  (`RepCountCircleView.showsWorkoutTimer`). The user moves on when they're done.
- `listenForCount` (phone only): see [07](07-voice-counting.md).

`TimerView` (`Views/Custom/RepCount/TimerView.swift`, compiled into both apps):

- Counts up to `countTo` seconds while `RunnerState.isRunning` is true, and pauses when it
  goes false.
- Measures elapsed time from **dates**, not by counting timer ticks. Ticks stop while the
  app is in the background, so a tick counter would fall behind. Here a timer that ran
  out while the app was backgrounded completes as soon as it resumes.
- Runs as a `.task(id: runnerState.isRunning)`. Changing `isRunning` cancels the task, and
  the time run so far is banked. It sleeps to the next whole second so the clock steps
  evenly.
- Calls `onComplete` **once**. A card you swipe back to after its time ran out does not
  auto-advance again.

The carousels wire every card's "time's up" callback to `nextCard(with: index)`. That
call only advances if `index == activeCard`, which protects against a preloaded
neighbouring page's timer firing. On the phone the circuit and exercise cards have
separate `onTimeUp` callbacks, because their `onComplete` reports results. The preview and
rest cards' `onComplete` means "time's up".

## 4. Recording a run on the phone

`WorkoutCardCarouselView.ViewModel` (`@MainActor`) records into Core Data as you go.
`historyDict` maps card index to the metric recorded for that card:

| Event | What is recorded |
| --- | --- |
| Carousel appears | `startedAt = .now`, so the run's duration is measured from here |
| A **circuit card is left** (`recordCircuit`) | The set begins. Creates its `CircuitMetric` once, and stores the energy total and time **at this moment** (`setStartEnergy`, `setStartTime`). Leaving it again restarts the reading without creating a second metric. |
| An **exercise card is left** (`recordRound`) | Creates the `RoundMetric` with the wheel's reps, the weight (0 for bodyweight rounds) and the time on the card. Revisiting a card **overwrites** reps and weight and **adds** to the duration. |
| The **summary is reached** | `finishWorkout()` waits one second so the last exercise card can report as it deactivates. Then it calls `finishRun`. |
| **End** is pressed (or the watch ends it) | `finishRun(…, endedEarly: true)` |

`finishRun` records **once**, whichever of those happens first. Then
`recordWorkout(activeEnergy:averageHeartRate:peakHeartRate:duration:endedAt:)`:

1. Creates the `History` (`run = History.nextRun(of: workout)`) and attaches the set
   metrics and their round metrics, in card order.
2. Splits the energy and time across sets with `SetSplits`
   (`Shared/WorkoutTransfer.swift`):
   - `perSet(startReadings:finalReading:)`: each set's calories run from its own start
     reading to the next set's start, or to the final total for the last set.
   - `durations(starts:end:)`: the same for time, in seconds.
3. Saves the context. Then it backs the History up to Firestore
   (`DataStore.saveHistoryToCloud`).

Measuring each set's own share matters because HealthKit only gives a running total.
Before `SetSplits`, every set recorded the workout's total at the moment it began.

### Ending early

- Rounds that were completed, meaning their card was left, are **kept**, including those
  of a set still in progress.
- The card on screen when the run ended is **not** kept, because it wasn't completed.
- If no round was completed at all, nothing is saved, and the metrics created along the
  way are deleted (`discardRecordedMetrics`), so no orphans are left behind.
- A run that already reached the summary was recorded then. Ending it afterwards
  (dismissing the summary) does nothing.

### Phone-only runs

The phone always asks HealthKit for authorisation and asks the watch app to start. With
no watch, there is no session to mirror. The run still works and records reps, weights,
times and durations, with zero calories and heart rate. The phone does **not** run a
HealthKit session of its own. Heart rate and calories are for people with an Apple Watch.

## 5. Recording a run on the watch

The watch carousel feeds a `WatchRunRecorder` (`RepCounter Watch App/Core/`) instead of
Core Data. The watch keeps no History:

- `enter(card:at:)` / `leave(card:at:activeEnergy:)` on every page change. Leaving a
  circuit card starts its set, just as on the phone. Leaving an exercise card adds the time
  spent on it.
- `setReps(_:forCard:)` whenever the rep wheel settles.
- `finish(at:activeEnergy:averageHeartRate:peakHeartRate:endedEarly:)` builds a
  `WorkoutRunRecord`, using the same `SetSplits` rules and the same early-end rules. It
  returns `nil` if already finished, if the workout has no id, or if the run ended before
  any round was done.

The carousel view model's `recordRun(with:sendingThrough:endedEarly:)` sends the record
to the phone (`SyncMessage.workoutRun`, guaranteed delivery), then stores the run's best
reps as records locally ([§6](#6-personal-records-last-record)). Reaching the summary
card calls it. So does the **End** control (`ControlView`), with `endedEarly: true`.

On the watch, an exercise card logs reps on a wheel and, for a weighted round, the weight
beside it (the Digital Crown turns the selected wheel), starting at the plan's weight. A
timed exercise shows a clock counting up instead of the reps, and the workout's countdown
is hidden there, as on the phone. The carousel keeps each card's `CardInput`, and
`WatchRunRecorder.log(_:forCard:)` records it: the precise weight in `RoundRecord.weightKg`
(whole kilograms in `weight`, for older phone apps). The summary card shows the run at a
glance (`RunSummarySnapshot`: time, reps, volume, energy, records beaten). A watch-only run
has no intro or rest cards; a phone-started run shows the phone's, see
[05](05-watch-sync.md#live-runs-the-phones-cards-on-the-watch).

While a run is on screen, the watch sets `ConnectivityManager.isRunInProgress`. A
catalogue update that arrives meanwhile is held in `pendingWorkouts` and applied when the
run ends. Applying it mid-run would rebuild the very circuits and rounds on screen.

## 6. Personal records ("Last record")

Progress is **strength**, not only reps. A set is a `SetPerformance` (`Shared/Weight.swift`):
reps, weight in kg, and for a timed exercise its time. Sets of an exercise compare by
`isBetter(than:)`: weighted sets by estimated one-rep max (Epley, weight × (1 + reps ÷ 30)),
so 25 kg × 6 beats 20 kg × 10; bodyweight sets by reps; timed ones by time. A weighted set
beats a bodyweight one, and on an equal estimate the heavier wins.

**On the phone** (`PersonalRecords(benchmarks:)`, from `PersonalRecords.benchmarks(in:before:excluding:)`),
each exercise has an `ExerciseBenchmark`: the best set of its **last session** (what the
card asks you to beat: *Last time: 20 kg × 8*), the best set ever, and the heaviest weight.
Exercise cards `register(_ performance:forExerciseID:card:)` as the wheel or weight moves,
and read:

- *New record!* when the set beats `best(forExerciseID:excludingCard:)`, the best ever
  including this run's other cards; *Better than last time!* when it beats last session;
  *Great job!* at the planned target.
- The weight to start on, once per card: `startingWeight(forExerciseID:beforeCard:)` (the
  latest earlier card of the exercise this run, then last session's), else the plan's.

The reps-only API below is still what the watch uses (it has no weight picker).

`Shared/PersonalRecords.swift`. The watch's record is **the most reps ever logged for an
exercise**, across every workout, keyed by exercise id.

- `PersonalRecords(historical:)` starts from earlier runs:
  - Phone: `PersonalRecords.historicalBest(in: context)` takes the maximum `repCount` of
    the `RoundMetric`s of saved runs (`circuitMetric.history != nil`), grouped by
    `round.exercise.id`. Runs of archived workouts still count.
  - History: `PersonalRecords.historicalBest(in:before:)` gives the records as they stood
    when a run started, leaving out that run and every later one. An old run is never
    judged against records set after it.
  - Watch: `PersonalRecords.stored()` reads `UserDefaults` key `personalRecords`. The
    phone keeps it filled by including each exercise's record in the catalogue it sends
    (`ExerciseSnapshot.record`). The watch also merges in its own runs' reps, so the next
    watch run shows them before the phone has replied.
- During a run, each exercise card calls `register(reps, forExerciseID:, card:)` as the
  wheel moves. `record(forExerciseID:excludingCard:)` combines the historical best with
  this run's other cards. So the second set of squats shows the first set's reps as the
  record to beat.
- **A card never counts its own reps**, or the record would climb as you turn the wheel.
- No record, or a record of 0, reads as "No record yet". `PersonalRecords.text(for:)` is
  the shared wording.

The carousel creates one `PersonalRecords` per run and injects it with
`.environmentObject(vm.records)`. It is `@MainActor` and publishes this run's entries so
the cards redraw when a sibling card changes the record.

After a phone run is saved, the new `RoundMetric`s trigger a catalogue republish to the
watch ([05](05-watch-sync.md)), so the watch's records update too.

## 7. The summary

- **Phone:** reaching `WorkoutSummaryCardView` records the run **first**
  (`finishRun`, which keeps it as `recordedHistory`), then shows `WorkoutSummaryView` as a
  full-screen cover for that `History`. The summary reuses `HistoryView.ViewModel` and
  shows:
  - the run's stat tiles;
  - **How Did It Go?** (`RunFeedbackEditor`): a 1–10 effort rating (session RPE, from
    *Very easy* to *All out*) and a note of up to 500 characters, saved with **Save
    Feedback** onto the `History` and backed up for the user's coaches
    ([09](09-firebase.md)). The same editor is on the run's History screen;
  - **Achievements**: new records (with the set they beat), exercises done better than
    last time, badges this run earned, the week streak, what to aim for when nothing
    improved, and the next badge;
  - the comparison with the last run;
  - the workout's last 8 runs as a chart;
  - this week against last week;
  - each exercise's best set against its record and last time.

  Dismissing it ends the run and pops back to the detail screen. The UI tests read its
  total reps (`AID.RunWorkout.summaryReps`) off the *Total Reps* tile.
- **Watch:** reaching the summary card records the run, sends it to the phone and stops
  the HealthKit session. HealthKit then finishes saving the workout **in the background**.
  The card's **Done** closes the run and returns to the workout's detail screen, with
  nothing to wait on.
- **Watch, ended early** with **End**: the rounds done so far are recorded, the run
  closes, and `SummaryView` shows HealthKit's `HKWorkout` totals (duration, energy, heart
  rate, water) once they arrive. It shows *Saving to Health…* until then, and its **Done**
  is always available, so the screen can never hold the user up.
- **Watch, ended from the phone:** if the session ends while the run is still on screen,
  the run records what was done, closes, and shows the same summary.

The History list (`HistoryListView`) is the activity log:

- **This Week**: this week so far beside last week (`ProgressSummary.weekTotals`).
- The **last 10 runs**, newest first.
- **View More** (shown past 10 runs) opens `AllWorkoutHistoryView`. It lists every run,
  grouped by month when sorted by date, and is searchable by workout name with an
  *All / Favorites* scope. `HistoryFilterView` filters by focus, difficulty and date range,
  and sorts by newest, oldest, longest or most energy.

Progress over all time lives on the profile (`UserProfileView`), built in one pass by
`ProgressSummary` (`Models/ProgressSummary.swift`, oldest run first):

- **All Time** tiles: workouts, the current week streak, volume, heaviest lift, reps,
  energy, time, records beaten and exercises logged. A week without a workout ends the streak only once
  that week is over.
- **Trends** (`TrendsCard`): one metric (volume by default, reps, energy, workouts or time) per week
  over 4 or 12 weeks, or per month over a year. Empty periods show as zero. The total is
  compared with the same length of time before it.
- **Badges** (`BadgesView`): tiers per `BadgeMetric` in `BadgeCatalog`, each earned by the
  run that crosses its threshold. Badges use SF Symbols as stand-ins
  (`BadgeDefinition.systemImage`, drawn in `BadgeView`) until the artwork exists.
- **Personal Records** (`PersonalRecordsListView`): each exercise's best set and heaviest
  weight, when it was set, and a chart of its best set per run (`ExerciseRecordTrendView`:
  estimated one-rep max for weighted exercises, reps or time otherwise). A run breaks a record
  only by beating one that already existed, as on the History screen.

- **Most Run Workouts**: the top three, by number of runs.

On the History list, the workout cards sit below **This Week**, so the UI tests scroll to
them (`openLatestHistory(for:)`).

**Deleting a run:** the trash button on a run's History screen, or long-pressing its card in
a list, asks first, then `HistoryDeletion.delete` removes the History (its sets and rounds
cascade), its backup and feedback, and republishes progress. Records, trends and badges
are worked out from the runs, so they drop it. The watch keeps the best reps it has seen,
so its record for an exercise doesn't go down.

Each run (`HistoryView`) opens with stat tiles (volume — reps × weight over weighted
rounds, in the chosen unit — total reps, time, energy and heart rate). **Personal Records**
(`HistoryRecords.swift`) sets each exercise's best set against how it stood before the run
(`benchmarks(in:before:excluding:)`, [§6](#6-personal-records-last-record)): *New record*,
*Stronger* (better than last time), *Same as last time*, *Below last time*, or *First
record* for an exercise never logged before. Only a beaten record counts
as a record: it earns the trophy on the round that set it and beside the exercise in the
reps chart, so a first run isn't all trophies. Next, the run is compared with the previous
run of the same workout (`StatDelta`, by `createdAt`). Swift Charts in
`HistoryCharts.swift` rank reps per exercise, plot reps per set against the run's average
(touch a bar for its time and energy) and show each muscle group's share of reps (a round's
reps are split evenly across its exercise's muscle groups; groups past the top five fold into
"Other"). The **Sets** list then shows every performed set with its time, calories and each
round's reps and weight. The UI tests read `AID.History.totalReps` ("Total Reps: n"),
`AID.History.circuit` and `AID.History.round` there.
