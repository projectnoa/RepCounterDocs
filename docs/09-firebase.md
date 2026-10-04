# 09 · Firebase and cloud services

[← Concurrency](08-concurrency.md) · [README](../README.md) · Next: [Settings and storage →](10-settings-and-storage.md)

Firebase is phone-only. The watch never talks to it. Setup happens in `RepCounterApp.init`
(`FirebaseApp.configure()`), which reads `RepCounter/GoogleService-Info.plist`.

| Service | Used for | Code |
| --- | --- | --- |
| **Auth** | Signing in with Apple, Google, email and password, or anonymously. The user id stamps user workouts (`Workout.owner`). | `AuthManager`, `AppleSignInManager`, `GoogleSignInManager` |
| **Remote Config** | Delivering the exercise and workout catalogue | `RemoteConfigProvider` |
| **Firestore** | Backing up and restoring a signed-in user's workouts, runs and favourites | `CloudSync`, `CloudRestore`, `FirestoreBackupStore` |
| **Analytics** | Linked, with default collection | — |
| **App Check** | Proving requests come from the genuine app on a real device | `AppCheckConfiguration` |

## Auth

`AuthManager` (`@MainActor`, `ObservableObject`) publishes the Firebase `user` and an
`authState`:

| `AuthState` | Meaning |
| --- | --- |
| `.signedOut` | No Firebase user |
| `.authenticated` | Anonymous Firebase user |
| `.signedIn` | Signed in with a provider (Apple or Google) |

- A Firebase auth state listener, added in `init`, keeps `user` and `authState` current.
  On launch it also checks that the Apple ID or Google credential is still valid
  (`verifySignInProvider`).
- **Sign in with Apple:** `AppleSignInManager` runs an `ASAuthorizationController` request
  with a random nonce. The request carries the nonce's SHA-256 (CryptoKit), and
  `AuthManager.appleAuth(_:nonce:)` exchanges the credential with Firebase.
- **Google:** `GoogleSignInManager.signInWithGoogle()` presents Google's flow, then
  `AuthManager.googleAuth(_:)` exchanges the tokens.
- **Email and password** (`AuthManager+Email.swift`, `EmailAuthView`):
  - `signUp(email:password:displayName:)` links the email to an anonymous account, keeping
    its uid, or creates a new account, then sends a verification email.
  - `signIn(email:password:)`, `sendPasswordReset(email:)`, `resendEmailVerification()`,
    and `reloadUser()` to pick up a confirmed address.
  - `isEmailVerified` is `true` for Apple and Google accounts. The profile shows a banner
    until an email account confirms its address. Connecting with people will require it.
  - `EmailAuthMessage.text(for:)` turns Firebase's error codes into advice. A wrong
    password and an unknown email read the same, as email enumeration protection makes
    Firebase report them.
- **Anonymous:** `signInAnonymously()`.
- If a user is already signed in (for example anonymously), a new credential is **linked**
  to that account instead of replacing it (`authLink`). "Already in use" errors fall back to
  a normal sign-in.
- `deleteUserAccount(password:)` re-authenticates and revokes the Apple or Google token as
  required. For an email account that signed in over a minute ago it needs the password,
  and without it throws `AuthErrors.ReauthenticateEmail`, so the profile asks for it and
  tries again. It then deletes the backup, the public profile, and finally the Firebase
  user. The profile screen then erases the device (`UserDataReset.eraseDevice`) and signs
  in anonymously, so nothing is left for a later sign-in to upload into a new account.
- `userID` is what the app stamps on user workouts. Under XCUITest it is a fixed
  `"uitest-user"` and Firebase Auth isn't touched at all.

The first-launch login cover and the profile screen live in `Views/Auth/`
(`LoginView`, `WelcomeBackView`, `UserProfileView`).

## Remote Config: the catalogue

`RemoteConfigProvider` (`@MainActor`):

- Keys, as `RemoteConfigValueKey`: `target_list`, `muscle_group_list`, `equipment_list`,
  `exercise_list` and `workout_list`. Each is a JSON array matching the Codable `Model` of
  the entity ([03](03-data-model.md#entity-code-three-layers-per-entity)). An exercise
  held for a time (planks, wall sits) has `"timed": true`; keep `exercise_list` in step
  with the bundled `exercises.json`, which flags them. Rounds take an optional precise
  `weightKg` and a `note`, and circuits a `note`.
- Fetch timeout 5 s. Minimum fetch interval **0 in Debug** and **12 hours in Release**
  (`isDebug`, set by `#if DEBUG`).
- `fetchCloudValues()` is called at launch, except in previews and tests. It fetches,
  activates, and publishes `freshDataAvailable`. On failure, including offline, it still
  publishes `false`, so the splash screen never waits forever.
- `setupListener()` subscribes to real-time config updates and republishes when they
  arrive.
- `objects(forKey:)` decodes a key's JSON. When a key is empty or can't be decoded, the
  callers (`DataStore.load`) fall back to the bundled JSON in `Models/Data/`.

`SplashView` reacts to `freshDataAvailable` by running `CatalogueSync` (see
[03](03-data-model.md#loading-the-catalogue-cataloguesync)).

**To publish a catalogue change:** edit the JSON in the Remote Config console (or the
bundled files for the offline fallback), keeping ids stable. Ids are how `CatalogueSync`
updates entities in place. Changing an id makes it a new entity, and the old one is deleted
if nothing refers to it.

## App Check

`AppCheckConfiguration.install()` runs in `RepCounterApp.init`, **before**
`FirebaseApp.configure()`. From then on, calls to the Firebase services that support App
Check, such as Firestore, Storage and Authentication, carry an App Check token. The
console's **App Check → APIs** tab lists which services can be enforced. Once
enforcement is on for a service, it rejects requests without a valid token. That includes scripts reusing the API key from
`GoogleService-Info.plist`, modified builds, and emulators.

`RepCounterAppCheckProviderFactory` picks the provider:

| Build | Provider |
| --- | --- |
| Release, on a device that supports App Attest | `AppAttestProvider` (`DCAppAttestService`, key in the Secure Enclave) |
| Release, on a device without App Attest | `DeviceCheckProvider` |
| Debug, or the simulator | `AppCheckDebugProvider`, compiled out of Release |

- The app target links `FirebaseAppCheck` and has the
  `com.apple.developer.devicecheck.appattest-environment` entitlement set to `production`.
  TestFlight and App Store builds always use production, whatever the entitlement says.
- **Debug tokens:** on first launch, a Debug build logs
  `Firebase App Check debug token: <uuid>`. Add it in the Firebase console under
  **App Check → Apps → RepCounter (iOS) → ⋮ → Manage debug tokens**. For CI, or to keep
  one token per machine, set the `FIRAAppCheckDebugToken` environment variable in the
  scheme to a token registered there. Debug tokens are secrets: never commit them.
- The watch doesn't talk to Firebase, so it needs no App Check.

**Rolling out enforcement:** register both providers in the console, then leave the
services **unenforced** while App Check metrics show the share of verified requests. Turn
on enforcement per service once nearly all traffic is verified. Versions of the app from
before App Check send no token, so enforcing locks them out; wait for users to update.

## Public profile: `profiles/{uid}`

What other users see about someone: `PublicProfile` holds `displayName` (1–30
characters), `avatarID` (`avatar-01` … `avatar-20`) and `updatedAt`, and nothing else.
There is no email, no photo and no full name. The default name is only the first name from
the account.

- `ProfileStore.shared` keeps a copy in `UserDefaults`, so the avatar shows while signed
  out or offline. The copy remembers which account it belongs to (`publicProfile.owner`),
  so signing in to another account never publishes the previous one's name.
- `sync()` runs after `CloudSync.sync` in `RootView`. The newer of the device's copy and the
  published profile wins (`updatedAt`), and an account without one gets one published.
- `EditProfileView`, opened from the profile header, picks the name and avatar.
- **Avatars** (`Models/Avatar.swift`): `AvatarCatalog.all` is the 20 predefined avatars.
  `AvatarView` draws the image asset named after the id (`avatar-07`, …) when the asset
  catalogue has one, and an SF Symbol placeholder until then. Add artwork by adding image
  sets with those names; no code changes. Never renumber the ids: profiles store them and
  the security rules check them.
- The rules let any signed-in user read a profile, and only its owner write it, with just
  these three fields, validated.

## Connections: following other users

Someone shows their invite. The person who scans or types it asks to follow them. They
approve, as a **friend** (sees progress and achievements, can message) or a **coach**
(also manages workouts they assign). Stages 3–5 add those abilities. This stage is the
connection itself.

| Path | Holds |
| --- | --- |
| `invites/{code}` | `Invite`: `ownerID`, `createdAt`, `expiresAt` (7 days) |
| `connections/{followerID}_{followeeID}` | `Connection`: `status` (`pending`/`approved`), `role` (`friend`/`coach`), `inviteCode`, dates |
| `users/{uid}/nicknames/{otherID}` | `nickname`, which only the user sees |

- **Invite codes** (`InviteCode`): 8 characters from an alphabet without look-alikes (no
  0/O, 1/I/L, 5/S). They're shown as `RC-7K2Q-9MXA`, and the QR code holds
  `repcounter://connect/7K2Q9MXA`. `normalized(_:)` reads any of these forms. A code never
  reveals the owner's uid. There is one code at a time, renewed when less than a day is
  left, and **New Code** withdraws the old one.
- **`ConnectionsManager.shared`** (`@MainActor`): `start()` runs from `RootView` after the
  profile sync. It listens to both sides of the user's connections and publishes
  `requests`, `following` and `followers` as `Person`s: connection, profile and nickname.
  `lookUp(code:)` shows who a code belongs to before `requestToFollow(_:)`. Then there's
  `approve(_:as:)`, `setRole(_:for:)`, `remove(_:)` (decline, remove, unfollow or cancel)
  and `setNickname(_:for:)`.
- **Who can connect** (`availability`): Apple, Google, or a confirmed email. Anonymous and
  unconfirmed accounts see why they can't. `reloadUser()` refreshes the ID token after
  confirming, because the rules read `email_verified` from it.
- **Screens** (`Views/Connections/`):
  - Profile → **People** (`ConnectionsView`), with **My Code** (`InviteCodeView`, QR code
    and share) and **Add Person** (`AddConnectionView`). That uses VisionKit's
    `DataScannerViewController`, or typing where scanning isn't available.
  - `PersonDetailView` for the nickname, the role and removing.
  - Home shows `ConnectionsInboxCard` while requests wait.
- **Links:** the `repcounter` URL scheme is registered, so a QR code scanned with the Camera
  app, or a shared link, opens the app. `RootView.onOpenURL` sets
  `pendingInviteCode`, and Home asks to confirm it. Other URLs go to
  `GIDSignIn.sharedInstance.handle`.
- **Rules:** a request can only be created with a valid, unexpired invite owned by the
  person to follow, under the deterministic id, as `pending`. Only the person followed can
  approve or change the role. Either person can delete it. Invites can be read by code but
  only listed by their owner.
- **Account deletion** removes the user's connections, invites and nicknames
  (`deleteAllData(for:)`), before the profile and the Firebase user.

### Shared progress (stage 3)

People the user follows, once they've approved them as a friend or coach, share their
progress. **PersonDetailView → View Progress** opens `SharedProgressView`: the same
dashboard as the profile (`ProgressOverview`, `TrendsCard`, `BadgeStrip`,
`RecentRecordsCard`, `MostRunWorkoutsView`), plus their 10 most recent workouts
(`SharedRunDetailView`).

- **One read, not hundreds:** each user publishes their progress as one document,
  `users/{uid}/shared/progress` (`ProgressSnapshot`). `CloudSync.publishProgress(in:)`
  writes it after every saved run (`save(_ history:)`, which watch runs go through too) and
  at the end of each sync. `SharedProgressLoader` reads that one document, then a single
  run (`fetchRun(id:for:)`) when the user opens one of the recent workouts.
- **Feedback, for coaches:** when the viewer is the person's coach
  (`connection.role == .coach`), the loader also reads their latest 50 `RunFeedback`s,
  newest run first (`readsFeedback`). `SharedProgressView` shows *Their Feedback* at the
  top, an effort badge on each recent workout, and the note on the run's detail. For a
  friend it isn't read, and the rules would refuse it.
- **The snapshot** holds totals, streak weeks, records, badge progress and most-run
  workouts, all counted over every run, plus the latest 1,500 per-run rows (for trends and
  recent workouts) and each record's latest 30 chart points. `timesBroken` and
  `mostRunWorkouts` are stored rather than derived from those rows, so trimming never
  changes them. `ProgressSummary(snapshot:now:)` rebuilds the dashboard and works out the
  **current** streak for when it's read. That keeps it far below Firestore's 1 MB limit.
- **Fallback:** someone who hasn't published a summary yet (an older app version), or one
  of an older `ProgressSnapshot.currentVersion` (version 2 made records strength-based),
  has their runs read instead.
- **The same numbers they see:** `ProgressSummary` is built from `RunInput`s, which come
  from a `History` on the device (`init(history:)`) or a shared `CloudRun`
  (`init(run:exerciseNames:)`). Exercise names come from the local catalogue, by the
  `exerciseID` each backed-up round stores. Sets and rounds are ordered by `createdAt`,
  then position (`CloudRun.isOrdered`).
- **Rule:** `runs` and `shared/progress` can be read by the owner, or by someone whose connection
  `{follower}_{owner}` is `approved` (`isApprovedFollower`). That's one document read, by
  the deterministic id. Nothing else under `users/{uid}` is shared: not their workouts,
  favourites or nicknames. A denied read shows as "Not shared with you"
  (`SharedProgressLoader.State.notShared`).
- Removing a follower, or them unfollowing, ends access at once, since the rule checks the
  connection on every read.

### Coaching (stage 4)

A follower the user approved as a **coach** makes workouts for them. The coach can create,
edit and delete only the workouts they made. The trainee runs them, on the phone and the
watch, and can duplicate one to make it their own, or remove it.

| Path | Holds |
| --- | --- |
| `users/{traineeID}/assigned/{workoutID}` | `AssignedWorkout`: `coachID`, `workout` (`Workout.Model`), `createdAt`, `updatedAt`, and `exercises`: the custom exercises (`Exercise.CustomModel`) the workout uses, which the trainee may not have |

- **Coach** (`CoachWorkoutsSection`, on the trainee's `PersonDetailView`):
  - lists `CoachingManager.workouts(assignedTo:)`;
  - **New Workout** and tapping one open the regular `WorkoutEditView`, with
    `persistsLocally: false` and `AssignedWorkoutCloud` as its cloud, so **Save** sends it
    to the trainee and nothing lands in the coach's own workouts;
  - editing starts from `CoachingManager.draft(of:)`, a copy in a scratch context that's
    never saved;
  - **Assign One of My Workouts** sends a copy with new ids
    (`Workout.Model.duplicated(owner:)`);
  - swiping deletes.
- **Trainee** (`CoachingManager.start()` from `RootView`):
  - a listener applies the `assigned` collection with `CoachWorkoutsImporter`. Each
    workout becomes a `Workout` with `owner` and `assignedBy` set to the coach, keeping
    circuit and round ids;
  - one no longer there is removed, or archived if it has runs (`CatalogueSync.remove`);
  - `refresh()` runs once before `CloudSync.sync`, so runs of assigned workouts restore
    after a reinstall.
- **Where assigned workouts show:**
  - **From Your Coach** in the Workouts tab (`Workout.listedCoachWorkouts`);
  - on the watch (they're `Workout.listed`);
  - on Home while new or changed (`CoachingManager.unseen` and `CoachWorkoutsInboxRows`,
    cleared when the workout is opened).
- **Never the trainee's own:** `Workout.isUserWorkout()` is false for them, so they can't
  be edited, aren't in My Workouts, and aren't backed up to the trainee's `workouts`.
  `CatalogueSync` leaves them alone, since they have an owner. `CoachWorkoutBanner` on
  the detail screen shows who they're from, with **Duplicate to My Workouts** and
  **Remove**.
- **Rules:** creating or updating needs `isApprovedCoach(traineeID)`, meaning the
  connection `{coach}_{trainee}` is approved **with the coach role**, `coachID` equal to
  the writer, and the document id equal to the workout's. The trainee reads all and can
  delete any. A coach reads and deletes only their own. When a coach is changed to a
  friend or removed, they can no longer change the workouts; the trainee keeps them.
- Account deletion empties `assigned` with the rest of `users/{uid}`.

### Messages and safety (stage 5)

> **Messaging is switched off: a future feature.** `MessagingFeature.isEnabled` is
> `false`, so the app hides the Messages button, "Message" on a person, the unread rows
> on Home, and doesn't listen for threads. `messagingEnabled()` in `firestore.rules`
> returns `false`, so no one can start a thread or send a message even outside the app;
> reading and deleting (account deletion) still work. Blocking and reporting people stay.
> Without push notifications chat isn't useful, and run feedback and workout notes cover
> what a coach and trainee need. To bring it back, see `MessagingFeature` in
> `Core/Messaging.swift`: switch both flags on, deploy the rules, add remote notifications
> (APNs through Firebase Cloud Messaging) and restore the copy that mentions messaging.
> The rest of this section describes it as built.

People joined by an **approved connection, in either direction**, message each other. One
thread per pair:

| Path | Holds |
| --- | --- |
| `threads/{uidA}_{uidB}` (sorted) | `MessageThread`: `members`, the last message's preview (120 characters), sender and time, and `readAt` per member |
| `threads/{id}/messages/{id}` | `ChatMessage`: `senderID`, `text` (1–1,000 characters), `sentAt` (the server's time) |
| `users/{uid}/blocked/{otherID}` | Someone the user blocked; only they can read it |
| `reports/{id}` | `Report`: who, whom, why, and the message if it's about one; no user can read it |

- **`MessagesManager.shared`** (started from `RootView`):
  - listens to the user's threads (`members` `array-contains` uid);
  - `send(_:to:)` validates the text, checks `ConnectionsManager.canMessage`, masks it with
    `MessageFilter`, then writes the message and the thread's preview in one batch;
  - `markRead(_:)` sets only the user's `readAt`;
  - `unreadThreads` feeds Home (`UnreadMessagesInboxRows`) and the People screen's
    Messages button.
- **Screens:**
  - `MessagesView`: threads, and **New Message** to anyone connected;
  - `ChatView`: the conversation, 100 latest messages; long-press a message to report it,
    and the ⋯ menu has Report and Block;
  - **Message**, **Report** and **Block** on `PersonDetailView`;
  - a **Blocked** section on the People screen to unblock.
- **Safety, as App Review requires of user content (guideline 1.2):**
  - **Community Guidelines** (`CommunityGuidelinesView`) are agreed to once, before the
    first message or follow request.
  - **Filtering:** `MessageFilter` masks objectionable words, whole words only, on
    sending and again on showing.
  - **Reporting:** `ConnectionsManager.report` writes to `reports`, which you review in
    the console, within 24 hours as the guidelines promise.
  - **Blocking:** `ConnectionsManager.block` records the block, ends every connection with
    that person both ways, hides their thread, and refuses their invite code.
- **Rules:**
  - Threads and messages can be read by their two members only.
  - A message can be written only while `canMessage(a, b)` holds: an approved connection
    either way, and no block either way, checked with `exists` first so a missing
    document is simply false.
  - `sentAt` and `lastMessageAt` must equal `request.time`.
  - A message's rule reads its thread with `getAfter`, since the first message creates the
    thread in the same batch.
  - Follow requests are refused from people the followee blocked.
  - Reports can only be created.
- **The chat only listens once the thread exists.** A listener the rules refuse stops for
  good, and there is nothing to read before the first message.
- **Account deletion** removes every thread the user is in, with its messages, and their
  blocked list, before the connections, the profile and the Firebase user.

## Firestore: the user's backup

A signed-in user's data is backed up so it comes back after reinstalling the app and
signing in again. Everything lives under the user's own document:

| Path | Holds | Written when |
| --- | --- | --- |
| `users/{uid}` | `CloudFavorites`: favourite workout and exercise ids | A favourite is toggled, and on each sync |
| `users/{uid}/workouts/{workoutID}` | `Workout.Model` of a **user** workout, with `archived` and `createdAt` | Created, edited, favourited or deleted in the editor, and on each sync |
| `users/{uid}/exercises/{exerciseID}` | `Exercise.CustomModel` of a **custom exercise** the user made, with `inLibrary` | Created, edited or removed from the library in the exercise form, and on each sync. Deleted when the device deletes it (`CloudSync.deleteExercises`). Restored before workouts, whose rounds refer to it |
| `users/{uid}/runs/{historyID}` | `CloudRun` of a recorded run | A run is saved on the phone, a watch run is imported, or a sync finds it missing. Deleted with the run (`CloudSync.deleteRun`) |
| `users/{uid}/feedback/{historyID}` | `RunFeedback`: `runID`, `workoutName`, `runDate`, `effort` (1–10), `note` (≤ 500), `updatedAt` | Feedback is saved or cleared, and on each sync (restored onto runs without any, uploaded when missing). Only the user and their approved **coaches** can read it; friends can read runs but not this, which is why it isn't part of `CloudRun` |

Progress, personal records, badges, streaks and trends are all worked out from the runs
(`ProgressSummary`, `PersonalRecords`), so restoring the runs restores them too. Nothing
derived is stored.

- **`CloudRun`** (`Core/CloudBackup.swift`) is compact. It keeps the run's, sets' and
  rounds' ids and `createdAt` dates, which order the sets and date the run, and refers to
  the workout, circuits and rounds **by id**. It doesn't embed the workout, as the old
  `History.Model` did, twice over.
- Workout documents are written **whole** (`setData(from:)` without `merge`). A merge would
  keep fields the edit cleared, because the encoder omits `nil` rather than writing null.
- A deleted user workout with runs is archived on the device, and its backup is kept with
  `archived: true`, so its runs can be restored. One without runs has its backup deleted.
- Catalogue workouts aren't stored, since Remote Config is their source. Their favourites
  go in `CloudFavorites`.
- Firestore's offline cache queues writes and sends them once the device is online.

### Who is backed up

Only accounts signed in with **Apple or Google** (`CloudSync.signedInAccountID`). An
anonymous account can't be recovered after a reinstall. Also, **Sign out** signs back in
anonymously, which must not copy the previous user's data into a new account. Linking
Apple or Google to an anonymous account keeps its uid, so the first sync afterwards
uploads what was recorded while anonymous.

### Syncing: `CloudSync`

`CloudSync.shared` (`Core/CloudSync.swift`, `@MainActor`) is the only way in:

- `save(_ workout:)`, `deleteWorkout(id:)`, `save(_ history:)` and `saveFavorites(in:)`
  write as things happen. `FirestoreWorkoutStore`, the `WorkoutCloudStore` seam the editor
  and workout cards use, forwards to it.
- `sync(context:)` runs from `RootView`'s `.task(id:)` once the main view shows, which is
  after the catalogue has loaded, and again whenever the signed-in account changes. It:
  1. fetches the user's workouts, runs and favourites, plus any legacy `workouts` the user
     owns;
  2. restores what the device is missing (`CloudRestore.apply`);
  3. uploads every user workout, every run the backup is missing, and the favourites.
- `status` (`unavailable`, `syncing`, `synced(Date)`, `failed`) drives the backup row on
  the profile, which also has **Sync Now**.
- `deleteAllData(for:)` empties `users/{uid}`'s subcollections and deletes the document.
  `AuthManager.deleteUserAccount` calls it after re-authenticating and **before**
  `user.delete()`, while the rules still let the user delete. If it fails, the account
  isn't deleted.

### Restoring: `CloudRestore`

`CloudRestore.apply` (`Core/CloudRestore.swift`) only ever **adds**, matching by id.
Anything already on the device is kept as it is, so it is safe to run on every launch.

- User workouts are rebuilt with `CatalogueSync.syncCircuits`, keeping circuit and round
  ids, and re-archived if they were archived.
- Runs come next. A run whose workout can't be found is **skipped**, for example a
  catalogue workout the catalogue has since dropped. It stays in the backup for a later
  restore. So is a set whose circuit, or a round whose round, can't be found. It never
  traps, unlike the models' `create` methods.
- Favourites are marked, never unmarked.

Catalogue circuit and round ids come from the catalogue JSON, so they are the same on
every install. That is what lets restored runs of catalogue workouts link back up.

### Security rules

`firestore.rules`, at the repository root, lets each user read and write only
`users/{their uid}/…`. The legacy top-level `workouts` collection is read-only, and only
for documents the user owns. The legacy `histories` collection is closed. Deploy the rules
with the Firebase console or `firebase deploy --only firestore:rules`.

### Legacy collections

Before this, user workouts went to a top-level `workouts` collection (with `owner`) and
runs to `histories`. A sync reads the user's own legacy workouts once and moves them under
`users/{uid}`. Legacy `histories` documents have no owner and no dates, so they can't be
restored. Devices that still have the app upload their runs on the first sync instead.

### Tests

Nothing is written while `UITestConfiguration.skipsCloudSync` is true, that is under
XCUITest or when hosting unit tests (`CloudSync(isEnabled:)`). `CloudSyncTests` drives
`CloudSync` and `CloudRestore` against an in-memory `CloudBackupStore`.
`WorkoutCloudSpy` in `WorkoutTests.swift` checks what the editor and workout cards send
through `WorkoutCloudStore`.
