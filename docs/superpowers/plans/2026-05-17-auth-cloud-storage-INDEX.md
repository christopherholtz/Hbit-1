# Auth + Cloud Storage — Implementation Plan Index

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement each sprint task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** [`docs/superpowers/specs/2026-05-17-auth-cloud-storage-design.md`](../specs/2026-05-17-auth-cloud-storage-design.md)

**Goal:** Replace local `shared_preferences` persistence with Firebase Auth + Firestore, behind a repository abstraction, while preserving the existing Tiny Habits engine and producing testable, working software at each sprint boundary.

**Architecture:** Repository pattern (`AuthRepository`, `HabitMapRepository`, `CheckInRepository`) sits between Flutter screens and Firebase SDKs. Screens never import `firebase_*` directly. Day-keyed check-in collection is the source of truth for habit completion; per-habit `done` booleans are removed.

**Tech Stack:** Flutter 3.10+, Dart 3.10+, `firebase_core`, `firebase_auth`, `cloud_firestore`, `google_sign_in`, `flutter_timezone`, `timezone`, `intl`. Dev: `fake_cloud_firestore`, `firebase_auth_mocks`, `flutter_test`. Security rules tested via Node `@firebase/rules-unit-testing`.

---

## Sprint sequencing

```
Sprint 1 ──► Sprint 2 ──► Sprint 3 ──► Sprint 4 ──► Sprint 5
(data model)  (repos)      (auth UI)    (cloud      (history,
                                         storage)    security)
```

Each sprint produces working, testable software. Sprints must be executed in order — later sprints depend on artifacts from earlier ones.

| Sprint | File | Goal | Working state at end |
|---|---|---|---|
| 1 | [`-sprint-1-data-model.md`](2026-05-17-auth-cloud-storage-sprint-1-data-model.md) | Add Firebase deps. Introduce new domain types. Refactor existing models without breaking the local-storage flow. | App runs identically to today, but with new types in place and Firebase libraries linked. |
| 2 | [`-sprint-2-repositories.md`](2026-05-17-auth-cloud-storage-sprint-2-repositories.md) | Define repository interfaces. Build fake implementations for tests. Implement Firebase-backed implementations with full repository tests. | Repositories exist and are fully tested, but the app still uses `HabitStorage`. |
| 3 | [`-sprint-3-auth-screens.md`](2026-05-17-auth-cloud-storage-sprint-3-auth-screens.md) | Add `AuthGate`, `SignInScreen`, `RegisterScreen`, `ForgotPasswordDialog`. Initialise Firebase in `main.dart`. | A user must sign in to reach the app. After sign-in, the existing local-storage map flow is used. |
| 4 | [`-sprint-4-map-cloud.md`](2026-05-17-auth-cloud-storage-sprint-4-map-cloud.md) | Migrate `AspirationScreen` and `HabitMapScreen` to use the repositories. Remove `done` from `TinyHabit`. Delete `HabitStorage`. Add `ProfileScreen`. | App is fully cloud-backed. Toggling a habit writes a check-in to Firestore. Generating a new map archives the old one. |
| 5 | [`-sprint-5-history-security.md`](2026-05-17-auth-cloud-storage-sprint-5-history-security.md) | Add `MapHistoryScreen` and `ArchivedMapViewScreen`. Deploy Firestore Security Rules with a Node test harness. Update README with Firebase setup instructions. | Map history is browsable. Security rules are enforced and tested. New developers can clone-and-run with documented setup. |

## Cross-sprint conventions

### File-path prefix
All Flutter paths are relative to `hbit/`. Example: `lib/models/app_user.dart` means `hbit/lib/models/app_user.dart` on disk. The plan uses the short form unless ambiguity demands the full path.

### Working directory
All `flutter` / `dart` / `pub` commands run with `workdir: hbit/`. All `git` and root-level commands run from the repo root.

### Commit style
- Type prefixes: `feat:`, `fix:`, `refactor:`, `test:`, `chore:`, `docs:`
- One logical change per commit
- Each task ends with a commit step

### TDD cycle (every code task)
1. Write the failing test
2. Run it; confirm it fails for the expected reason
3. Write the minimal implementation to pass
4. Run the test; confirm it passes
5. Run the full test suite; confirm nothing else broke
6. Commit

### Running tests
```powershell
flutter test
```
Run from `hbit/`. For a single test file:
```powershell
flutter test test/path/to/file_test.dart
```

### Linting
```powershell
flutter analyze
```
Run from `hbit/`. Must pass with zero issues before any commit.

## Out-of-band setup (one-time, before Sprint 3)

These are not part of any task because they are not source code, but Sprint 3 depends on them:

1. Install Firebase CLI: `npm install -g firebase-tools` then `firebase login`
2. Install FlutterFire CLI: `dart pub global activate flutterfire_cli`
3. Create a Firebase project at https://console.firebase.google.com
4. Run `flutterfire configure` from `hbit/` and select the project (generates `lib/firebase_options.dart`)
5. In the Firebase Console, enable **Authentication → Sign-in method → Email/Password** and **Google**
6. Android: place `google-services.json` in `hbit/android/app/`
7. iOS: place `GoogleService-Info.plist` in `hbit/ios/Runner/`

Sprint 5 covers deploying Firestore Security Rules.

## Done definition

The subsystem is complete when:

- All five sprints are merged
- `flutter test` passes from `hbit/`
- `flutter analyze` passes with zero issues from `hbit/`
- `npm test` in `firestore-tests/` passes (security rules)
- A new user can: register → enter aspiration → see habit map → toggle habits → archive map → view history, all backed by Firestore
- Restarting the app preserves state
- Signing out and back in on a second device shows the same data
