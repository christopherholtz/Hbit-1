# Auth + Cloud Storage — Design Specification

**Status:** Approved (brainstorming session 2026-05-17)
**Subsystem:** #1 of 4 in the Hbit app roadmap
**Successor subsystems:** #2 RAG/LLM engine, #3 Gamification, #4 Onboarding & redesign

---

## 1. Purpose

Replace the current `shared_preferences`-based local persistence with Firebase
Authentication and Cloud Firestore so that:

1. Each user has an account (email/password or Google Sign-In).
2. Habit maps and per-day check-ins are stored in the cloud, syncing across
   devices and surviving uninstalls.
3. A clean repository abstraction sits between screens and Firebase, enabling
   testing and protecting future subsystems (LLM engine, gamification) from
   leaking Firebase types into UI code.
4. The data model lays the groundwork for gamification (streaks, calendar,
   scores) without requiring a schema migration later.

This subsystem ships **no gamification features and no LLM features**. It is
the foundation those subsystems will build on.

## 2. Non-goals

- Apple Sign-In (flagged; iOS App Store will require it before public release)
- Anonymous-first / guest accounts
- Migration of existing local-storage data (early-stage app — discard)
- Streaks, calendar, scores, badges (subsystem #3)
- LLM/RAG-driven habit generation (subsystem #2)
- Visual redesign / onboarding flow (subsystem #4)
- Integration tests against a live Firebase project
- Golden-file UI tests

## 3. Locked-in decisions

| # | Decision |
|---|----------|
| 1 | Backend = Firebase (Auth + Firestore; future Cloud Functions for subsystem #2) |
| 2 | Auth providers = Email/password + Google Sign-In |
| 3 | Hard sign-in gate (no anonymous use) |
| 4 | One active map per user + archived history |
| 5 | Per-day check-in log (event-sourced) |
| 6 | Check-ins stored day-keyed: `users/{uid}/maps/{mapId}/checkIns/{YYYY-MM-DD}` |
| 7 | Repository pattern between screens and Firestore |
| 8 | Timezone stored on user doc; user-overridable in profile |
| 9 | No check-in carry-over between maps (each map has independent history) |
| 10 | No migration of existing local-storage data |
| 11 | `ProfileScreen` included in this subsystem (sign-out + timezone) |
| 12 | DI via plain `InheritedWidget`; no `provider`/`riverpod` package added |

## 4. Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      Flutter App                          │
│                                                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Screens (AuthGate, SignIn, Register,              │  │
│  │           Aspiration, HabitMap, Profile,           │  │
│  │           MapHistory, ArchivedMapView)             │  │
│  └─────────────────────┬──────────────────────────────┘  │
│                        │ depends on                      │
│  ┌─────────────────────▼──────────────────────────────┐  │
│  │  Repository interfaces (abstract Dart classes):     │  │
│  │    AuthRepository                                   │  │
│  │    HabitMapRepository                               │  │
│  │    CheckInRepository                                │  │
│  └─────────────────────┬──────────────────────────────┘  │
│                        │ implemented by                  │
│  ┌─────────────────────▼──────────────────────────────┐  │
│  │  Firebase implementations:                          │  │
│  │    FirebaseAuthRepository                           │  │
│  │    FirestoreHabitMapRepository                      │  │
│  │    FirestoreCheckInRepository                       │  │
│  └─────────────────────┬──────────────────────────────┘  │
└────────────────────────┼──────────────────────────────────┘
                         │
                ┌────────▼────────┐
                │   Firebase       │
                │   - Auth         │
                │   - Firestore    │
                │     (offline     │
                │      cache on)   │
                └──────────────────┘
```

### Key properties

- **Repository pattern.** Screens import from `lib/data/{auth,habit_map,check_in}/*_repository.dart` (abstract). They never import `package:firebase_*` or `package:cloud_firestore` directly.
- **Streams as the contract.** Repositories expose `Stream<HabitMap?>`, `Stream<AppUser?>`, `Stream<Set<String>>`. Firestore's real-time listeners feed these streams.
- **Offline-first.** Firestore's offline cache is enabled (default). Toggling a habit while offline updates the local cache instantly; sync resumes on reconnect.
- **Auth gate at root.** `AuthGate` listens to `authRepository.userStream`. Unauthenticated → `SignInScreen`. Authenticated → app flow.
- **Engine stays pure.** `HabitEngine.generate` is unchanged. The map it produces is persisted via repository, not via `shared_preferences`.

## 5. Firestore schema

```
users/{uid}                                      ← user profile doc
  email: string
  displayName: string?
  photoUrl: string?
  timezone: string                               ← IANA zone, e.g. "Europe/Berlin"
  createdAt: timestamp
  lastSignInAt: timestamp
  activeMapId: string?                           ← pointer to current map

users/{uid}/maps/{mapId}                         ← one document per habit map
  id: string                                     ← duplicated for client convenience
  aspiration: {
    identity: string,
    timeframe: string,                           ← matches existing model field
    motivation: string,
    createdAt: timestamp
  }
  habits: [                                      ← bounded embedded array (≤25)
    {
      id: string,
      domain: string,
      domainLabel: string,
      behavior: string,
      anchorPrompt: string,
      tinyAction: string,
      celebration: string,
      impact: int,
      ease: int
    }, ...
  ]
  status: "active" | "archived"
  createdAt: timestamp
  archivedAt: timestamp?

users/{uid}/maps/{mapId}/checkIns/{YYYY-MM-DD}   ← day-keyed check-in log
  date: string                                   ← "2026-05-17" — matches doc id
  habitIds: [string]                             ← habits completed that day
  updatedAt: timestamp                           ← serverTimestamp
```

### Schema notes

- **Habits embedded.** ≤25 items, always read with the map, immutable after generation. Embedding = 1 doc read for the whole active map.
- **`activeMapId` denormalized** on `users/{uid}` for cheap cold start: 3 reads total (user + active map + today's check-ins).
- **`id` duplicated** inside the map doc so clients holding only the doc data have a complete object.
- **No `done` flag** on individual habits. "Done today" = `habitId ∈ checkIns/{today}.habitIds`.
- **Day-key format** is `YYYY-MM-DD` in the user's stored timezone, computed at write time. A traveler whose stored zone is `Europe/Berlin` writes Berlin-dated check-ins even if their phone is in `America/Los_Angeles`.
- **No empty check-in docs.** When the last habit id is removed from a day, the doc is deleted in the same transaction. This keeps future calendar queries clean.

## 6. Repository contracts

### `AuthRepository`

```dart
abstract class AuthRepository {
  Stream<AppUser?> get userStream;
  AppUser? get currentUser;

  Future<void> signInWithEmail(String email, String password);
  Future<void> registerWithEmail(
    String email,
    String password, {
    String? displayName,
  });
  Future<void> signInWithGoogle();
  Future<void> sendPasswordReset(String email);
  Future<void> signOut();
  Future<void> updateTimezone(String ianaZone);
}
```

### `HabitMapRepository`

```dart
abstract class HabitMapRepository {
  Stream<HabitMap?> activeMapStream(String uid);
  Future<HabitMap> createMap(
    String uid, {
    required Aspiration aspiration,
    required List<TinyHabit> habits,
  });
  Future<void> archiveActiveMap(String uid);
  Stream<List<HabitMapSummary>> archivedMapsStream(String uid);
  Future<HabitMap> getArchivedMap(String uid, String mapId);
}
```

### `CheckInRepository`

```dart
abstract class CheckInRepository {
  Stream<Set<String>> todayCheckInsStream(String uid, String mapId);
  Future<void> toggleCheckIn(String uid, String mapId, String habitId);
  Stream<Map<DateTime, Set<String>>> checkInsInRangeStream(
    String uid,
    String mapId, {
    required DateTime start,
    required DateTime end,
  });
}
```

### Domain types

- `AppUser` — `{uid, email, displayName?, photoUrl?, timezone, createdAt, lastSignInAt}`. Never expose `firebase_auth.User` to screens.
- `HabitMapSummary` — `{id, identity, archivedAt, habitCount}`. Lightweight projection for `MapHistoryScreen` list.

## 7. Screen inventory

| Screen | New / Modified / Deleted | Responsibility |
|---|---|---|
| `AuthGate` | New | Routes between `SignInScreen` / `AspirationScreen` / `HabitMapScreen` based on auth state and active-map presence |
| `SignInScreen` | New | Email/password sign-in, "Sign up" link, "Continue with Google", "Forgot password?" |
| `RegisterScreen` | New | Email/password/displayName registration. Creates `users/{uid}` doc on success |
| `ForgotPasswordDialog` | New | Single email field → `sendPasswordReset` |
| `AspirationScreen` | Modified | Submit calls `habitMapRepository.createMap` instead of in-memory callback |
| `HabitMapScreen` | Modified | Reads from `activeMapStream` + `todayCheckInsStream`. Tapping a habit calls `toggleCheckIn`. "New map" calls `archiveActiveMap` |
| `ProfileScreen` | New | Display name, email, timezone selector, sign-out, link to `MapHistoryScreen` |
| `MapHistoryScreen` | New | Lists archived maps via `archivedMapsStream` |
| `ArchivedMapViewScreen` | New | Read-only view of a single archived map |
| `RootPage` (in `main.dart`) | Deleted | Replaced by `AuthGate` |

## 8. Error handling

| Failure | User-facing behavior |
|---|---|
| Network down during sign-in | Inline banner: "No connection. Try again." Button stays enabled. |
| Wrong password | Inline error under password field: "Incorrect email or password." |
| Email already in use during register | Inline error: "An account with this email already exists. Sign in instead?" |
| Weak password (< 8 chars) | Client-side validator blocks submit |
| Firestore write fails after auth succeeded | Toast: "Saved locally — will sync when you're back online." (Firestore's offline cache handles this transparently; we just inform.) |
| Google sign-in cancelled by user | Silent no-op |
| Password reset email send fails | Toast: "Could not send reset email. Try again later." |

Repository methods throw typed exceptions from `lib/data/auth_exception.dart`:
`AuthException.wrongPassword`, `AuthException.emailInUse`, `AuthException.networkUnavailable`, `AuthException.unknown`. Screens map these to UI states.

## 9. Security rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {

    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;

      match /maps/{mapId} {
        allow read: if request.auth != null && request.auth.uid == uid;
        allow create: if request.auth != null
                      && request.auth.uid == uid
                      && isValidMap(request.resource.data);
        allow update: if request.auth != null
                      && request.auth.uid == uid
                      && isValidStatusUpdate(request.resource.data, resource.data);
        allow delete: if false;   // archive, never delete

        match /checkIns/{date} {
          allow read: if request.auth != null && request.auth.uid == uid;
          allow write: if request.auth != null
                       && request.auth.uid == uid
                       && isValidCheckIn(request.resource.data, date);
          allow delete: if request.auth != null && request.auth.uid == uid;
        }
      }
    }

    function isValidMap(d) {
      return d.keys().hasAll(['id', 'aspiration', 'habits', 'status', 'createdAt'])
          && d.status in ['active', 'archived']
          && d.habits is list
          && d.habits.size() <= 30;
    }

    function isValidStatusUpdate(next, prev) {
      // Only allow flipping status active → archived (and setting archivedAt).
      return next.status in ['active', 'archived']
          && next.id == prev.id
          && next.aspiration == prev.aspiration
          && next.habits == prev.habits;
    }

    function isValidCheckIn(d, date) {
      return d.date == date
          && d.habitIds is list
          && d.habitIds.size() > 0;   // empty docs must be deleted, not stored
    }
  }
}
```

## 10. Testing strategy

Four layers:

### Layer 1 — Pure Dart unit tests
- Models: serialization round-trip
- Engine: existing tests preserved (engine is unchanged)
- `TimezoneService`: day-rollover edge cases including DST and cross-timezone reckoning

### Layer 2 — Repository tests against `fake_cloud_firestore`
- `FirestoreHabitMapRepository`: create is transactional (map doc + `user.activeMapId` updated atomically); archive flips status and clears pointer; streams emit correctly
- `FirestoreCheckInRepository`: toggle adds/removes; doc deletes when last id removed; range query orders by date
- `FirebaseAuthRepository`: register creates `users/{uid}` doc; first Google sign-in creates user doc, second sign-in does not duplicate; sign-out emits null

### Layer 3 — Widget tests with fake repositories
- `AuthGate`: state machine (signed-out → SignIn; signed-in + no map → Aspiration; signed-in + map → HabitMap)
- `SignInScreen`: validation, error mapping
- `AspirationScreen`: submit calls `createMap` with correct args
- `HabitMapScreen`: renders from streams, toggling calls repo, archive confirmation dialog

### Layer 4 — Security rules tests
- Separate Node-based harness in `firestore-tests/` using `@firebase/rules-unit-testing`
- User A cannot read or write `users/B/*`
- Unauthenticated requests are denied
- Document-shape validation rejects malformed writes

### Coverage target
- Pure-Dart code (engine, models, services): 100%
- Repositories: all public methods, all error branches
- Screens: happy path + at least one error path each

## 11. Dependencies

### Added to `hbit/pubspec.yaml`
```yaml
dependencies:
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.0
  cloud_firestore: ^5.4.0
  google_sign_in: ^6.2.1
  intl: ^0.19.0
  timezone: ^0.9.4

dev_dependencies:
  fake_cloud_firestore: ^3.0.3
  firebase_auth_mocks: ^0.14.0
```

### Removed from `hbit/pubspec.yaml`
```yaml
shared_preferences   # no longer needed
```

### New top-level directory
```
firestore-tests/     # Node-based security rules test harness
  package.json
  rules.test.js
```

## 12. Platform configuration (out-of-code)

These steps are run once per developer and not committed source code:

- `flutterfire configure` (creates Firebase project, generates `firebase_options.dart`)
- Android: `google-services.json` placed in `hbit/android/app/`
- iOS: `GoogleService-Info.plist` placed in `hbit/ios/Runner/`; Google Sign-In URL scheme added to `Info.plist`
- Firebase Console: enable Email/Password and Google providers
- Firestore Security Rules deployed via `firebase deploy --only firestore:rules`

## 13. Risks and open questions

- **Apple Sign-In.** Required for iOS App Store before public submission. Scope-deferred but tracked as a known future addition.
- **Google Sign-In on web.** Web platform has a different config path. This spec covers Android + iOS; web is out of scope.
- **Email enumeration.** Firebase's default sign-in error messages can be used to enumerate registered emails. We accept this default for now; mitigations are a future hardening task.
- **Cost ceiling.** Firestore free tier (50k reads/day, 20k writes/day) is generous for early users. At ~100 DAU each opening the app twice with a 20-habit map and toggling 5 habits, daily reads ≈ 100 × 2 × 3 ≈ 600; writes ≈ 100 × 5 ≈ 500. Well within free tier.

---

End of specification.
