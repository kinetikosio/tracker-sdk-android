# Using the SDK

[← Kinetikos Tracker SDK for Android](../README.md)

Everything `Tracker` does, once the [quick start](../README.md) is in place. Everything is on `Tracker` and its types, in `com.kinetikoshealth.tracKer.sdk`. The `suspend` calls run from any coroutine.

- [Environments](#environments)
- [What the SDK adds to your app](#what-the-sdk-adds-to-your-app)
- [The token provider](#the-token-provider)
- [Consent](#consent)
- [Permissions and tracking](#permissions-and-tracking)
- [Recording a walk and the 6-minute walk test](#recording-a-walk-and-the-6-minute-walk-test)
- [Status and events](#status-and-events)
- [Uploads and mobile data](#uploads-and-mobile-data)
- [Recorded time and diagnostics](#recorded-time-and-diagnostics)
- [Sign-out](#sign-out)
- [What the SDK does on its own](#what-the-sdk-does-on-its-own)

## Environments

`Tracker.configure(context, environment, tokenProvider)` takes `TrackerEnvironment.Production`, `TrackerEnvironment.Staging`, or `TrackerEnvironment.Custom(url)` for any other Kinetikos backend you are given, such as development (`https://tracker-api-dev.kinetikos.io`). Use staging or development while testing, with the credentials Kinetikos gives you for that environment ([which backend](backend.md#which-backend)). The same SDK version works with every environment.

Call `configure` in `Application.onCreate`, before anything else: Android starts your app's process there for the SDK's background work too (after a reboot, for an alarm or a walking transition), with no activity.

## What the SDK adds to your app

These merge into your manifest when you add the dependency:

| What | Why |
|---|---|
| `ACTIVITY_RECOGNITION` | The "Physical activity" permission: Android tells the SDK when walking starts and stops. Required. |
| `POST_NOTIFICATIONS` | Android 13 and later: shows the recording notice. Optional: tracking works without it, invisibly. |
| A foreground service (`health`, or `dataSync` where `health` isn't available) with `FOREGROUND_SERVICE*` | Keeps recording while the screen is off. It shows a notice with **your app's icon**; tapping it opens your app. |
| `SCHEDULE_EXACT_ALARM`, `RECEIVE_BOOT_COMPLETED`, `WAKE_LOCK` | Keeps background tracking armed over long idle periods and after a reboot. |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | Lets your app offer the battery-optimisation exemption, which helps on strict phones. The SDK never asks for it by itself. |
| `INTERNET`, `ACCESS_NETWORK_STATE` | Uploads, and knowing when they can go. |
| A transparent activity | Shows the system permission prompts for `requestPermissions()`. It's the SDK's only screen. |

The recording notice reads "Tracker", with "Tracking (idle)", "Recording walk" or "6-minute walk test" below it. See [Testing and going live](going-live.md#google-play) for what the Play Console asks about these permissions.

## The token provider

The SDK calls your token provider whenever it needs a token, including **when Android starts your app in the background, with no UI**. It must:

- return a participant token from your server ([Your backend](backend.md#3-give-your-app-participant-tokens)), using the session your app keeps, e.g. a refresh token in encrypted storage;
- when `forceRefresh` is `true`, fetch a **new** token instead of returning a cached one: the SDK passes it after Kinetikos refused the last token (it expired), and then retries the request once;
- **never show UI** or wait for the user to sign in;
- return `null` when nobody is signed in to your app;
- throw for a temporary failure (offline, timeout). The SDK keeps recording and tries again later.

```kotlin
object ParticipantTokens {
    suspend fun fetch(forceRefresh: Boolean): String? {
        val session = MyAuth.storedSession() ?: return null                 // nobody signed in
        if (!forceRefresh) TokenCache.valid()?.let { return it }            // still good for a while
        val token = MyBackend.participantToken(session)                     // your server's token endpoint
        TokenCache.store(token)
        return token
    }
}
```

The SDK reads which participant a token is for. **If a token is for a different person than before, the SDK first deletes everything the previous person left on the phone**, so data can never go up under the wrong person.

## Consent

Nothing is recorded until the person gives consent in your app. Show the SDK's consent text, with a checkbox for the declaration, and record the agreement:

```kotlin
val document = Tracker.consentDocument(locale = Locale.getDefault().language)
// Show document.text (paragraphs separated by a blank line),
// and a checkbox labelled document.declaration. Enable "I agree" once it's ticked.

// On "I agree":
val recorded = Tracker.giveConsent(document.version, document.locale)
// A failure means no participant yet: your token provider returned null. Make sure the user is signed in.
```

- The text is available in English and Portuguese; any other language gets English.
- Consent works offline and is sent to Kinetikos as soon as possible. Uploads wait until Kinetikos has received it.
- `Tracker.withdrawConsent()` records a withdrawal: recording stops at once. The person can consent again later. Offer it somewhere in your settings.

A ready-made Consent screen: [Screens you can copy](screens.md#4-consent).

## Permissions and tracking

After consent, ask for the permissions, then start tracking:

```kotlin
val permissions = Tracker.requestPermissions()   // shows the system prompts that are needed
Tracker.startTracking()
```

- The SDK **never prompts on its own**; you decide when. Call `requestPermissions()` while your app is on screen. `Tracker.permissions()` tells you what's granted: `ACTIVITY_RECOGNITION` is required, `NOTIFICATIONS` is optional. If the person denied a permission for good, send them to your app's page in Android Settings (`Settings.ACTION_APPLICATION_DETAILS_SETTINGS`).
- `startTracking()` is remembered across launches and reboots. It's safe, and recommended, to call it on every launch once the person is signed in.
- `stopTracking()` pauses background recording without withdrawing consent.
- Turning off "Physical activity" after granting it is a **withdrawal** of consent ([what happens](#what-the-sdk-does-on-its-own)), not a pause: say so near your permission settings, and ask for consent again when `state` is `WITHDRAWN`. Asking for the permission *before* recording consent, as Kinetikos' app does, keeps a consent off Kinetikos' ledger for someone who then refuses it.

## Recording a walk and the 6-minute walk test

```kotlin
Tracker.startWalkTest()          // false if something is already recording
Tracker.stopWalkTest()           // stopping early discards the test

Tracker.startManualRecording()   // the person stops it themselves
Tracker.stopManualRecording()    // saved and uploaded
```

The walk test runs for `Tracker.walkTestDurationSeconds` (6 minutes) and fails if the person stops moving for `Tracker.walkTestIdleTimeoutSeconds` (45 seconds). Only a completed test is uploaded.

The SDK makes no sound unless you ask: set `Tracker.walkTestCompletionFeedback = true` in `Application.onCreate` to play a short tone, and vibrate, when a walk test completes, so the person knows they can stop with the phone in a pocket. The vibration needs `android.permission.VIBRATE` in your manifest; without it only the tone plays.

## Status and events

```kotlin
// Follow the status, e.g. in a ViewModel or a LaunchedEffect:
Tracker.status.collect { status ->
    when (status.state) {
        TrackerStatus.State.NEEDS_CONSENT -> showConsent()
        TrackerStatus.State.NEEDS_PERMISSIONS -> showPermissionHelp()
        TrackerStatus.State.REVOKED -> showNoLongerTracked()
        TrackerStatus.State.STUDY_INACTIVE -> showStudyNotActive()
        TrackerStatus.State.WITHDRAWN -> showConsentWithdrawn()
        TrackerStatus.State.ACTIVE, TrackerStatus.State.STOPPED,
        TrackerStatus.State.SIGNED_OUT, TrackerStatus.State.NOT_CONFIGURED -> Unit
    }
    countdown = status.walkTest.remainingSeconds   // while status.walkTest.running
}

// One-off events:
Tracker.events.collect { event ->
    if (event is TrackerEvent.WalkTestFinished) showResult(event.outcome)
}
```

- `state` reports the first thing in the way, in this order: `NOT_CONFIGURED`, `SIGNED_OUT`, `REVOKED`, `STUDY_INACTIVE`, `WITHDRAWN`, `NEEDS_CONSENT`, `NEEDS_PERMISSIONS`, `STOPPED`, `ACTIVE`.
- `recording` says what is recording now: `PASSIVE` (background), `MANUAL`, `WALK_TEST`. `sessionStartedAt` is when the manual recording or walk test began, for a count-up timer.
- While a walk test runs, a new status comes every second, so a countdown can show `walkTest.remainingSeconds` as it is.
- `sync` tells you how many recordings are waiting (`pendingCount`, `uploadingCount`), when the last upload happened (`lastSyncedAt`), the last error, and `waitingFor` what: `NETWORK`, `WIFI` (on mobile data, which isn't allowed), `LOW_DATA_MODE` (Data Saver), `SERVER_CONSENT` or `WITHDRAWAL_CHECK`.
- `backgroundWork` is `OFF` when the person restricted your app's background use (Settings › Apps › your app › Battery), and `RESTRICTED` under Battery Saver or when Android put the app in its most restricted standby bucket. It never blocks `ACTIVE`, but background walks are missed or late: tell the person where to change it.

The events are `WalkTestFinished(outcome)` (`COMPLETED`, `STOPPED_EARLY`, `STOPPED_MOVING`, `INTERRUPTED`), `Revoked`, `StudyInactive`, `ConsentWithdrawn` and `Error(message)`.

## Uploads and mobile data

The SDK uploads by itself, on Wi-Fi by default, whether your app is open or not. `Tracker.setAllowCellular(true)` lets it use mobile data; `Tracker.allowCellular` reads the current setting, for your own switch. `Tracker.syncNow()` uploads now, e.g. from a "Sync" button, and asks Kinetikos for the participant's Consent first.

## Recorded time and diagnostics

- `Tracker.recordedTime(days = 7)` is how much walking this phone recorded on each of the last days, today last, uploaded or not: `passiveSeconds` (background) and `activeSeconds` (recorded walks and completed walk tests). It moves as soon as a walk is recorded. What Kinetikos makes of it comes from Kinetikos itself: see [the Kinetikos API](screens.md#9-the-kinetikos-api).
- `Tracker.diagnostics()` describes what the SDK is doing on this phone, for support. It holds no tokens, no external id and no sensor data; its `toString()` is a plain-text report for a support ticket.
- `Tracker.sdkVersion` is this SDK's version.

## Sign-out

Call it when the person signs out of **your** app, **before** your own sign-out, so a last upload can still get a token:

```kotlin
Tracker.signOut()
MyAuth.signOut()
```

The SDK stops, spends up to 5 seconds uploading what's pending, then stops sending anything. What couldn't be sent stays on the phone:
- **The same person signs in again:** it uploads.
- **Someone else signs in:** it's deleted before their first token is used.
- **Nobody signs back in:** it's deleted after 90 days.

Call `startTracking()` again after the next sign-in.

## What the SDK does on its own

| Situation | What happens |
|---|---|
| Revoked by your server | Recording stops; what wasn't sent is deleted; `state` is `REVOKED`, and the `Revoked` event fires. A new invitation brings the person back: they keep their history but must give consent again (`NEEDS_CONSENT`). |
| Institution suspended, or the study ended | Recording and uploads stop; what was recorded before stays on the phone. `state` is `STUDY_INACTIVE`, and the `StudyInactive` event fires. The SDK checks every 15 minutes or so (less often as it lasts), and when the institution is active again everything resumes by itself, with no new sign-in. |
| Consent withdrawn (in your app, or by Kinetikos staff) | Recording stops; anything recorded after the withdrawal is deleted unsent. `state` is `WITHDRAWN`. The SDK checks with Kinetikos every 6 hours or so, so a withdrawal recorded by staff stops tracking without your app being opened. |
| "Physical activity" turned off after it was granted with consent | That is a withdrawal, recorded and sent to Kinetikos like `withdrawConsent()`, the next time the SDK runs. `state` is `WITHDRAWN` and the `ConsentWithdrawn` event fires. Turning the permission back on doesn't restore consent: show your Consent screen again. |
| Offline | Recordings are kept and uploaded later. |
| Token provider returns `null` or throws | Recording continues if it was on; uploads wait. |
| A different person's token | The previous person's data on the phone is deleted first. |
| App killed, or the phone restarted | Background tracking resumes by itself, and an interrupted recording is closed and kept. A walk test that was running ends as `INTERRUPTED`. |
| Background use restricted, or Battery Saver on | Walks are still recorded while your app is open, and uploaded then; `backgroundWork` says so. |
| Recordings older than 90 days | Deleted from the phone, uploaded or not. |
