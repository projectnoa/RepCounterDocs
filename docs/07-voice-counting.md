# 07 · Voice rep counting

[← HealthKit](06-healthkit.md) · [README](../README.md) · Next: [Concurrency →](08-concurrency.md)

With **Listen for count** on, the phone listens while an exercise card is active and moves
the rep wheel as you count out loud: "one, two, three…".

## Where it is available

- **iPhone only.** watchOS has no speech recognition API: the watchOS SDK has no `Speech`
  framework, only `SoundAnalysis`, which classifies sounds but doesn't transcribe words.
  An earlier design streamed raw microphone audio from the watch to the phone over
  WatchConnectivity for transcription. It never worked reliably and has been removed.
  Voice counting on the watch would need a different approach, such as on-watch sound
  classification.
- The **Listen for count** toggle on a workout's detail screen appears only when
  **Settings → Ask to listen for count** (`showSpeechRecognitionOption`) is on. It is on by
  default.

## Permissions

Listening needs **speech recognition** and **microphone** permission. The usage strings are
the `INFOPLIST_KEY_NSSpeechRecognitionUsageDescription` and
`INFOPLIST_KEY_NSMicrophoneUsageDescription` build settings.

Permission is asked for **when the toggle is switched on**, not when the first exercise
starts, where the system prompt would interrupt the run:

```swift
Toggle("Listen for count", isOn: $workoutConfig.listenForCount)
    .onChange(of: workoutConfig.listenForCount) { _, isOn in
        guard isOn else { return }
        Task {
            if await SpeechRecognizer.requestPermissions() == false {
                workoutConfig.listenForCount = false   // switch back off
                isListeningDenied = true               // explain, and point to Settings
            }
        }
    }
```

## The pieces

```
ExerciseCardView (isActive)
   └── TranscriptionEngine (@MainActor, one per run)      transcript → rep count
          └── SpeechRecognizer (actor)                    microphone → transcript
                 ├── AVAudioEngine input tap → SFSpeechAudioBufferRecognitionRequest
                 └── SFSpeechRecognizer.recognitionTask (partial results)
```

### `SpeechRecognizer` (`RepCounter/Core/SpeechRecognizer.swift`)

An **actor** that owns the audio engine, the recognition request and the task. That way
their state can't be touched from two places at once.

- Recognises `en-US` speech, with `shouldReportPartialResults = true` and
  `taskHint = .dictation`.
- `requiresOnDeviceRecognition` is set whenever the recogniser supports on-device
  recognition. Counting then works offline and no audio leaves the phone.
- `prepareEngine(onDevice:)` sets the audio session to `.playAndRecord` / `.measurement`
  with `.duckOthers`. It installs a tap on the input node that appends every buffer to the
  request, then starts the engine.
- `startTranscribing(onUpdate:)`, `resetTranscript()` and `stopTranscribing()` are
  `@MainActor` entry points that hop into the actor. They run **in the order they were
  called** (`enqueue`, each waiting on the one before): separate tasks aren't guaranteed
  to, and a card's stop landing after the next card's start switched listening off on the
  new card. `reset()` ends the audio, cancels the task, stops the engine and removes the
  tap.
- The recognition result handler is `nonisolated`, because Speech calls it on its own
  queue. It reports each transcript through the `@Sendable` callback and stops the
  engine on a final result or an error.

### `TranscriptionEngine` (`RepCounter/Core/TranscriptionEngine.swift`)

`@MainActor`. It turns an ever-growing transcript into a count:

- `mapArray` lists the spellings of each number from 1 to 100: digits, words, and hyphenated
  and spaced forms ("twenty-one", "twenty one", "twentyone", "21").
- `findNext(to:on:)` looks for **the next one or two numbers** after the current count in
  the transcript. So a missed word ("…three, five") still moves on, but a stray larger
  number elsewhere in the sentence doesn't jump the count. The count never goes down.
- `start(onUpdate:)` creates the `SpeechRecognizer` lazily, resets the transcript and
  starts transcribing. `stop()` stops it and resets the count to 0.
- Each start begins a new **session**, and `stop()` ends it; results from an ended session
  are dropped. Ending the audio makes the recognizer hand over one last final result
  *after* it's stopped. Counted again from 0 it came out as 1, and set the card just left
  to "1 rep", which the rest card then showed.

`WorkoutCardCarouselView.ViewModel` creates one engine per run and hands it to every
exercise card.

### On the exercise card

`ExerciseCardView` starts the engine when it becomes active and stops it when it becomes
inactive ([04](04-workout-run.md#which-card-is-active-phone)). Each count the engine
reports sets `currentCount`, which moves the rep wheel, but only while the card is active:
a count arriving after it was left is ignored. From there it's the same as turning
the wheel by hand: the record updates and the value is recorded when the card is left. The
card shows a red **Listening** label while enabled.

## Testing

`TranscriptionEngineTests` covers the number table: every rep from 1 to 100 is spelled
out and recognisable from its digits, spellings are unique and never blank, and 0 is
unmapped. It also covers stopping before starting and the `String.contains([String]?)`
helper. The microphone path has no automated tests. Check it by hand on a device: it needs
a real microphone and permission prompts.
