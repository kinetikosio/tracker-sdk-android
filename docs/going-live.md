# Testing and going live

[← Kinetikos Tracker SDK for Android](../README.md)

## Testing

- **Use a real phone.** An emulator has no walking to detect.
- Point the SDK at staging or development, with a test tenant and that environment's credentials from Kinetikos.
- Background work isn't immediate: Android decides when the SDK's background jobs run. Open the app to trigger an upload, or call `syncNow()`.
- Walk to test passive recording, with the phone in a pocket: Android reports walking after a minute or so, and the recording starts then.
- Try the phones your users have. Some manufacturers stop background apps more eagerly than Android itself; `status.backgroundWork` and `Tracker.diagnostics()` show what the SDK is allowed to do there, and the battery-optimisation exemption helps.

## Google Play

The SDK's permissions merge into your app, so the Play Console asks about them as if they were your own:

- **Foreground service types.** The recording service is a `health` foreground service (and `dataSync` where `health` isn't available). In **App content → Foreground service permissions**, declare them, and explain that the app records the person's walking with the motion sensors, in the background, for a health study or programme, with a short video of the notice while tracking.
- **Physical activity.** `ACTIVITY_RECOGNITION` is a runtime permission the person grants; say in your store listing and in the app why walking is recorded.
- **Exact alarms.** `SCHEDULE_EXACT_ALARM` keeps background tracking armed over long idle periods. Where Android doesn't grant it, the SDK falls back to inexact alarms by itself.
- **Battery optimisation.** `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` only lets your app *offer* the exemption; the SDK never asks for it. If you don't offer it and would rather not declare it, remove it from your manifest:

  ```xml
  <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" tools:node="remove" />
  ```

## Privacy

For the Play Console's **Data safety** form: the SDK collects the person's **physical activity** (motion-sensor recordings of their walking) and the phone's model and Android version, linked to the participant, and sends them to Kinetikos over an encrypted connection. Nothing is used for advertising or shared with anyone else. The person can withdraw consent in your app, and data on the phone is deleted when they're revoked, when someone else signs in, or after 90 days.

## Before you release

- [ ] Server: client credentials kept server-side; invite; participant-token endpoint for your app; revoke ([Your backend](backend.md)).
- [ ] Dependency added, with Kinetikos' Maven repository.
- [ ] `Tracker.configure` first thing in `Application.onCreate`, with `TrackerEnvironment.Production` and your production credentials on your server.
- [ ] The token provider works in the background without UI; returns `null` when signed out.
- [ ] Consent screen with `consentDocument` and `giveConsent`; withdrawal in settings.
- [ ] `requestPermissions`, then `startTracking` on every launch while signed in.
- [ ] `signOut()` before your own sign-out.
- [ ] Play Console: foreground service types declared, Data safety filled in.
- [ ] Tested on real phones against staging or development, including with the app closed.
