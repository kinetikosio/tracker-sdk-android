# Kinetikos Tracker SDK for Android

Records a person's walking in the background, uploads it to Kinetikos, and runs
the 6-minute walk test, inside your app. This repository is where the SDK is
published: a Maven repository under `maven/`, one folder per version. The
latest is **1.0.0-beta.1**.

## Add it to your app

In `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://raw.githubusercontent.com/kinetikosio/tracker-sdk-android/master/maven")
    }
}
```

In your app module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.kinetikoshealth:tracker-sdk:1.0.0-beta.1")
}
```

Your app must compile against Android 15 (`compileSdk = 35`) or later; it can
run on Android 12 (`minSdk = 31`) or later. The SDK's manifest, permissions and
shrinker rules merge into your app on their own.

A version with a suffix (`-beta.1`) is for testing; Gradle only uses it when
you ask for that exact version.

## Then

Configure it in your `Application.onCreate`, with a `tokenProvider` that returns
a participant token from your backend:

```kotlin
Tracker.configure(this, TrackerEnvironment.Production) { forceRefresh ->
    myBackend.participantToken(forceRefresh)
}
```

Kinetikos gives you your tenant, client credentials and the integration guide.
