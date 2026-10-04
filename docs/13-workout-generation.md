# 13 · Workout generation

[← Conventions and gotchas](12-conventions-and-gotchas.md) · [README](../README.md)

The workout editor can draft a workout's plan from what's in its info section: the
title, focus, difficulty and description. It uses Apple's on-device language model
through the **FoundationModels** framework. Nothing leaves the device, there's no account
or network needed, and nothing is saved until the person taps Save, like any other edit.

| | |
| --- | --- |
| Code | `RepCounter/Core/WorkoutGenerator.swift` |
| UI | `WorkoutEditInfoView` (the button), `WorkoutEditView` (running it), `WorkoutEditItemsView.ViewModel.replacePlan(with:)` (showing the result) |
| Platform | iPhone only, iOS 26 or later, with Apple Intelligence turned on and its model downloaded. The watch doesn't compile it. |

## What the person sees

Under the **Description** field there's a **Generate Workout** button.

- It's **hidden** where the model can't be used: `WorkoutEditView.generateAction` is `nil`
  unless `WorkoutGeneration.isAvailable`. That checks `#available(iOS 26.0, *)` and
  `SystemLanguageModel.default.availability == .available`. It's hidden on iOS 17 and 18,
  on devices without Apple Intelligence, while the model is still downloading, and on a
  simulator whose Mac doesn't have Apple Intelligence on.
- It's **disabled** until there's something to work from:
  `WorkoutEditInfoView.ViewModel.canGenerate` needs a title, at least one focus and a
  description (difficulty always has a value).
- Tapping it puts the keyboard away, shows "Generating…" with a spinner, and disables the
  button until the model answers.
- If the plan already has exercises, a **"Replace the plan?"** alert asks first. The
  generated plan replaces every circuit and round.
- The result fills the editor's circuits and rounds, where the person can change anything
  before saving. Generating again gives a different workout (see
  [Choosing the exercises](#choosing-the-exercises)).
- A failure shows **"Couldn't generate a workout"** with the error's description.

It works in every editor that shows the info section, including a coach's editor for a
trainee's workout ([09](09-firebase.md)).

## How it works

```
WorkoutEditInfoView.ViewModel.generationRequest   (title, targets, difficulty, description)
        │
        ▼
WorkoutGenerator.generate(_:in:)
        ├─ candidates(for:in:)      the app's exercises the model may choose from
        ├─ circuitCount(for:)       2, 3 or 4 circuits, from the difficulty
        ├─ LanguageModelSession.respond(to:generating: GeneratedWorkout.self)
        └─ plan(from:candidates:circuitCount:)   checked, mapped back to Exercise objects
        │
        ▼
WorkoutPlanDraft  ──▶  WorkoutEditItemsView.ViewModel.replacePlan(with:)
```

`WorkoutGenerator` is `@MainActor` and reads Core Data on the context of the request's
targets (the editor's context), so the `Exercise` objects in the draft can go straight
into the editor's rows.

### Choosing the exercises

The model never makes up exercises. It's given a **numbered list** of the app's own and
answers with numbers, which `plan(from:…)` maps back to `Exercise` objects.

`candidates(for:in:)` builds that list:

1. Every exercise in the person's library (`Exercise.inLibrary`), so their own custom
   exercises are included and ones they removed aren't ([03](03-data-model.md)).
2. Only those whose target is one of the chosen focuses.
3. Only those no harder than the chosen difficulty. If fewer than 6 remain, it falls back
   to every difficulty, so an easy workout for a narrow focus still has enough to choose
   from.
4. Shuffled, then the first `candidateLimit` (40). The on-device model has a small
   context window, so it sees a sample instead of the whole catalogue. The shuffle also
   means asking again gives a different workout.

No exercises for the focus throws `GenerationError.noExercises`.

### Shaping the workout

The **number of circuits is chosen by the app, not the model**: `circuitCount(for:)`
gives 2 for easy, 3 for moderate and 4 for hard, and the prompt asks for that many. Left
to itself, the model tended to put every exercise in one long circuit.

The instructions (`instructions`) tell the model to act as a personal trainer:

- give each circuit its own theme (a warm-up, one muscle group, a finisher);
- put two to four exercises in each, without repeating one in another circuit;
- refer to exercises only by their number;
- use more sets and reps and shorter rest for harder workouts;
- choose seconds instead of reps for exercises marked *timed*.

The prompt (`prompt(for:candidates:circuitCount:)`) carries the title, focus, difficulty,
description, circuit count and the numbered list. Each exercise is marked *timed* or not,
and with its equipment or *bodyweight*.

**What the person typed only goes in the prompt, never in the instructions.** The model
gives instructions more weight than the prompt, so this keeps a description from changing
how it's told to behave.

### What the model generates

The answer is generated straight into Swift types with `@Generable`, so there's no JSON
to parse. `@Guide` constrains each value, so the model can't produce one out of range:

| Type | Field | Guide |
| --- | --- | --- |
| `GeneratedWorkout` | `circuits` | 2 to 4 |
| `GeneratedCircuit` | `theme` | A short theme. Generated first so the model settles what the circuit is for before choosing its exercises. Not kept. |
| | `sets` | 1 to 5 |
| | `restSeconds` | 10 to 120 |
| | `exercises` | 2 to 4 |
| `GeneratedRound` | `exerciseNumber` | The exercise's number in the list |
| | `reps` | 4 to 30 |
| | `seconds` | 15 to 120 |

Properties are generated in the order they're declared, which is why `theme` comes
before `exercises`.

### Reading the answer

`plan(from:candidates:circuitCount:)` doesn't trust the answer:

- A number that isn't on the list is dropped.
- An exercise repeated within a circuit is dropped.
- A circuit left with no exercises is dropped.
- If everything still ends up in **one circuit of more than four exercises**,
  `split(_:into:)` divides it as evenly as possible into `circuitCount` circuits (no more
  circuits than there are pairs of exercises), each keeping the original sets and rest.

An empty plan throws `GenerationError.emptyPlan` ("Try describing it differently").

### Into the editor

`replacePlan(with:)` replaces the editor's rows with a header, the rounds and an **add**
row for each circuit:

- Sets and rest come from the circuit.
- Reps come from the round.
- A timed exercise gets the round's seconds; any other exercise gets the editor's usual
  60 seconds.
- A weighted exercise starts at the editor's default weight (15 kg, or 35 lb), and any
  other at 0.

Saving goes through the editor's normal validation and save, so a generated workout is
an ordinary workout from then on.

## Testing it

- **Unit tests:** the steps that don't need the model are `static` so tests can call them
  directly: `candidates(for:in:)`, `circuitCount(for:)`, `plan(from:candidates:circuitCount:)`
  and `split(_:into:)`. `plan(from:…)` takes a `GeneratedWorkout` you can build by hand.
  None are covered yet.
- **The model itself** can't run in tests reliably, and its answers change from run to
  run. Try it on a device with Apple Intelligence, or a simulator on a Mac that has it on.
- **UI tests** never see the button: it's hidden wherever the model isn't available, and
  the UI test simulators aren't expected to have it.

## Gotchas

- **Keep the list short.** The on-device model's context window is small. More exercises,
  or longer details per exercise, can fail with a context-size error.
  `candidateLimit` keeps the prompt within it.
- **Check availability every time.** The model can be unavailable even on a supported
  device (Apple Intelligence off, model still downloading, low battery).
  `WorkoutGeneration.isAvailable` is read each time the editor draws.
- **Keep everything behind `@available(iOS 26.0, *)`.** The app supports iOS 17, so every
  FoundationModels type, the `@Generable` types included, is marked, and `generatePlan()`
  checks `#available` before it uses them.
- **Change the guides, not the reading, to change what's generated.** If a guide's range
  changes, check `replacePlan(with:)` and the editor's validation still accept the values.
