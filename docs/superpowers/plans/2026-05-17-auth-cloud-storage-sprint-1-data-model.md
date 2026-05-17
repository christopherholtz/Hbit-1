# Sprint 1 — Data Model Refactor + Dependencies

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** [`../specs/2026-05-17-auth-cloud-storage-design.md`](../specs/2026-05-17-auth-cloud-storage-design.md)
**Index:** [`2026-05-17-auth-cloud-storage-INDEX.md`](2026-05-17-auth-cloud-storage-INDEX.md)

**Goal:** Introduce Firebase libraries and new domain types without breaking the existing `shared_preferences` flow. At the end of this sprint, the app still runs and behaves exactly as it does today.

**Architecture:** Additive-only changes. New files (`AuthException`, `AppUser`, `HabitMapSummary`, `TimezoneService`). Existing files extended with optional fields preserving backward-compat JSON. No behavior changes.

**Tech Stack:** Dart 3.10+, Flutter 3.10+. New dependencies: `firebase_core`, `firebase_auth`, `cloud_firestore`, `google_sign_in`, `flutter_timezone`, `timezone`, `intl`. Dev: `fake_cloud_firestore`, `firebase_auth_mocks`.

---

## File-path convention

All Flutter paths are relative to `hbit/`. All commands run with `workdir: hbit/` unless otherwise stated.

## File structure created or modified

| Path | Role |
|---|---|
| `pubspec.yaml` | Add new dependencies; do not remove anything yet |
| `lib/data/auth_exception.dart` | NEW — typed errors thrown by auth code |
| `lib/models/app_user.dart` | NEW — domain user (decoupled from `firebase_auth.User`) |
| `lib/models/habit_map_summary.dart` | NEW — lightweight projection for archived-map lists |
| `lib/services/timezone_service.dart` | NEW — IANA timezone + day-key reckoning |
| `lib/models/habit_map.dart` | MODIFY — add optional `id`, `status`, `archivedAt`; backward-compat JSON |
| `lib/engine/habit_engine.dart` | MODIFY — add `generateHabits()` returning `List<TinyHabit>`; keep existing `generate()` |
| `test/models/app_user_test.dart` | NEW |
| `test/models/habit_map_summary_test.dart` | NEW |
| `test/models/habit_map_test.dart` | NEW |
| `test/data/auth_exception_test.dart` | NEW |
| `test/services/timezone_service_test.dart` | NEW |
| `test/engine/habit_engine_test.dart` | NEW — covers the new `generateHabits()` |

The existing `test/widget_test.dart` is **not** modified in this sprint.

---

## Task 1: Add new dependencies to pubspec.yaml

**Files:**
- Modify: `hbit/pubspec.yaml`

- [ ] **Step 1: Open pubspec.yaml and update the `dependencies` and `dev_dependencies` blocks**

Replace lines 30–48 (the entire `dependencies` and `dev_dependencies` blocks) with:

```yaml
dependencies:
  flutter:
    sdk: flutter

  cupertino_icons: ^1.0.8
  shared_preferences: ^2.5.5

  # Firebase platform + Auth + Firestore
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.0
  cloud_firestore: ^5.4.0
  google_sign_in: ^6.2.1

  # Time / locale utilities
  intl: ^0.19.0
  timezone: ^0.9.4
  flutter_timezone: ^3.0.1

dev_dependencies:
  flutter_test:
    sdk: flutter

  flutter_lints: ^6.0.0

  # In-memory Firebase fakes for repository and widget tests
  fake_cloud_firestore: ^3.0.3
  firebase_auth_mocks: ^0.14.1
```

`shared_preferences` stays in this sprint — it is removed in Sprint 4 once the cloud-backed flow replaces it.

- [ ] **Step 2: Run `flutter pub get`**

Run (workdir `hbit/`):
```powershell
flutter pub get
```
Expected: completes with no errors. New packages download. If a transitive version conflict surfaces, prefer the latest minor of `cloud_firestore` and bump the others as needed.

- [ ] **Step 3: Run existing tests to confirm no regression**

Run (workdir `hbit/`):
```powershell
flutter test
```
Expected: all 4 existing tests in `test/widget_test.dart` pass.

- [ ] **Step 4: Run analyzer**

Run (workdir `hbit/`):
```powershell
flutter analyze
```
Expected: zero issues.

- [ ] **Step 5: Commit**

```powershell
git add hbit/pubspec.yaml hbit/pubspec.lock
git commit -m "chore: add firebase, timezone, intl dependencies"
```

---

## Task 2: AuthException typed error class

**Files:**
- Create: `hbit/lib/data/auth_exception.dart`
- Test: `hbit/test/data/auth_exception_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/data/auth_exception_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/data/auth_exception.dart';

void main() {
  test('AuthException carries a kind and an optional message', () {
    const exception = AuthException(AuthErrorKind.wrongPassword, 'Wrong.');
    expect(exception.kind, AuthErrorKind.wrongPassword);
    expect(exception.message, 'Wrong.');
  });

  test('AuthException.toString exposes kind and message', () {
    const exception = AuthException(AuthErrorKind.emailInUse, 'Taken.');
    expect(exception.toString(), contains('emailInUse'));
    expect(exception.toString(), contains('Taken.'));
  });

  test('all AuthErrorKind values are listed', () {
    expect(AuthErrorKind.values, containsAll(<AuthErrorKind>[
      AuthErrorKind.wrongPassword,
      AuthErrorKind.emailInUse,
      AuthErrorKind.invalidEmail,
      AuthErrorKind.weakPassword,
      AuthErrorKind.userNotFound,
      AuthErrorKind.networkUnavailable,
      AuthErrorKind.cancelled,
      AuthErrorKind.unknown,
    ]));
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/data/auth_exception_test.dart
```
Expected: FAIL — `lib/data/auth_exception.dart` does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/data/auth_exception.dart`:

```dart
/// Categorised authentication errors. Repositories translate Firebase-specific
/// error codes into these so screens can render appropriate UI without
/// importing the Firebase SDKs.
enum AuthErrorKind {
  wrongPassword,
  emailInUse,
  invalidEmail,
  weakPassword,
  userNotFound,
  networkUnavailable,
  cancelled,
  unknown,
}

class AuthException implements Exception {
  const AuthException(this.kind, [this.message]);

  final AuthErrorKind kind;
  final String? message;

  @override
  String toString() => 'AuthException(${kind.name}): ${message ?? ''}';
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/data/auth_exception_test.dart
```
Expected: PASS, all 3 tests.

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass, zero issues.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/data/auth_exception.dart hbit/test/data/auth_exception_test.dart
git commit -m "feat: add AuthException with AuthErrorKind enum"
```

---

## Task 3: AppUser domain model

**Files:**
- Create: `hbit/lib/models/app_user.dart`
- Test: `hbit/test/models/app_user_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/models/app_user_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/models/app_user.dart';

void main() {
  final created = DateTime.utc(2026, 1, 2, 3, 4, 5);
  final signedIn = DateTime.utc(2026, 5, 17, 10, 0, 0);

  AppUser user() => AppUser(
        uid: 'uid-123',
        email: 'a@b.test',
        displayName: 'Alice',
        photoUrl: null,
        timezone: 'Europe/Berlin',
        createdAt: created,
        lastSignInAt: signedIn,
      );

  test('AppUser carries identifying fields', () {
    final u = user();
    expect(u.uid, 'uid-123');
    expect(u.email, 'a@b.test');
    expect(u.displayName, 'Alice');
    expect(u.timezone, 'Europe/Berlin');
  });

  test('AppUser supports JSON round-trip', () {
    final u = user();
    final restored = AppUser.fromJson(u.toJson());
    expect(restored.uid, u.uid);
    expect(restored.email, u.email);
    expect(restored.displayName, u.displayName);
    expect(restored.timezone, u.timezone);
    expect(restored.createdAt, u.createdAt);
    expect(restored.lastSignInAt, u.lastSignInAt);
  });

  test('AppUser.copyWith updates only the provided field', () {
    final u = user();
    final copy = u.copyWith(timezone: 'America/Los_Angeles');
    expect(copy.timezone, 'America/Los_Angeles');
    expect(copy.uid, u.uid);
    expect(copy.email, u.email);
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/models/app_user_test.dart
```
Expected: FAIL — `lib/models/app_user.dart` does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/models/app_user.dart`:

```dart
/// Domain representation of a signed-in user. Decoupled from
/// `firebase_auth.User` so that no Firebase types leak into screens or tests.
class AppUser {
  const AppUser({
    required this.uid,
    required this.email,
    required this.timezone,
    required this.createdAt,
    required this.lastSignInAt,
    this.displayName,
    this.photoUrl,
  });

  final String uid;
  final String email;
  final String? displayName;
  final String? photoUrl;

  /// IANA timezone identifier, e.g. "Europe/Berlin".
  final String timezone;

  final DateTime createdAt;
  final DateTime lastSignInAt;

  AppUser copyWith({
    String? email,
    String? displayName,
    String? photoUrl,
    String? timezone,
    DateTime? lastSignInAt,
  }) =>
      AppUser(
        uid: uid,
        email: email ?? this.email,
        displayName: displayName ?? this.displayName,
        photoUrl: photoUrl ?? this.photoUrl,
        timezone: timezone ?? this.timezone,
        createdAt: createdAt,
        lastSignInAt: lastSignInAt ?? this.lastSignInAt,
      );

  Map<String, dynamic> toJson() => {
        'uid': uid,
        'email': email,
        'displayName': displayName,
        'photoUrl': photoUrl,
        'timezone': timezone,
        'createdAt': createdAt.toIso8601String(),
        'lastSignInAt': lastSignInAt.toIso8601String(),
      };

  factory AppUser.fromJson(Map<String, dynamic> json) => AppUser(
        uid: json['uid'] as String,
        email: json['email'] as String,
        displayName: json['displayName'] as String?,
        photoUrl: json['photoUrl'] as String?,
        timezone: json['timezone'] as String,
        createdAt: DateTime.parse(json['createdAt'] as String),
        lastSignInAt: DateTime.parse(json['lastSignInAt'] as String),
      );
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/models/app_user_test.dart
```
Expected: PASS, all 3 tests.

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass, zero issues.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/models/app_user.dart hbit/test/models/app_user_test.dart
git commit -m "feat: add AppUser domain model"
```

---

## Task 4: HabitMapSummary domain model

**Files:**
- Create: `hbit/lib/models/habit_map_summary.dart`
- Test: `hbit/test/models/habit_map_summary_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/models/habit_map_summary_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/models/habit_map_summary.dart';

void main() {
  test('HabitMapSummary carries lightweight projection fields', () {
    final archived = DateTime.utc(2026, 4, 1);
    final summary = HabitMapSummary(
      id: 'map-1',
      identity: 'a senior engineering leader',
      archivedAt: archived,
      habitCount: 12,
    );

    expect(summary.id, 'map-1');
    expect(summary.identity, 'a senior engineering leader');
    expect(summary.archivedAt, archived);
    expect(summary.habitCount, 12);
  });

  test('HabitMapSummary supports JSON round-trip', () {
    final summary = HabitMapSummary(
      id: 'map-2',
      identity: 'a technical writer',
      archivedAt: DateTime.utc(2026, 3, 14, 9, 30),
      habitCount: 9,
    );

    final restored = HabitMapSummary.fromJson(summary.toJson());
    expect(restored.id, summary.id);
    expect(restored.identity, summary.identity);
    expect(restored.archivedAt, summary.archivedAt);
    expect(restored.habitCount, summary.habitCount);
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/models/habit_map_summary_test.dart
```
Expected: FAIL — file does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/models/habit_map_summary.dart`:

```dart
/// Lightweight projection of an archived [HabitMap], used by the map history
/// list to avoid loading the full habit array for every row.
class HabitMapSummary {
  const HabitMapSummary({
    required this.id,
    required this.identity,
    required this.archivedAt,
    required this.habitCount,
  });

  final String id;
  final String identity;
  final DateTime archivedAt;
  final int habitCount;

  Map<String, dynamic> toJson() => {
        'id': id,
        'identity': identity,
        'archivedAt': archivedAt.toIso8601String(),
        'habitCount': habitCount,
      };

  factory HabitMapSummary.fromJson(Map<String, dynamic> json) =>
      HabitMapSummary(
        id: json['id'] as String,
        identity: json['identity'] as String,
        archivedAt: DateTime.parse(json['archivedAt'] as String),
        habitCount: json['habitCount'] as int,
      );
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/models/habit_map_summary_test.dart
```
Expected: PASS.

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/models/habit_map_summary.dart hbit/test/models/habit_map_summary_test.dart
git commit -m "feat: add HabitMapSummary projection model"
```

---

## Task 5: TimezoneService

The service computes the user's "today" string for check-in document IDs, using a stored IANA zone (so a traveller's day boundaries do not shift).

**Files:**
- Create: `hbit/lib/services/timezone_service.dart`
- Test: `hbit/test/services/timezone_service_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/services/timezone_service_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/services/timezone_service.dart';
import 'package:timezone/data/latest.dart' as tzdata;

void main() {
  setUpAll(tzdata.initializeTimeZones);

  group('TimezoneService.todayKey', () {
    test('returns YYYY-MM-DD in the given IANA zone', () {
      // 2026-05-17 23:30 in Berlin is the same instant as 2026-05-17 21:30 UTC.
      final instant = DateTime.utc(2026, 5, 17, 21, 30);
      final service = TimezoneService();

      final key = service.todayKey(timezone: 'Europe/Berlin', now: instant);

      expect(key, '2026-05-17');
    });

    test('crosses to the next day at local midnight', () {
      // 2026-05-17 22:05 UTC is 2026-05-18 00:05 in Berlin (CEST = UTC+2).
      final instant = DateTime.utc(2026, 5, 17, 22, 5);
      final service = TimezoneService();

      final key = service.todayKey(timezone: 'Europe/Berlin', now: instant);

      expect(key, '2026-05-18');
    });

    test('a traveller keeps their stored zone, not device-local', () {
      // Same instant: in Berlin it is the 18th, in Los Angeles still the 17th.
      final instant = DateTime.utc(2026, 5, 17, 22, 5);
      final service = TimezoneService();

      expect(
        service.todayKey(timezone: 'Europe/Berlin', now: instant),
        '2026-05-18',
      );
      expect(
        service.todayKey(timezone: 'America/Los_Angeles', now: instant),
        '2026-05-17',
      );
    });

    test('throws ArgumentError on unknown zone', () {
      final service = TimezoneService();
      expect(
        () => service.todayKey(
          timezone: 'Not/A_Zone',
          now: DateTime.utc(2026, 5, 17),
        ),
        throwsArgumentError,
      );
    });
  });

  group('TimezoneService.parseDayKey', () {
    test('parses a YYYY-MM-DD string into the start-of-day in the zone', () {
      final service = TimezoneService();
      final parsed = service.parseDayKey('2026-05-17', timezone: 'Europe/Berlin');

      // Start of 2026-05-17 in Berlin (CEST) is 2026-05-16 22:00 UTC.
      expect(parsed.toUtc(), DateTime.utc(2026, 5, 16, 22, 0));
    });
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/services/timezone_service_test.dart
```
Expected: FAIL — `lib/services/timezone_service.dart` does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/services/timezone_service.dart`:

```dart
import 'package:timezone/timezone.dart' as tz;

/// Computes calendar-day strings for a stored user timezone.
///
/// The `timezone` package's data must be initialised before use (typically
/// via `timezone/data/latest.dart` at app start, and in `setUpAll` in tests).
class TimezoneService {
  /// Returns the `YYYY-MM-DD` key representing "today" in [timezone] for the
  /// given [now] instant. If [now] is null, the current wall-clock instant is
  /// used.
  String todayKey({required String timezone, DateTime? now}) {
    final location = _locationOrThrow(timezone);
    final instant = now ?? DateTime.now();
    final local = tz.TZDateTime.from(instant, location);
    return _format(local.year, local.month, local.day);
  }

  /// Parses a `YYYY-MM-DD` key into the start-of-day [DateTime] in [timezone].
  DateTime parseDayKey(String dayKey, {required String timezone}) {
    final location = _locationOrThrow(timezone);
    final parts = dayKey.split('-');
    if (parts.length != 3) {
      throw ArgumentError.value(dayKey, 'dayKey', 'Expected YYYY-MM-DD');
    }
    final year = int.parse(parts[0]);
    final month = int.parse(parts[1]);
    final day = int.parse(parts[2]);
    return tz.TZDateTime(location, year, month, day);
  }

  tz.Location _locationOrThrow(String timezone) {
    try {
      return tz.getLocation(timezone);
    } on tz.LocationNotFoundException {
      throw ArgumentError.value(timezone, 'timezone', 'Unknown IANA zone');
    }
  }

  String _format(int year, int month, int day) =>
      '${year.toString().padLeft(4, '0')}-'
      '${month.toString().padLeft(2, '0')}-'
      '${day.toString().padLeft(2, '0')}';
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/services/timezone_service_test.dart
```
Expected: PASS, all 5 tests.

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/services/timezone_service.dart hbit/test/services/timezone_service_test.dart
git commit -m "feat: add TimezoneService for day-key reckoning"
```

---

## Task 6: Extend HabitMap with id / status / archivedAt (backward compatible)

The existing JSON shape persists user data via `shared_preferences`. We must keep loading old data, while adding the new fields used by Firestore-backed maps.

**Files:**
- Modify: `hbit/lib/models/habit_map.dart`
- Create: `hbit/test/models/habit_map_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/models/habit_map_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/models/aspiration.dart';
import 'package:hbit/models/habit_map.dart';
import 'package:hbit/models/tiny_habit.dart';

void main() {
  Aspiration anyAspiration() => Aspiration(
        identity: 'a senior engineer',
        timeframe: '3 years',
        motivation: 'build great products',
        createdAt: DateTime.utc(2026, 5, 17),
      );

  TinyHabit anyHabit({String id = 'h1'}) => TinyHabit(
        id: id,
        domain: 'engineering',
        domainLabel: 'Engineering',
        behavior: 'Daily PR review',
        anchorPrompt: 'After I sit at my desk',
        tinyAction: 'I will open one PR',
        celebration: 'I say "nice"',
        impact: 4,
        ease: 4,
      );

  test('legacy JSON without status/id/archivedAt still parses', () {
    // Shape persisted by the pre-cloud app.
    final legacyJson = {
      'aspiration': anyAspiration().toJson(),
      'habits': [anyHabit().toJson()],
    };

    final map = HabitMap.fromJson(legacyJson);

    expect(map.habits, hasLength(1));
    expect(map.id, isNull);
    expect(map.status, HabitMapStatus.active);
    expect(map.archivedAt, isNull);
  });

  test('JSON round-trip preserves id, status, archivedAt', () {
    final archived = DateTime.utc(2026, 4, 1, 10);
    final original = HabitMap(
      id: 'map-42',
      aspiration: anyAspiration(),
      habits: [anyHabit()],
      status: HabitMapStatus.archived,
      archivedAt: archived,
    );

    final restored = HabitMap.fromJson(original.toJson());

    expect(restored.id, 'map-42');
    expect(restored.status, HabitMapStatus.archived);
    expect(restored.archivedAt, archived);
    expect(restored.habits, hasLength(1));
  });

  test('habitsByDomain groups by label preserving order', () {
    final map = HabitMap(
      aspiration: anyAspiration(),
      habits: [
        anyHabit(id: 'a')..done = false,
        anyHabit(id: 'b'),
      ],
    );
    expect(map.habitsByDomain['Engineering'], hasLength(2));
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/models/habit_map_test.dart
```
Expected: FAIL — `HabitMapStatus`, `id`, `archivedAt` do not exist.

- [ ] **Step 3: Write the implementation**

Replace the contents of `hbit/lib/models/habit_map.dart` with:

```dart
import 'aspiration.dart';
import 'tiny_habit.dart';

/// Lifecycle state of a [HabitMap]. Sprint 1 introduces the type and
/// backward-compat parsing; Sprint 4 uses it to drive cloud archiving.
enum HabitMapStatus { active, archived }

/// A generated Tiny Habits map: one aspiration plus the set of tiny habits
/// the engine selected to move the user toward it.
class HabitMap {
  HabitMap({
    required this.aspiration,
    required this.habits,
    this.id,
    this.status = HabitMapStatus.active,
    this.archivedAt,
  });

  /// Server-assigned id once the map is persisted to Firestore. Null for maps
  /// that have only ever lived in `shared_preferences`.
  final String? id;

  final Aspiration aspiration;
  final List<TinyHabit> habits;
  final HabitMapStatus status;
  final DateTime? archivedAt;

  int get doneCount => habits.where((h) => h.done).length;

  /// Habits grouped by domain label, preserving the engine's ordering.
  Map<String, List<TinyHabit>> get habitsByDomain {
    final grouped = <String, List<TinyHabit>>{};
    for (final habit in habits) {
      grouped.putIfAbsent(habit.domainLabel, () => []).add(habit);
    }
    return grouped;
  }

  Map<String, dynamic> toJson() => {
        if (id != null) 'id': id,
        'aspiration': aspiration.toJson(),
        'habits': habits.map((h) => h.toJson()).toList(),
        'status': status.name,
        if (archivedAt != null) 'archivedAt': archivedAt!.toIso8601String(),
      };

  factory HabitMap.fromJson(Map<String, dynamic> json) => HabitMap(
        id: json['id'] as String?,
        aspiration:
            Aspiration.fromJson(json['aspiration'] as Map<String, dynamic>),
        habits: (json['habits'] as List)
            .map((e) => TinyHabit.fromJson(e as Map<String, dynamic>))
            .toList(),
        status: _statusFromJson(json['status']),
        archivedAt: json['archivedAt'] != null
            ? DateTime.parse(json['archivedAt'] as String)
            : null,
      );

  static HabitMapStatus _statusFromJson(Object? raw) {
    if (raw is! String) return HabitMapStatus.active;
    return HabitMapStatus.values.firstWhere(
      (s) => s.name == raw,
      orElse: () => HabitMapStatus.active,
    );
  }
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/models/habit_map_test.dart
```
Expected: PASS, all 3 tests.

- [ ] **Step 5: Run full test suite to confirm legacy widget_test still passes**

```powershell
flutter test
flutter analyze
```
Expected: all 4 original tests + all new tests pass; zero analyzer issues.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/models/habit_map.dart hbit/test/models/habit_map_test.dart
git commit -m "feat: extend HabitMap with id/status/archivedAt (backward compatible)"
```

---

## Task 7: Add HabitEngine.generateHabits() — list-only variant

Sprint 2 needs the engine to produce a habit *list* (so the repository can wrap it into a `HabitMap` with a server-assigned id). The existing `HabitEngine.generate(aspiration)` is preserved for the current `AspirationScreen` flow.

**Files:**
- Modify: `hbit/lib/engine/habit_engine.dart`
- Create: `hbit/test/engine/habit_engine_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/engine/habit_engine_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/engine/habit_engine.dart';
import 'package:hbit/models/aspiration.dart';

void main() {
  Aspiration aspirationFor(String identity, String motivation,
          [String timeframe = '3 years']) =>
      Aspiration(
        identity: identity,
        timeframe: timeframe,
        motivation: motivation,
        createdAt: DateTime(2026),
      );

  test('generateHabits returns the same list as generate(...).habits', () {
    final aspiration = aspirationFor(
      'a senior software engineer',
      'I want to build great products and lead a team',
    );

    final viaList = HabitEngine.generateHabits(aspiration);
    final viaMap = HabitEngine.generate(aspiration);

    expect(viaList.length, viaMap.habits.length);
    for (var i = 0; i < viaList.length; i++) {
      expect(viaList[i].id, viaMap.habits[i].id);
      expect(viaList[i].behavior, viaMap.habits[i].behavior);
    }
  });

  test('generateHabits respects timeframe quota', () {
    final short = HabitEngine.generateHabits(
      aspirationFor('a senior engineering leader',
          'I want to manage a team', '1 year'),
    );
    final long = HabitEngine.generateHabits(
      aspirationFor('a senior engineering leader',
          'I want to manage a team', '10 years'),
    );

    expect(short.length, lessThan(long.length));
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/engine/habit_engine_test.dart
```
Expected: FAIL — `generateHabits` is not defined.

- [ ] **Step 3: Modify the engine**

Replace the contents of `hbit/lib/engine/habit_engine.dart` with:

```dart
import '../data/behavior_library.dart';
import '../models/aspiration.dart';
import '../models/habit_map.dart';
import '../models/tiny_habit.dart';

/// The Tiny Habits rule engine.
///
/// Turns an [Aspiration] into a list of [TinyHabit]s following BJ Fogg's
/// method:
///   1. Match the aspiration text to relevant career domains.
///   2. Pull candidate behaviors from those domains ("Magic Wanding").
///   3. Rank by Golden Behavior score (impact x feasibility).
///   4. Always fold in universal growth foundations.
///
/// Sprint 1 splits the API into `generateHabits` (list, used by repositories
/// in Sprint 2+) and `generate` (full map, retained for the existing flow).
class HabitEngine {
  const HabitEngine._();

  /// Returns the ordered list of tiny habits for [aspiration]. No map id is
  /// assigned; callers wrap the list into a [HabitMap] themselves.
  static List<TinyHabit> generateHabits(Aspiration aspiration) {
    final text =
        '${aspiration.identity} ${aspiration.motivation}'.toLowerCase();

    final domainScores = <String, int>{};
    kDomainKeywords.forEach((domain, keywords) {
      var score = 0;
      for (final keyword in keywords) {
        if (text.contains(keyword)) score++;
      }
      if (score > 0) domainScores[domain] = score;
    });

    final focusDomains = domainScores.keys.toList()
      ..sort((a, b) => domainScores[b]!.compareTo(domainScores[a]!));
    final selectedDomains = focusDomains.take(3).toList();

    final domainQuota = _domainQuota(aspiration.timeframe);

    final domainPool = kBehaviorLibrary
        .where((b) => selectedDomains.contains(b.domain))
        .toList()
      ..sort((a, b) => _golden(b).compareTo(_golden(a)));

    final universalPool = kBehaviorLibrary
        .where((b) => b.domain == kUniversalDomain)
        .toList()
      ..sort((a, b) => _golden(b).compareTo(_golden(a)));

    final universalQuota = selectedDomains.isEmpty ? universalPool.length : 3;

    final selected = <BehaviorTemplate>[
      ...domainPool.take(domainQuota),
      ...universalPool.take(universalQuota),
    ];

    final habits = <TinyHabit>[];
    for (var i = 0; i < selected.length; i++) {
      final template = selected[i];
      habits.add(
        TinyHabit(
          id: '${template.domain}-$i',
          domain: template.domain,
          domainLabel: kDomainLabels[template.domain] ?? template.domain,
          behavior: template.behavior,
          anchorPrompt: template.anchorPrompt,
          tinyAction: template.tinyAction,
          celebration: template.celebration,
          impact: template.impact,
          ease: template.ease,
        ),
      );
    }

    return habits;
  }

  /// Legacy entry point: returns a full [HabitMap]. Retained for the
  /// pre-cloud `AspirationScreen` flow; Sprint 4 removes the call site.
  static HabitMap generate(Aspiration aspiration) {
    return HabitMap(
      aspiration: aspiration,
      habits: generateHabits(aspiration),
    );
  }

  static int _golden(BehaviorTemplate b) => b.impact * b.ease;

  static int _domainQuota(String timeframe) {
    final years = _parseYears(timeframe);
    if (years <= 1) return 4;
    if (years <= 4) return 5;
    return 6;
  }

  static int _parseYears(String timeframe) {
    final lower = timeframe.toLowerCase();
    final match = RegExp(r'\d+').firstMatch(lower);
    if (match == null) return 3;
    final value = int.tryParse(match.group(0)!) ?? 3;
    if (lower.contains('month')) {
      return (value / 12).ceil().clamp(1, 99);
    }
    return value.clamp(1, 99);
  }
}
```

- [ ] **Step 4: Run the new test to confirm it passes**

```powershell
flutter test test/engine/habit_engine_test.dart
```
Expected: PASS.

- [ ] **Step 5: Run full test suite (including the original widget_test)**

```powershell
flutter test
flutter analyze
```
Expected: all original tests still pass; zero analyzer issues.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/engine/habit_engine.dart hbit/test/engine/habit_engine_test.dart
git commit -m "refactor: split HabitEngine.generateHabits from generate"
```

---

## Sprint 1 — Done definition

- [ ] `flutter test` from `hbit/` passes (all original + new tests)
- [ ] `flutter analyze` from `hbit/` reports zero issues
- [ ] `flutter run` launches the app and the existing aspiration → habit-map → toggle → reset flow works identically to before
- [ ] All 7 tasks committed in sequence
- [ ] No file in `hbit/lib/` imports `firebase_*` or `cloud_firestore` yet (verify with `grep -r "firebase" hbit/lib` — should be empty)
- [ ] Tag the sprint completion: `git tag sprint-1-data-model`

Next: [Sprint 2 — Repository layer](2026-05-17-auth-cloud-storage-sprint-2-repositories.md).
