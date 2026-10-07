# Bus Sathi

Bus Sathi is an Android app for bus and passenger-vehicle drivers in Jammu and Srinagar. A driver registers once, taps **Start Trip** when the journey begins and **End Trip** when it finishes. While the trip is active the app records the route with GPS, and at the end it uploads the route to Firebase. The collected routes are used for research on bus routes and public transport planning.

| | |
|---|---|
| Application ID | `com.karroh.bussathi` |
| Current version | 1.78 (versionCode 24) |
| Minimum Android | 7.0 (API 24) |
| Target / compile SDK | 36 |
| Language | Kotlin |
| Backend | Firebase Authentication + Cloud Firestore |

## Contents

- [Features](#features)
- [How the app flows](#how-the-app-flows)
- [Privacy and research consent](#privacy-and-research-consent)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Firestore data model](#firestore-data-model)
- [Setup](#setup)
- [Building and signing a release](#building-and-signing-a-release)
- [Publishing to Google Play](#publishing-to-google-play)
- [What is not in this repository](#what-is-not-in-this-repository)
- [Troubleshooting](#troubleshooting)
- [More documentation](#more-documentation)
- [License](#license)

## Features

### Driver registration and login
- Email and password accounts through Firebase Authentication.
- Registration collects name, email, phone number, vehicle/bus number, license number, city (Jammu or Srinagar) and vehicle capacity (5 to 13, 17 to 21 or 32 to 52 seater).
- The profile is saved to Firestore under `drivers/{uid}`.
- The home screen greets the driver by name and shows their bus number.

### Live GPS trip tracking
- Location is requested every 5 seconds at high accuracy using the Fused Location Provider.
- Tracking runs in a foreground service (`LocationTrackingService`) with an ongoing notification, so it keeps recording when the app is in the background or the screen is off.
- The home screen and the notification show the number of points recorded and the distance covered, live.
- Distance is calculated with the Haversine formula.
- A spike filter drops bad GPS readings: any point that would need a speed above 30 m/s (about 108 km/h) from the previous point, or that reports accuracy worse than 100 m, is thrown away. This matters on mountain roads where the signal bounces.
- An active trip survives the app being closed. The trip ID and start time are stored on the device, and the app reconnects to the trip on the next launch.
- If GPS is switched off during a trip, the app shows a blocking "Trip Paused" dialog and a high-priority notification that opens Location settings.

### Trip upload to Firebase
- Ending a trip uploads one document to the `trips` collection with the driver, start and end times, every route point and the total distance.
- Trip IDs are readable: `<VEHICLE_NUMBER>_<yyyyMMdd_HHmmss>`.
- Trips with fewer than 50 recorded points or zero distance are discarded on the device and are not uploaded. This keeps accidental taps and stationary tests out of the data set.
- After a successful upload the trip distance is added to the driver's `totalKm`.

### Leaderboard
- Shows the top 50 drivers ranked by total kilometres driven.
- The list is cached on the device and refreshed from Firestore at most once every 24 hours, which keeps Firestore reads low.

### Research consent popup
- Shown on first launch. The driver must accept before using the app. See [Privacy and research consent](#privacy-and-research-consent).

### Other behaviour
- **Daily reminder**: a WorkManager job posts a notification at 8:00 AM each day, rotating through a set of Hindi and Urdu messages that remind the driver to start their trip.
- **Forced updates**: the app uses Google Play In-App Updates in *immediate* mode, so drivers always run the latest version.
- **Anti-spoofing check**: the main screen is blocked while Developer Options or USB debugging are enabled on the phone, to stop mock locations. See the note on this in [Setup](#setup).
- **Intro animation**: a Lottie animation on launch and an animated bus while a trip is running.

## How the app flows

```
IntroAnimationActivity  (Lottie intro, 2.5 s)
        |
SplashActivity          (logo, 2 s)
        |
   consent accepted? -- no --> ConsentDialog --> Accept (saved) or Exit App
        | yes
   logged in? ---------- no --> LoginActivity <--> RegisterActivity
        | yes
MainActivity            (Start Trip / End Trip, live stats, Leaderboard, Logout)
        |
LocationTrackingService (foreground GPS recording while a trip is active)
```

## Privacy and research consent

Bus Sathi is a research app, so consent to research use of the data is mandatory.

**When it appears.** `SplashActivity` checks consent before it checks login. Until consent is granted, the popup is shown on every launch, including for drivers who were already logged in before the popup was added.

**What the driver sees.** A short summary of what is recorded and why, an expandable "Read full privacy details" section, and **Accept** and **Decline** buttons. The popup cannot be dismissed by tapping outside it or pressing Back.

**Accept.** The choice and a timestamp are saved on the device (SharedPreferences file `consent_prefs`) and the app continues. The popup is not shown again.

**Decline.** A second dialog explains that consent is required and offers **Review again** or **Exit App**. A decline is never saved, so the popup comes back on the next launch. There is no way into the app without accepting.

**What the popup tells the driver is collected.**
- Account details entered at registration: name, email, phone number, vehicle number, license number, city and vehicle capacity.
- Trip data while a trip is active: GPS points (latitude, longitude, time and accuracy), trip start and end time, and total distance.

**When location is recorded.** Only between Start Trip and End Trip. The app does not request background location permission; recording outside the app works through the visible foreground service.

**Things to know.**
- The consent record lives only on the device. It is not written to Firestore, so clearing app data or reinstalling shows the popup again, and there is no server-side record of who consented and when.
- The consent text lives in `app/src/main/res/values/strings.xml` (the `consent_*` strings). If you change what the app collects, update that text and your Google Play Data safety form together.

## Tech stack

| Area | What is used |
|---|---|
| Language | Kotlin 1.9.20, Java 17 bytecode |
| Build | Gradle 8.13 (wrapper), Android Gradle Plugin 8.13.2, Kotlin DSL |
| UI | XML layouts with ViewBinding, Material Components, ConstraintLayout, Lottie 6.4.0 |
| Auth and database | Firebase BoM 32.7.0: Firebase Authentication, Cloud Firestore |
| Location | Google Play Services Location 21.1.0 |
| Async | Kotlin Coroutines and Flow 1.7.3 |
| Background work | Foreground service for tracking, WorkManager 2.9.0 for the daily reminder |
| Updates | Google Play In-App Updates 2.1.0 |
| Other | Gson 2.10.1 (leaderboard cache), reCAPTCHA 18.4.0 (pinned for a Play Console security warning) |

The code is organised as activities that call repository classes directly. There are no ViewModels.

## Project structure

```
BusTrackerApp/
├── app/
│   ├── build.gradle.kts            # App module: SDK levels, version, R8, dependencies
│   ├── proguard-rules.pro          # R8 keep rules (Firestore/Gson models, line numbers)
│   └── src/main/
│       ├── AndroidManifest.xml     # Permissions, activities, foreground service
│       ├── java/com/bustracker/    # Kotlin sources (see below)
│       └── res/                    # Layouts, strings, drawables, Lottie JSON, icons
├── tools/
│   ├── export_trip_csv.py          # Admin script: export a trip's route points to CSV
│   └── EXPORT_TRIP_README.md
├── build.gradle.kts                # Plugin versions
├── settings.gradle.kts
├── gradle.properties               # JVM heap for R8, AndroidX flags
├── local.properties.template       # Copy to local.properties and set sdk.dir
├── SETUP.md                        # Step-by-step Firebase setup
├── TESTING_GPS_ON_EMULATOR.md      # Simulating routes on an emulator
└── VIEWING_FIREBASE_DATA.md        # Reading trips in the Firebase console
```

Kotlin sources, under `app/src/main/java/com/bustracker/`:

```
data/model/
    Driver.kt                  Driver profile stored in drivers/{uid}
    DriverModel.kt             Leaderboard row (name, licenseNumber, totalKm)
    LocationPoint.kt           One GPS point
    Trip.kt                    A full trip with its route points
    TripStatus.kt              ACTIVE / COMPLETED
data/repository/
    AuthRepository.kt          Sign in, registration, driver profile, sign out
    LocationRepository.kt      GPS updates, spike filter, in-memory route points
    TripRepository.kt          Firestore writes and reads for trips
service/
    LocationTrackingService.kt Foreground service, notification, GPS-off alert
ui/splash/
    IntroAnimationActivity.kt  Launcher activity, Lottie intro
    SplashActivity.kt          Consent gate, then login check
ui/consent/
    ConsentDialog.kt           The mandatory consent popup
ui/auth/
    LoginActivity.kt
    RegisterActivity.kt
ui/main/
    MainActivity.kt            Trip controls, permissions, updates, security checks
    LeaderboardActivity.kt     Top 50 drivers, 24-hour cache
    LeaderboardAdapter.kt
    DailyReminderWorker.kt     8 AM reminder notification
util/
    ConsentManager.kt          Stores the consent choice
    TripStateManager.kt        Persists the active trip across restarts
    DistanceCalculator.kt      Haversine distance
    DateUtils.kt               Trip ID and display date formats
    PermissionHelper.kt        Location permission helpers
```

**A note on package names.** The folder is `com/bustracker`, but almost every file declares `package com.karroh.bussathi...`, which is the real namespace and application ID. Two files, `DriverModel.kt` and `LeaderboardActivity.kt`, declare `com.bustracker...` instead. Kotlin allows the folder and package to differ, so this builds fine, and `proguard-rules.pro` keeps the model classes in both packages. Keep that rule if you move files around.

## Firestore data model

**`drivers/{uid}`**

| Field | Type | Notes |
|---|---|---|
| `uid`, `name`, `email`, `phone` | string | |
| `vehicleNumber`, `licenseNumber` | string | |
| `region` | string | `Jammu` or `Srinagar` |
| `vehicleCapacity` | string | One of the three capacity options |
| `createdAt` | number | Epoch milliseconds |
| `totalKm` | number | Incremented after each uploaded trip; the leaderboard sorts on it |

**`trips/{tripId}`**

| Field | Type | Notes |
|---|---|---|
| `id` | string | `<VEHICLE>_<yyyyMMdd_HHmmss>` |
| `driverId`, `driverName`, `driverEmail` | string | |
| `startTime`, `endTime` | number | Epoch milliseconds |
| `startTimeString`, `endTimeString` | string | `dd:MM:yyyy HH:mm:ss` |
| `routePoints` | array | Each item: `latitude`, `longitude`, `timestamp`, `accuracy` |
| `totalDistance` | number | Kilometres |
| `status` | string | `COMPLETED` |

The whole route is stored inside one trip document. Firestore limits a document to 1 MiB, so an extremely long trip (roughly many hours of 5-second points) could fail to upload.

Your Firestore security rules need to let a signed-in driver create and read their own `drivers` document, update its `totalKm`, create `trips`, and read `drivers` ordered by `totalKm` for the leaderboard. The rules are managed in the Firebase console and are not part of this repository.

## Setup

### What you need
- **Android Studio** with a bundled JDK 17 or newer. Use a recent release that supports Android Gradle Plugin 8.13 (Narwhal 2025.1 or later).
- **Android SDK Platform 36** installed through the SDK Manager.
- A **Firebase project** with Email/Password sign-in enabled and a Cloud Firestore database.

### Steps

1. Clone the repository.

   ```bash
   git clone https://github.com/krishaniitjammu/app-bus.git
   ```

2. Create `local.properties` in the project root. Android Studio normally does this for you on first open. Otherwise copy `local.properties.template` and set `sdk.dir` to your Android SDK path.

3. Add your own `google-services.json`. In the Firebase console, add an Android app with the package name `com.karroh.bussathi`, download `google-services.json` and place it at `app/google-services.json`. The build fails without this file. It is deliberately not in the repository.

4. Check the JDK path. `gradle.properties` contains

   ```properties
   org.gradle.java.home=C:/Program Files/Android/Android Studio/jbr
   ```

   This points at the default Android Studio location on Windows. On macOS or Linux, or if Android Studio is installed elsewhere, change this line to your JDK path or delete it so Gradle uses the JDK chosen in Android Studio.

5. Open the project in Android Studio, let Gradle sync, then run it on a device or emulator.

### Running a debug build
The main screen refuses to run while **Developer Options** or **USB debugging** are on, and both are on for any phone or emulator you deploy to from Android Studio. For local testing, temporarily make `isDevOptionsEnabled()` in `MainActivity.kt` return `false`, and do not commit that change.

For simulating a route on an emulator, see [TESTING_GPS_ON_EMULATOR.md](TESTING_GPS_ON_EMULATOR.md). Remember that a trip needs at least 50 points (a little over 4 minutes) and some distance before it is uploaded.

## Building and signing a release

Release builds have R8 turned on: code is shrunk and obfuscated (`isMinifyEnabled = true`) and unused resources are removed (`isShrinkResources = true`). `gradle.properties` gives Gradle a 2 GB heap because R8 needs more than the default.

Signing is not configured in the Gradle files, so no keystore or password is stored in this repository. Sign through Android Studio:

1. **Build > Generate Signed App Bundle / APK**.
2. Choose **Android App Bundle**.
3. Select your upload keystore and enter its passwords.
4. Choose the **release** variant.

The bundle is written to `app/release/app-release.aab`.

Keep the keystore and its passwords somewhere safe and outside the project folder. If the upload key is lost, you have to ask Google Play support to reset it before you can publish updates. `.gitignore` blocks `*.jks`, `*.keystore`, `keystore.properties` and `signing.properties` so they cannot be committed by accident.

## Publishing to Google Play

Before each upload:

1. **Raise the version.** In `app/build.gradle.kts`, increase `versionCode` (it must be higher than every build already uploaded) and update `versionName`. The repository is currently at versionCode 24, versionName 1.78.
2. **Target SDK.** The app targets API 36, which meets Google Play's current target API requirement. If Play Console reports that a bundle targets an older API level, that bundle was built before this change; rebuild and upload again.
3. **Build the signed bundle** as described above.

**Deobfuscation file (`mapping.txt`).** Because R8 obfuscates the code, crash reports show scrambled names unless Google Play has the mapping file for that exact build.

- When you upload an `.aab`, the mapping file is included in the bundle automatically and Play Console picks it up. This removes the "no deobfuscation file associated with this App Bundle" warning.
- A copy is also written to `app/build/outputs/mapping/release/mapping.txt`. Keep a copy for every released version. To add it by hand, open the release in Play Console, go to the App bundle explorer, open the Downloads tab and upload it under ReTrace mapping file.
- The mapping file is different for every build. A file from one build will not decode crashes from another.
- `proguard-rules.pro` keeps source file and line number information so decoded stack traces point to real lines.

**Play Console declarations this app needs.**
- **Data safety**: the app collects precise location, name, email address, phone number and other identifiers (vehicle and license numbers), and uses them for app functionality and research.
- **Foreground service**: the app declares a `location` foreground service, which Play asks you to justify.
- **Privacy policy**: required because the app collects personal and location data.

**After changing data models.** Classes that Firestore or Gson read by reflection must keep their field names. The rules in `proguard-rules.pro` cover everything under `data.model`. If you add such a class in another package, add a keep rule for it, and always test a release build before publishing since debug builds are not obfuscated.

## What is not in this repository

These files are kept out by `.gitignore`. You need to supply your own where noted.

| File or folder | Why it is excluded | What to do |
|---|---|---|
| `app/google-services.json` | Firebase project configuration and API key | Download your own from the Firebase console |
| `*.jks`, `*.keystore`, `keystore.properties`, `signing.properties` | Release signing key and passwords | Keep your keystore outside the project |
| `local.properties` | Machine-specific SDK path | Created by Android Studio |
| `build/`, `app/build/`, `.gradle/` | Generated build output and caches | Rebuilt automatically |
| `app/release/`, `*.apk`, `*.aab` | Built app files | Rebuild from source |
| `*.hprof` | Java heap dumps (very large) | Not needed |
| `*.csv`, `*.gpx` | Exported trip data containing real driver locations | Never commit real trip data |
| Most of `.idea/` | Personal Android Studio settings | Not needed |

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| Build fails with "File google-services.json is missing" | Add your own file at `app/google-services.json`. |
| Gradle sync fails with a Java home error | Fix or remove `org.gradle.java.home` in `gradle.properties`. |
| "Security Restriction" dialog on the main screen | Developer Options or USB debugging is on. See [Running a debug build](#running-a-debug-build). |
| Trip ended but nothing appears in Firestore | The trip had fewer than 50 points or zero distance and was discarded by design. |
| Points are not being recorded | GPS is off, location permission was denied, or readings are being filtered for poor accuracy (worse than 100 m). |
| Leaderboard looks out of date | It refreshes at most once every 24 hours. Clearing app data forces a refresh. |
| Leaderboard is empty | Only drivers who have a `totalKm` value appear, which is set after their first uploaded trip. Firestore rules must also allow the query. |
| Release build crashes but debug works | An R8 keep rule is missing for a class read by reflection. See `proguard-rules.pro`. |
| Out of memory during a release build | Raise `-Xmx` in `org.gradle.jvmargs` in `gradle.properties`. |

## More documentation

- [SETUP.md](SETUP.md): Firebase project setup, step by step.
- [TESTING_GPS_ON_EMULATOR.md](TESTING_GPS_ON_EMULATOR.md): simulating movement on an emulator.
- [VIEWING_FIREBASE_DATA.md](VIEWING_FIREBASE_DATA.md): finding and reading trips in the Firebase console.
- [tools/EXPORT_TRIP_README.md](tools/EXPORT_TRIP_README.md): exporting a trip's route to CSV.
- [CONTRIBUTING.md](CONTRIBUTING.md): contribution guidelines.

These older guides were written for earlier versions of the app. Where they disagree with this README, this README is the current one.

## License

Released under the MIT License. See [LICENSE](LICENSE).
