# Sprint 2 — Repository Layer

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** [`../specs/2026-05-17-auth-cloud-storage-design.md`](../specs/2026-05-17-auth-cloud-storage-design.md)
**Index:** [`2026-05-17-auth-cloud-storage-INDEX.md`](2026-05-17-auth-cloud-storage-INDEX.md)
**Prerequisite:** Sprint 1 complete.

**Goal:** Define three repository interfaces (`AuthRepository`, `HabitMapRepository`, `CheckInRepository`), wire up a DI container, build in-memory fake implementations for testing, and ship Firebase-backed implementations with full repository-level test coverage. The app still uses `HabitStorage` at the end of this sprint — repositories are not yet consumed by screens.

**Architecture:** Each repository is a Dart `abstract class`. Firebase implementations live in `lib/data/<topic>/firebase_*.dart` and translate domain types to/from Firestore documents. Fakes live in `test/fakes/` and implement the same interfaces in memory. Tests use `fake_cloud_firestore` and `firebase_auth_mocks`.

**Tech Stack:** Same as Sprint 1.

---

## File structure created

| Path | Role |
|---|---|
| `lib/data/auth/auth_repository.dart` | NEW — abstract interface |
| `lib/data/auth/firebase_auth_repository.dart` | NEW — Firebase impl |
| `lib/data/habit_map/habit_map_repository.dart` | NEW — abstract interface |
| `lib/data/habit_map/firestore_habit_map_repository.dart` | NEW — Firestore impl |
| `lib/data/check_in/check_in_repository.dart` | NEW — abstract interface |
| `lib/data/check_in/firestore_check_in_repository.dart` | NEW — Firestore impl |
| `lib/data/repositories.dart` | NEW — container |
| `lib/app/repositories_scope.dart` | NEW — `InheritedWidget` for DI |
| `test/fakes/fake_auth_repository.dart` | NEW |
| `test/fakes/fake_habit_map_repository.dart` | NEW |
| `test/fakes/fake_check_in_repository.dart` | NEW |
| `test/data/auth/firebase_auth_repository_test.dart` | NEW |
| `test/data/habit_map/firestore_habit_map_repository_test.dart` | NEW |
| `test/data/check_in/firestore_check_in_repository_test.dart` | NEW |
| `test/fakes/fake_auth_repository_test.dart` | NEW (sanity test for fake) |

---

## Task 1: AuthRepository abstract interface

**Files:**
- Create: `hbit/lib/data/auth/auth_repository.dart`

This task is an interface definition only — no tests required at this stage; consumers will exercise it.

- [ ] **Step 1: Create the directory and file**

Create `hbit/lib/data/auth/auth_repository.dart`:

```dart
import '../../models/app_user.dart';

/// Repository contract for authentication. Implementations may be backed by
/// Firebase, fakes for tests, or anything that fulfils the contract.
abstract class AuthRepository {
  /// Emits the currently signed-in user, or `null` when signed out. Emits
  /// immediately on subscription with the current state.
  Stream<AppUser?> get userStream;

  /// Snapshot of the current user (synchronous). Null when signed out or
  /// before the first emission of [userStream] completes.
  AppUser? get currentUser;

  Future<void> signInWithEmail({
    required String email,
    required String password,
  });

  Future<void> registerWithEmail({
    required String email,
    required String password,
    String? displayName,
  });

  Future<void> signInWithGoogle();

  Future<void> sendPasswordReset({required String email});

  Future<void> signOut();

  /// Updates the stored IANA timezone for the current user. No-op if signed out.
  Future<void> updateTimezone(String ianaZone);

  /// Releases any underlying resources (stream controllers, listeners). Safe
  /// to call multiple times.
  Future<void> dispose();
}
```

- [ ] **Step 2: Run analyzer**

```powershell
flutter analyze
```
Expected: zero issues.

- [ ] **Step 3: Commit**

```powershell
git add hbit/lib/data/auth/auth_repository.dart
git commit -m "feat: add AuthRepository abstract interface"
```

---

## Task 2: HabitMapRepository abstract interface

**Files:**
- Create: `hbit/lib/data/habit_map/habit_map_repository.dart`

- [ ] **Step 1: Create the file**

Create `hbit/lib/data/habit_map/habit_map_repository.dart`:

```dart
import '../../models/aspiration.dart';
import '../../models/habit_map.dart';
import '../../models/habit_map_summary.dart';
import '../../models/tiny_habit.dart';

/// Repository contract for the user's habit maps (active + archived).
abstract class HabitMapRepository {
  /// Emits the user's active map, or `null` when no active map exists. Emits
  /// immediately on subscription with the current state.
  Stream<HabitMap?> activeMapStream(String uid);

  /// Creates a new active map for [uid]. Archives any existing active map
  /// transactionally. Returns the newly created map (with its assigned id).
  Future<HabitMap> createMap(
    String uid, {
    required Aspiration aspiration,
    required List<TinyHabit> habits,
  });

  /// Archives the current active map for [uid]. Does nothing if no active map
  /// exists.
  Future<void> archiveActiveMap(String uid);

  /// Emits a list of archived maps, newest first, as lightweight summaries.
  Stream<List<HabitMapSummary>> archivedMapsStream(String uid);

  /// One-off fetch of a single archived map by id.
  Future<HabitMap> getArchivedMap(String uid, String mapId);
}
```

- [ ] **Step 2: Run analyzer**

```powershell
flutter analyze
```
Expected: zero issues.

- [ ] **Step 3: Commit**

```powershell
git add hbit/lib/data/habit_map/habit_map_repository.dart
git commit -m "feat: add HabitMapRepository abstract interface"
```

---

## Task 3: CheckInRepository abstract interface

**Files:**
- Create: `hbit/lib/data/check_in/check_in_repository.dart`

- [ ] **Step 1: Create the file**

Create `hbit/lib/data/check_in/check_in_repository.dart`:

```dart
/// Repository contract for daily habit check-ins. Check-ins are stored
/// day-keyed under a map (`YYYY-MM-DD`) and contain the set of habit ids
/// completed that day.
abstract class CheckInRepository {
  /// Emits the set of habit ids checked-in for "today" under [mapId]. "Today"
  /// is computed using the user's stored timezone (resolved by the
  /// implementation, not by the caller).
  Stream<Set<String>> todayCheckInsStream(String uid, String mapId);

  /// Adds [habitId] to today's set if absent, removes it if present. When the
  /// last habit id is removed, the day document is deleted (no empty docs).
  Future<void> toggleCheckIn(String uid, String mapId, String habitId);

  /// Emits all check-in days for [mapId] in the inclusive range
  /// [start]..[end], keyed by date (with the time part set to midnight in the
  /// user's stored timezone). Used by the calendar in Subsystem #3.
  Stream<Map<DateTime, Set<String>>> checkInsInRangeStream(
    String uid,
    String mapId, {
    required DateTime start,
    required DateTime end,
  });
}
```

- [ ] **Step 2: Run analyzer**

```powershell
flutter analyze
```
Expected: zero issues.

- [ ] **Step 3: Commit**

```powershell
git add hbit/lib/data/check_in/check_in_repository.dart
git commit -m "feat: add CheckInRepository abstract interface"
```

---

## Task 4: Repositories container + RepositoriesScope

**Files:**
- Create: `hbit/lib/data/repositories.dart`
- Create: `hbit/lib/app/repositories_scope.dart`
- Test: `hbit/test/app/repositories_scope_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/app/repositories_scope_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/app/repositories_scope.dart';
import 'package:hbit/data/repositories.dart';

import '../fakes/fake_auth_repository.dart';
import '../fakes/fake_habit_map_repository.dart';
import '../fakes/fake_check_in_repository.dart';

void main() {
  testWidgets('RepositoriesScope.of exposes the configured Repositories',
      (tester) async {
    final repos = Repositories(
      auth: FakeAuthRepository(),
      habitMaps: FakeHabitMapRepository(),
      checkIns: FakeCheckInRepository(),
    );

    Repositories? captured;
    await tester.pumpWidget(
      RepositoriesScope(
        repositories: repos,
        child: Builder(
          builder: (context) {
            captured = RepositoriesScope.of(context);
            return const SizedBox.shrink();
          },
        ),
      ),
    );

    expect(captured, same(repos));
  });

  testWidgets('RepositoriesScope.of throws when no scope is present',
      (tester) async {
    await tester.pumpWidget(
      Builder(
        builder: (context) {
          expect(() => RepositoriesScope.of(context), throwsFlutterError);
          return const SizedBox.shrink();
        },
      ),
    );
  });
}
```

This test pulls in fake repositories that we build in Tasks 5–7. The test will not compile yet; that is expected — we run Step 2 after the fakes exist.

- [ ] **Step 2: Implement the container and scope**

Create `hbit/lib/data/repositories.dart`:

```dart
import 'auth/auth_repository.dart';
import 'check_in/check_in_repository.dart';
import 'habit_map/habit_map_repository.dart';

/// Aggregates the three repositories so they can be passed around as one
/// dependency. Owned by the app shell; disposed when the app shuts down.
class Repositories {
  Repositories({
    required this.auth,
    required this.habitMaps,
    required this.checkIns,
  });

  final AuthRepository auth;
  final HabitMapRepository habitMaps;
  final CheckInRepository checkIns;

  Future<void> dispose() async {
    await auth.dispose();
  }
}
```

Create `hbit/lib/app/repositories_scope.dart`:

```dart
import 'package:flutter/widgets.dart';

import '../data/repositories.dart';

/// Makes a [Repositories] instance available to descendant widgets.
class RepositoriesScope extends InheritedWidget {
  const RepositoriesScope({
    super.key,
    required this.repositories,
    required super.child,
  });

  final Repositories repositories;

  static Repositories of(BuildContext context) {
    final scope =
        context.dependOnInheritedWidgetOfExactType<RepositoriesScope>();
    if (scope == null) {
      throw FlutterError(
        'RepositoriesScope.of() called with a context that does not contain '
        'a RepositoriesScope. Wrap your app in a RepositoriesScope at the '
        'root of the widget tree.',
      );
    }
    return scope.repositories;
  }

  @override
  bool updateShouldNotify(RepositoriesScope oldWidget) =>
      repositories != oldWidget.repositories;
}
```

- [ ] **Step 3: Defer running tests** until fakes exist (Tasks 5–7). Add a temporary skip note here and move on. After Task 7, return and run the suite.

- [ ] **Step 4: Run the analyzer (it should pass because the fakes only matter at test time)**

```powershell
flutter analyze lib/
```
Expected: zero issues for `lib/`. (Test errors are expected until fakes exist.)

- [ ] **Step 5: Commit**

```powershell
git add hbit/lib/data/repositories.dart hbit/lib/app/repositories_scope.dart hbit/test/app/repositories_scope_test.dart
git commit -m "feat: add Repositories container and RepositoriesScope DI widget"
```

---

## Task 5: FakeAuthRepository

**Files:**
- Create: `hbit/test/fakes/fake_auth_repository.dart`
- Test: `hbit/test/fakes/fake_auth_repository_test.dart`

The fake is reused across many tests and benefits from its own correctness check.

- [ ] **Step 1: Write the failing test**

Create `hbit/test/fakes/fake_auth_repository_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/models/app_user.dart';

import 'fake_auth_repository.dart';

void main() {
  AppUser sampleUser({String uid = 'uid-1'}) => AppUser(
        uid: uid,
        email: 'a@b.test',
        timezone: 'Europe/Berlin',
        createdAt: DateTime.utc(2026, 1, 1),
        lastSignInAt: DateTime.utc(2026, 5, 17),
      );

  test('starts signed-out', () {
    final repo = FakeAuthRepository();
    expect(repo.currentUser, isNull);
  });

  test('userStream emits null then signed-in user', () async {
    final repo = FakeAuthRepository();
    final emissions = <AppUser?>[];
    final sub = repo.userStream.listen(emissions.add);

    await Future<void>.delayed(Duration.zero);
    repo.setUser(sampleUser());
    await Future<void>.delayed(Duration.zero);

    expect(emissions, [null, isA<AppUser>()]);
    await sub.cancel();
  });

  test('signOut emits null', () async {
    final repo = FakeAuthRepository(initialUser: sampleUser());
    expect(repo.currentUser, isNotNull);
    await repo.signOut();
    expect(repo.currentUser, isNull);
  });

  test('updateTimezone mutates the current user', () async {
    final repo = FakeAuthRepository(initialUser: sampleUser());
    await repo.updateTimezone('America/Los_Angeles');
    expect(repo.currentUser!.timezone, 'America/Los_Angeles');
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/fakes/fake_auth_repository_test.dart
```
Expected: FAIL — `fake_auth_repository.dart` does not exist.

- [ ] **Step 3: Implement the fake**

Create `hbit/test/fakes/fake_auth_repository.dart`:

```dart
import 'dart:async';

import 'package:hbit/data/auth/auth_repository.dart';
import 'package:hbit/data/auth_exception.dart';
import 'package:hbit/models/app_user.dart';

/// In-memory implementation of [AuthRepository] for tests.
///
/// Tests can drive state via [setUser], [setNextError], etc. The fake does
/// not validate emails or passwords — it just records calls and emits.
class FakeAuthRepository implements AuthRepository {
  FakeAuthRepository({AppUser? initialUser})
      : _current = initialUser {
    _controller = StreamController<AppUser?>.broadcast(
      onListen: () => _controller.add(_current),
    );
  }

  AppUser? _current;
  late final StreamController<AppUser?> _controller;

  AuthException? nextSignInError;
  AuthException? nextRegisterError;
  AuthException? nextGoogleError;
  AuthException? nextResetError;

  /// Recorded calls — for assertions in tests.
  final List<String> recordedCalls = [];

  @override
  Stream<AppUser?> get userStream => _controller.stream;

  @override
  AppUser? get currentUser => _current;

  /// Test affordance: directly set the current user.
  void setUser(AppUser? user) {
    _current = user;
    _controller.add(user);
  }

  @override
  Future<void> signInWithEmail({
    required String email,
    required String password,
  }) async {
    recordedCalls.add('signInWithEmail:$email');
    if (nextSignInError != null) {
      final e = nextSignInError!;
      nextSignInError = null;
      throw e;
    }
    setUser(_userFor(email));
  }

  @override
  Future<void> registerWithEmail({
    required String email,
    required String password,
    String? displayName,
  }) async {
    recordedCalls.add('registerWithEmail:$email');
    if (nextRegisterError != null) {
      final e = nextRegisterError!;
      nextRegisterError = null;
      throw e;
    }
    setUser(_userFor(email, displayName: displayName));
  }

  @override
  Future<void> signInWithGoogle() async {
    recordedCalls.add('signInWithGoogle');
    if (nextGoogleError != null) {
      final e = nextGoogleError!;
      nextGoogleError = null;
      throw e;
    }
    setUser(_userFor('google@example.test', displayName: 'Google User'));
  }

  @override
  Future<void> sendPasswordReset({required String email}) async {
    recordedCalls.add('sendPasswordReset:$email');
    if (nextResetError != null) {
      final e = nextResetError!;
      nextResetError = null;
      throw e;
    }
  }

  @override
  Future<void> signOut() async {
    recordedCalls.add('signOut');
    setUser(null);
  }

  @override
  Future<void> updateTimezone(String ianaZone) async {
    recordedCalls.add('updateTimezone:$ianaZone');
    final u = _current;
    if (u == null) return;
    setUser(u.copyWith(timezone: ianaZone));
  }

  @override
  Future<void> dispose() async {
    await _controller.close();
  }

  AppUser _userFor(String email, {String? displayName}) => AppUser(
        uid: 'fake-${email.hashCode}',
        email: email,
        displayName: displayName,
        timezone: 'Europe/Berlin',
        createdAt: DateTime.utc(2026, 1, 1),
        lastSignInAt: DateTime.utc(2026, 5, 17),
      );
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/fakes/fake_auth_repository_test.dart
```
Expected: PASS, all 4 tests.

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass.

- [ ] **Step 6: Commit**

```powershell
git add hbit/test/fakes/fake_auth_repository.dart hbit/test/fakes/fake_auth_repository_test.dart
git commit -m "test: add FakeAuthRepository"
```

---

## Task 6: FakeHabitMapRepository

**Files:**
- Create: `hbit/test/fakes/fake_habit_map_repository.dart`

The fake is exercised by `RepositoriesScope` and many later widget tests. A small sanity test is bundled here.

- [ ] **Step 1: Write the implementation**

Create `hbit/test/fakes/fake_habit_map_repository.dart`:

```dart
import 'dart:async';

import 'package:hbit/data/habit_map/habit_map_repository.dart';
import 'package:hbit/models/aspiration.dart';
import 'package:hbit/models/habit_map.dart';
import 'package:hbit/models/habit_map_summary.dart';
import 'package:hbit/models/tiny_habit.dart';

class FakeHabitMapRepository implements HabitMapRepository {
  final Map<String, HabitMap?> _active = {};
  final Map<String, List<HabitMap>> _archived = {};
  final Map<String, StreamController<HabitMap?>> _activeControllers = {};
  final Map<String, StreamController<List<HabitMapSummary>>>
      _archivedControllers = {};

  int _nextId = 0;

  void seedActive(String uid, HabitMap map) {
    _active[uid] = map;
    _activeControllers[uid]?.add(map);
  }

  @override
  Stream<HabitMap?> activeMapStream(String uid) {
    final controller = _activeControllers.putIfAbsent(
      uid,
      () => StreamController<HabitMap?>.broadcast(
        onListen: () => _activeControllers[uid]!.add(_active[uid]),
      ),
    );
    return controller.stream;
  }

  @override
  Future<HabitMap> createMap(
    String uid, {
    required Aspiration aspiration,
    required List<TinyHabit> habits,
  }) async {
    final existing = _active[uid];
    if (existing != null) {
      final archived = HabitMap(
        id: existing.id,
        aspiration: existing.aspiration,
        habits: existing.habits,
        status: HabitMapStatus.archived,
        archivedAt: DateTime.now(),
      );
      _archived.putIfAbsent(uid, () => []).insert(0, archived);
      _emitArchived(uid);
    }

    final id = 'fake-map-${_nextId++}';
    final newMap = HabitMap(
      id: id,
      aspiration: aspiration,
      habits: habits,
    );
    _active[uid] = newMap;
    _activeControllers[uid]?.add(newMap);
    return newMap;
  }

  @override
  Future<void> archiveActiveMap(String uid) async {
    final existing = _active[uid];
    if (existing == null) return;
    final archived = HabitMap(
      id: existing.id,
      aspiration: existing.aspiration,
      habits: existing.habits,
      status: HabitMapStatus.archived,
      archivedAt: DateTime.now(),
    );
    _archived.putIfAbsent(uid, () => []).insert(0, archived);
    _active[uid] = null;
    _activeControllers[uid]?.add(null);
    _emitArchived(uid);
  }

  @override
  Stream<List<HabitMapSummary>> archivedMapsStream(String uid) {
    final controller = _archivedControllers.putIfAbsent(
      uid,
      () => StreamController<List<HabitMapSummary>>.broadcast(
        onListen: () => _emitArchived(uid),
      ),
    );
    return controller.stream;
  }

  @override
  Future<HabitMap> getArchivedMap(String uid, String mapId) async {
    final list = _archived[uid] ?? const [];
    return list.firstWhere(
      (m) => m.id == mapId,
      orElse: () => throw StateError('Archived map $mapId not found'),
    );
  }

  void _emitArchived(String uid) {
    final list = _archived[uid] ?? const [];
    final summaries = list
        .map((m) => HabitMapSummary(
              id: m.id ?? '',
              identity: m.aspiration.identity,
              archivedAt: m.archivedAt ?? DateTime.now(),
              habitCount: m.habits.length,
            ))
        .toList();
    _archivedControllers[uid]?.add(summaries);
  }
}
```

- [ ] **Step 2: Run analyzer**

```powershell
flutter analyze
```
Expected: zero issues.

- [ ] **Step 3: Commit**

```powershell
git add hbit/test/fakes/fake_habit_map_repository.dart
git commit -m "test: add FakeHabitMapRepository"
```

---

## Task 7: FakeCheckInRepository

**Files:**
- Create: `hbit/test/fakes/fake_check_in_repository.dart`

- [ ] **Step 1: Write the implementation**

Create `hbit/test/fakes/fake_check_in_repository.dart`:

```dart
import 'dart:async';

import 'package:hbit/data/check_in/check_in_repository.dart';

class FakeCheckInRepository implements CheckInRepository {
  /// Map of `${uid}:${mapId}` → today's habit ids.
  final Map<String, Set<String>> _today = {};
  final Map<String, StreamController<Set<String>>> _todayControllers = {};

  /// Map of `${uid}:${mapId}` → date → habit ids.
  final Map<String, Map<DateTime, Set<String>>> _range = {};
  final Map<String, StreamController<Map<DateTime, Set<String>>>>
      _rangeControllers = {};

  String _key(String uid, String mapId) => '$uid:$mapId';

  void seedToday(String uid, String mapId, Set<String> ids) {
    final k = _key(uid, mapId);
    _today[k] = {...ids};
    _todayControllers[k]?.add({..._today[k]!});
  }

  @override
  Stream<Set<String>> todayCheckInsStream(String uid, String mapId) {
    final k = _key(uid, mapId);
    final controller = _todayControllers.putIfAbsent(
      k,
      () => StreamController<Set<String>>.broadcast(
        onListen: () =>
            _todayControllers[k]!.add({...(_today[k] ?? const {})}),
      ),
    );
    return controller.stream;
  }

  @override
  Future<void> toggleCheckIn(String uid, String mapId, String habitId) async {
    final k = _key(uid, mapId);
    final set = _today.putIfAbsent(k, () => <String>{});
    if (set.contains(habitId)) {
      set.remove(habitId);
    } else {
      set.add(habitId);
    }
    _todayControllers[k]?.add({...set});
  }

  @override
  Stream<Map<DateTime, Set<String>>> checkInsInRangeStream(
    String uid,
    String mapId, {
    required DateTime start,
    required DateTime end,
  }) {
    final k = _key(uid, mapId);
    final controller = _rangeControllers.putIfAbsent(
      k,
      () => StreamController<Map<DateTime, Set<String>>>.broadcast(
        onListen: () => _rangeControllers[k]!
            .add({...(_range[k] ?? const <DateTime, Set<String>>{})}),
      ),
    );
    return controller.stream;
  }
}
```

- [ ] **Step 2: Run the deferred `RepositoriesScope` test from Task 4 now that all fakes exist**

```powershell
flutter test test/app/repositories_scope_test.dart
```
Expected: PASS, both tests.

- [ ] **Step 3: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass.

- [ ] **Step 4: Commit**

```powershell
git add hbit/test/fakes/fake_check_in_repository.dart
git commit -m "test: add FakeCheckInRepository"
```

---

## Task 8: FirebaseAuthRepository

**Files:**
- Create: `hbit/lib/data/auth/firebase_auth_repository.dart`
- Test: `hbit/test/data/auth/firebase_auth_repository_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/data/auth/firebase_auth_repository_test.dart`:

```dart
import 'package:fake_cloud_firestore/fake_cloud_firestore.dart';
import 'package:firebase_auth_mocks/firebase_auth_mocks.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/data/auth/firebase_auth_repository.dart';
import 'package:hbit/data/auth_exception.dart';

void main() {
  late MockFirebaseAuth auth;
  late FakeFirebaseFirestore firestore;
  late FirebaseAuthRepository repo;

  setUp(() {
    auth = MockFirebaseAuth();
    firestore = FakeFirebaseFirestore();
    repo = FirebaseAuthRepository(
      auth: auth,
      firestore: firestore,
      googleSignIn: null, // covered by separate test
      detectLocalTimezone: () async => 'Europe/Berlin',
    );
  });

  tearDown(() => repo.dispose());

  test('signed-out state emits null on userStream', () async {
    final first = await repo.userStream.first;
    expect(first, isNull);
  });

  test('registerWithEmail creates user doc with timezone', () async {
    await repo.registerWithEmail(
      email: 'new@example.test',
      password: 'hunter2hunter2',
      displayName: 'New User',
    );

    final user = repo.currentUser;
    expect(user, isNotNull);
    expect(user!.email, 'new@example.test');
    expect(user.displayName, 'New User');
    expect(user.timezone, 'Europe/Berlin');

    final doc = await firestore.collection('users').doc(user.uid).get();
    expect(doc.exists, isTrue);
    expect(doc.data()!['email'], 'new@example.test');
    expect(doc.data()!['timezone'], 'Europe/Berlin');
  });

  test('signInWithEmail uses existing user doc and updates lastSignInAt',
      () async {
    // Seed an existing account in the mock auth and user doc.
    await auth.createUserWithEmailAndPassword(
      email: 'returning@example.test',
      password: 'hunter2hunter2',
    );
    final uid = auth.currentUser!.uid;
    await firestore.collection('users').doc(uid).set({
      'email': 'returning@example.test',
      'displayName': null,
      'photoUrl': null,
      'timezone': 'America/Los_Angeles',
      'createdAt': DateTime.utc(2025, 12, 1),
      'lastSignInAt': DateTime.utc(2025, 12, 1),
      'activeMapId': null,
    });
    await auth.signOut();

    await repo.signInWithEmail(
      email: 'returning@example.test',
      password: 'hunter2hunter2',
    );

    expect(repo.currentUser, isNotNull);
    expect(repo.currentUser!.timezone, 'America/Los_Angeles');

    final doc = await firestore.collection('users').doc(uid).get();
    final lastSignIn = doc.data()!['lastSignInAt'];
    expect(lastSignIn, isNotNull);
  });

  test('signOut emits null', () async {
    await repo.registerWithEmail(
      email: 'so@example.test',
      password: 'hunter2hunter2',
    );
    expect(repo.currentUser, isNotNull);

    await repo.signOut();
    expect(repo.currentUser, isNull);
  });

  test('updateTimezone persists to user doc', () async {
    await repo.registerWithEmail(
      email: 'tz@example.test',
      password: 'hunter2hunter2',
    );
    final uid = repo.currentUser!.uid;

    await repo.updateTimezone('America/Los_Angeles');

    final doc = await firestore.collection('users').doc(uid).get();
    expect(doc.data()!['timezone'], 'America/Los_Angeles');
    expect(repo.currentUser!.timezone, 'America/Los_Angeles');
  });

  test('wrong-password throws AuthException(wrongPassword)', () async {
    await auth.createUserWithEmailAndPassword(
      email: 'wp@example.test',
      password: 'correct-password',
    );
    await auth.signOut();

    auth.mockUser = null;
    // firebase_auth_mocks doesn't enforce password matching; simulate by
    // configuring the next sign-in to fail.
    auth.whenCalled(MockMethodNames.signInWithEmailAndPassword).thenThrow(
          MockFirebaseAuthException(code: 'wrong-password'),
        );

    expect(
      () => repo.signInWithEmail(
        email: 'wp@example.test',
        password: 'wrong-password',
      ),
      throwsA(isA<AuthException>().having(
        (e) => e.kind,
        'kind',
        AuthErrorKind.wrongPassword,
      )),
    );
  });
}
```

Notes on the test:
- `firebase_auth_mocks` does not validate passwords by default — the wrong-password test uses its `whenCalled` API to inject the canonical Firebase error code.
- `FakeFirebaseFirestore` accepts arbitrary writes; we use it to inspect the user doc side-effects.
- The repository accepts injected dependencies (`auth`, `firestore`, `googleSignIn`, `detectLocalTimezone`) so the test never touches real Firebase.

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/data/auth/firebase_auth_repository_test.dart
```
Expected: FAIL — `firebase_auth_repository.dart` does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/data/auth/firebase_auth_repository.dart`:

```dart
import 'dart:async';

import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart' as fb;
import 'package:google_sign_in/google_sign_in.dart';

import '../../models/app_user.dart';
import '../auth_exception.dart';
import 'auth_repository.dart';

/// Callback returning the device's IANA timezone string. Extracted so tests
/// can stub the value without invoking the `flutter_timezone` platform plugin.
typedef LocalTimezoneDetector = Future<String> Function();

class FirebaseAuthRepository implements AuthRepository {
  FirebaseAuthRepository({
    required fb.FirebaseAuth auth,
    required FirebaseFirestore firestore,
    required GoogleSignIn? googleSignIn,
    required LocalTimezoneDetector detectLocalTimezone,
  })  : _auth = auth,
        _firestore = firestore,
        _googleSignIn = googleSignIn,
        _detectTimezone = detectLocalTimezone {
    _controller = StreamController<AppUser?>.broadcast(onListen: () {
      _controller.add(_currentUser);
    });
    _authSub = _auth.authStateChanges().listen(_onAuthStateChanged);
  }

  final fb.FirebaseAuth _auth;
  final FirebaseFirestore _firestore;
  final GoogleSignIn? _googleSignIn;
  final LocalTimezoneDetector _detectTimezone;

  AppUser? _currentUser;
  late final StreamController<AppUser?> _controller;
  late final StreamSubscription<fb.User?> _authSub;

  @override
  Stream<AppUser?> get userStream => _controller.stream;

  @override
  AppUser? get currentUser => _currentUser;

  Future<void> _onAuthStateChanged(fb.User? user) async {
    if (user == null) {
      _currentUser = null;
      _controller.add(null);
      return;
    }
    final appUser = await _loadOrCreateUserDoc(user);
    _currentUser = appUser;
    _controller.add(appUser);
  }

  Future<AppUser> _loadOrCreateUserDoc(fb.User user) async {
    final docRef = _firestore.collection('users').doc(user.uid);
    final snap = await docRef.get();
    if (snap.exists) {
      await docRef.update({'lastSignInAt': FieldValue.serverTimestamp()});
      final data = snap.data()!;
      return AppUser(
        uid: user.uid,
        email: data['email'] as String? ?? user.email ?? '',
        displayName: data['displayName'] as String?,
        photoUrl: data['photoUrl'] as String?,
        timezone: data['timezone'] as String? ?? 'UTC',
        createdAt: _timestampOr(data['createdAt'], DateTime.now()),
        lastSignInAt: DateTime.now(),
      );
    }
    final tz = await _detectTimezone();
    final now = DateTime.now();
    final newDoc = <String, dynamic>{
      'email': user.email,
      'displayName': user.displayName,
      'photoUrl': user.photoURL,
      'timezone': tz,
      'createdAt': FieldValue.serverTimestamp(),
      'lastSignInAt': FieldValue.serverTimestamp(),
      'activeMapId': null,
    };
    await docRef.set(newDoc);
    return AppUser(
      uid: user.uid,
      email: user.email ?? '',
      displayName: user.displayName,
      photoUrl: user.photoURL,
      timezone: tz,
      createdAt: now,
      lastSignInAt: now,
    );
  }

  DateTime _timestampOr(Object? raw, DateTime fallback) {
    if (raw is Timestamp) return raw.toDate();
    if (raw is DateTime) return raw;
    return fallback;
  }

  @override
  Future<void> signInWithEmail({
    required String email,
    required String password,
  }) async {
    try {
      await _auth.signInWithEmailAndPassword(email: email, password: password);
    } on fb.FirebaseAuthException catch (e) {
      throw _translate(e);
    }
  }

  @override
  Future<void> registerWithEmail({
    required String email,
    required String password,
    String? displayName,
  }) async {
    try {
      final result = await _auth.createUserWithEmailAndPassword(
        email: email,
        password: password,
      );
      if (displayName != null && displayName.isNotEmpty) {
        await result.user!.updateDisplayName(displayName);
      }
    } on fb.FirebaseAuthException catch (e) {
      throw _translate(e);
    }
  }

  @override
  Future<void> signInWithGoogle() async {
    final google = _googleSignIn;
    if (google == null) {
      throw const AuthException(
        AuthErrorKind.unknown,
        'Google Sign-In is not configured.',
      );
    }
    try {
      final account = await google.signIn();
      if (account == null) {
        throw const AuthException(AuthErrorKind.cancelled);
      }
      final googleAuth = await account.authentication;
      final credential = fb.GoogleAuthProvider.credential(
        accessToken: googleAuth.accessToken,
        idToken: googleAuth.idToken,
      );
      await _auth.signInWithCredential(credential);
    } on fb.FirebaseAuthException catch (e) {
      throw _translate(e);
    }
  }

  @override
  Future<void> sendPasswordReset({required String email}) async {
    try {
      await _auth.sendPasswordResetEmail(email: email);
    } on fb.FirebaseAuthException catch (e) {
      throw _translate(e);
    }
  }

  @override
  Future<void> signOut() async {
    await _googleSignIn?.signOut();
    await _auth.signOut();
  }

  @override
  Future<void> updateTimezone(String ianaZone) async {
    final user = _currentUser;
    if (user == null) return;
    await _firestore
        .collection('users')
        .doc(user.uid)
        .update({'timezone': ianaZone});
    _currentUser = user.copyWith(timezone: ianaZone);
    _controller.add(_currentUser);
  }

  @override
  Future<void> dispose() async {
    await _authSub.cancel();
    await _controller.close();
  }

  AuthException _translate(fb.FirebaseAuthException e) {
    switch (e.code) {
      case 'wrong-password':
      case 'invalid-credential':
        return AuthException(AuthErrorKind.wrongPassword, e.message);
      case 'email-already-in-use':
        return AuthException(AuthErrorKind.emailInUse, e.message);
      case 'invalid-email':
        return AuthException(AuthErrorKind.invalidEmail, e.message);
      case 'weak-password':
        return AuthException(AuthErrorKind.weakPassword, e.message);
      case 'user-not-found':
        return AuthException(AuthErrorKind.userNotFound, e.message);
      case 'network-request-failed':
        return AuthException(AuthErrorKind.networkUnavailable, e.message);
      default:
        return AuthException(AuthErrorKind.unknown, e.message);
    }
  }
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/data/auth/firebase_auth_repository_test.dart
```
Expected: PASS, all 6 tests. (If `MockFirebaseAuthException` / `whenCalled` API names differ in the pinned version of `firebase_auth_mocks`, replace with the equivalent in that version — read the package CHANGELOG. The structural behavior is the same.)

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass; zero issues.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/data/auth/firebase_auth_repository.dart hbit/test/data/auth/firebase_auth_repository_test.dart
git commit -m "feat: implement FirebaseAuthRepository with translation tests"
```

---

## Task 9: FirestoreHabitMapRepository

**Files:**
- Create: `hbit/lib/data/habit_map/firestore_habit_map_repository.dart`
- Test: `hbit/test/data/habit_map/firestore_habit_map_repository_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/data/habit_map/firestore_habit_map_repository_test.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:fake_cloud_firestore/fake_cloud_firestore.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/data/habit_map/firestore_habit_map_repository.dart';
import 'package:hbit/models/aspiration.dart';
import 'package:hbit/models/habit_map.dart';
import 'package:hbit/models/tiny_habit.dart';

void main() {
  late FakeFirebaseFirestore firestore;
  late FirestoreHabitMapRepository repo;

  Aspiration anyAspiration() => Aspiration(
        identity: 'a senior engineer',
        timeframe: '3 years',
        motivation: 'build great products',
        createdAt: DateTime.utc(2026, 5, 17),
      );

  TinyHabit habit(String id) => TinyHabit(
        id: id,
        domain: 'engineering',
        domainLabel: 'Engineering',
        behavior: 'Daily review',
        anchorPrompt: 'After I sit at my desk',
        tinyAction: 'I open one PR',
        celebration: 'I say nice',
        impact: 4,
        ease: 4,
      );

  setUp(() async {
    firestore = FakeFirebaseFirestore();
    await firestore.collection('users').doc('uid-1').set({
      'email': 'a@b.test',
      'timezone': 'Europe/Berlin',
      'activeMapId': null,
    });
    repo = FirestoreHabitMapRepository(firestore: firestore);
  });

  test('createMap writes the map doc and sets activeMapId', () async {
    final map = await repo.createMap(
      'uid-1',
      aspiration: anyAspiration(),
      habits: [habit('h1'), habit('h2')],
    );

    expect(map.id, isNotNull);
    expect(map.status, HabitMapStatus.active);

    final mapDoc = await firestore
        .collection('users')
        .doc('uid-1')
        .collection('maps')
        .doc(map.id)
        .get();
    expect(mapDoc.exists, isTrue);
    expect(mapDoc.data()!['status'], 'active');
    expect((mapDoc.data()!['habits'] as List).length, 2);

    final userDoc = await firestore.collection('users').doc('uid-1').get();
    expect(userDoc.data()!['activeMapId'], map.id);
  });

  test('createMap archives previous active map atomically', () async {
    final first = await repo.createMap(
      'uid-1',
      aspiration: anyAspiration(),
      habits: [habit('h1')],
    );
    final second = await repo.createMap(
      'uid-1',
      aspiration: anyAspiration(),
      habits: [habit('h2')],
    );

    final firstDoc = await firestore
        .collection('users')
        .doc('uid-1')
        .collection('maps')
        .doc(first.id)
        .get();
    expect(firstDoc.data()!['status'], 'archived');
    expect(firstDoc.data()!['archivedAt'], isNotNull);

    final userDoc = await firestore.collection('users').doc('uid-1').get();
    expect(userDoc.data()!['activeMapId'], second.id);
  });

  test('activeMapStream emits null then map then null after archive',
      () async {
    final emissions = <HabitMap?>[];
    final sub = repo.activeMapStream('uid-1').listen(emissions.add);
    await Future<void>.delayed(const Duration(milliseconds: 50));

    final created = await repo.createMap(
      'uid-1',
      aspiration: anyAspiration(),
      habits: [habit('h1')],
    );
    await Future<void>.delayed(const Duration(milliseconds: 50));

    await repo.archiveActiveMap('uid-1');
    await Future<void>.delayed(const Duration(milliseconds: 50));

    expect(emissions.first, isNull);
    expect(emissions.any((m) => m?.id == created.id), isTrue);
    expect(emissions.last, isNull);
    await sub.cancel();
  });

  test('archiveActiveMap is a no-op when no active map exists', () async {
    await repo.archiveActiveMap('uid-1');
    final userDoc = await firestore.collection('users').doc('uid-1').get();
    expect(userDoc.data()!['activeMapId'], isNull);
  });

  test('archivedMapsStream emits newest-first summaries', () async {
    final a = await repo.createMap(
      'uid-1',
      aspiration: anyAspiration(),
      habits: [habit('h1')],
    );
    await Future<void>.delayed(const Duration(milliseconds: 10));
    final b = await repo.createMap(
      'uid-1',
      aspiration: anyAspiration(),
      habits: [habit('h2')],
    );
    await repo.archiveActiveMap('uid-1');

    final list = await repo.archivedMapsStream('uid-1').first;
    expect(list.map((s) => s.id), [b.id, a.id]);
  });

  test('getArchivedMap returns full map', () async {
    final map = await repo.createMap(
      'uid-1',
      aspiration: anyAspiration(),
      habits: [habit('h1')],
    );
    await repo.archiveActiveMap('uid-1');

    final fetched = await repo.getArchivedMap('uid-1', map.id!);
    expect(fetched.id, map.id);
    expect(fetched.status, HabitMapStatus.archived);
    expect(fetched.habits, hasLength(1));
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/data/habit_map/firestore_habit_map_repository_test.dart
```
Expected: FAIL — implementation file does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/data/habit_map/firestore_habit_map_repository.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

import '../../models/aspiration.dart';
import '../../models/habit_map.dart';
import '../../models/habit_map_summary.dart';
import '../../models/tiny_habit.dart';
import 'habit_map_repository.dart';

class FirestoreHabitMapRepository implements HabitMapRepository {
  FirestoreHabitMapRepository({required FirebaseFirestore firestore})
      : _firestore = firestore;

  final FirebaseFirestore _firestore;

  DocumentReference<Map<String, dynamic>> _userDoc(String uid) =>
      _firestore.collection('users').doc(uid);

  CollectionReference<Map<String, dynamic>> _mapsCol(String uid) =>
      _userDoc(uid).collection('maps');

  @override
  Stream<HabitMap?> activeMapStream(String uid) {
    return _userDoc(uid).snapshots().asyncMap((userSnap) async {
      final activeId = userSnap.data()?['activeMapId'] as String?;
      if (activeId == null) return null;
      final mapSnap = await _mapsCol(uid).doc(activeId).get();
      if (!mapSnap.exists) return null;
      return _decodeMap(mapSnap.id, mapSnap.data()!);
    });
  }

  @override
  Future<HabitMap> createMap(
    String uid, {
    required Aspiration aspiration,
    required List<TinyHabit> habits,
  }) async {
    final newMapRef = _mapsCol(uid).doc();
    final now = DateTime.now();

    await _firestore.runTransaction((tx) async {
      final userSnap = await tx.get(_userDoc(uid));
      final previousActiveId = userSnap.data()?['activeMapId'] as String?;

      if (previousActiveId != null) {
        tx.update(_mapsCol(uid).doc(previousActiveId), {
          'status': 'archived',
          'archivedAt': FieldValue.serverTimestamp(),
        });
      }

      tx.set(newMapRef, _encodeMap(
        id: newMapRef.id,
        aspiration: aspiration,
        habits: habits,
        status: HabitMapStatus.active,
        archivedAt: null,
      ));

      tx.update(_userDoc(uid), {'activeMapId': newMapRef.id});
    });

    return HabitMap(
      id: newMapRef.id,
      aspiration: aspiration,
      habits: habits,
      status: HabitMapStatus.active,
      archivedAt: null,
    );
  }

  @override
  Future<void> archiveActiveMap(String uid) async {
    await _firestore.runTransaction((tx) async {
      final userSnap = await tx.get(_userDoc(uid));
      final activeId = userSnap.data()?['activeMapId'] as String?;
      if (activeId == null) return;
      tx.update(_mapsCol(uid).doc(activeId), {
        'status': 'archived',
        'archivedAt': FieldValue.serverTimestamp(),
      });
      tx.update(_userDoc(uid), {'activeMapId': null});
    });
  }

  @override
  Stream<List<HabitMapSummary>> archivedMapsStream(String uid) {
    return _mapsCol(uid)
        .where('status', isEqualTo: 'archived')
        .orderBy('archivedAt', descending: true)
        .snapshots()
        .map((qs) => qs.docs
            .map((d) => HabitMapSummary(
                  id: d.id,
                  identity:
                      (d.data()['aspiration'] as Map<String, dynamic>?)?[
                              'identity'] as String? ??
                          '',
                  archivedAt:
                      (d.data()['archivedAt'] as Timestamp?)?.toDate() ??
                          DateTime.fromMillisecondsSinceEpoch(0),
                  habitCount: (d.data()['habits'] as List?)?.length ?? 0,
                ))
            .toList());
  }

  @override
  Future<HabitMap> getArchivedMap(String uid, String mapId) async {
    final snap = await _mapsCol(uid).doc(mapId).get();
    if (!snap.exists) {
      throw StateError('Archived map $mapId not found for $uid');
    }
    return _decodeMap(snap.id, snap.data()!);
  }

  Map<String, dynamic> _encodeMap({
    required String id,
    required Aspiration aspiration,
    required List<TinyHabit> habits,
    required HabitMapStatus status,
    required DateTime? archivedAt,
  }) =>
      {
        'id': id,
        'aspiration': {
          'identity': aspiration.identity,
          'timeframe': aspiration.timeframe,
          'motivation': aspiration.motivation,
          'createdAt': Timestamp.fromDate(aspiration.createdAt),
        },
        'habits': habits
            .map((h) => {
                  'id': h.id,
                  'domain': h.domain,
                  'domainLabel': h.domainLabel,
                  'behavior': h.behavior,
                  'anchorPrompt': h.anchorPrompt,
                  'tinyAction': h.tinyAction,
                  'celebration': h.celebration,
                  'impact': h.impact,
                  'ease': h.ease,
                })
            .toList(),
        'status': status.name,
        'createdAt': FieldValue.serverTimestamp(),
        if (archivedAt != null) 'archivedAt': Timestamp.fromDate(archivedAt),
      };

  HabitMap _decodeMap(String id, Map<String, dynamic> data) {
    final aspirationData = data['aspiration'] as Map<String, dynamic>;
    final aspiration = Aspiration(
      identity: aspirationData['identity'] as String,
      timeframe: aspirationData['timeframe'] as String,
      motivation: aspirationData['motivation'] as String,
      createdAt:
          (aspirationData['createdAt'] as Timestamp?)?.toDate() ??
              DateTime.now(),
    );
    final habits = ((data['habits'] as List?) ?? const [])
        .map((raw) {
          final h = raw as Map<String, dynamic>;
          return TinyHabit(
            id: h['id'] as String,
            domain: h['domain'] as String,
            domainLabel: h['domainLabel'] as String,
            behavior: h['behavior'] as String,
            anchorPrompt: h['anchorPrompt'] as String,
            tinyAction: h['tinyAction'] as String,
            celebration: h['celebration'] as String,
            impact: h['impact'] as int,
            ease: h['ease'] as int,
          );
        })
        .toList();

    final status = (data['status'] as String?) == 'archived'
        ? HabitMapStatus.archived
        : HabitMapStatus.active;

    return HabitMap(
      id: id,
      aspiration: aspiration,
      habits: habits,
      status: status,
      archivedAt: (data['archivedAt'] as Timestamp?)?.toDate(),
    );
  }
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/data/habit_map/firestore_habit_map_repository_test.dart
```
Expected: PASS, all 6 tests.

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/data/habit_map/firestore_habit_map_repository.dart hbit/test/data/habit_map/firestore_habit_map_repository_test.dart
git commit -m "feat: implement FirestoreHabitMapRepository with transactional archive"
```

---

## Task 10: FirestoreCheckInRepository

**Files:**
- Create: `hbit/lib/data/check_in/firestore_check_in_repository.dart`
- Test: `hbit/test/data/check_in/firestore_check_in_repository_test.dart`

The repository needs the user's timezone to compute "today". Tests inject a fixed timezone and a fixed `now`.

- [ ] **Step 1: Write the failing test**

Create `hbit/test/data/check_in/firestore_check_in_repository_test.dart`:

```dart
import 'package:fake_cloud_firestore/fake_cloud_firestore.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/data/check_in/firestore_check_in_repository.dart';
import 'package:hbit/services/timezone_service.dart';
import 'package:timezone/data/latest.dart' as tzdata;

void main() {
  setUpAll(tzdata.initializeTimeZones);

  late FakeFirebaseFirestore firestore;
  late FirestoreCheckInRepository repo;

  setUp(() async {
    firestore = FakeFirebaseFirestore();
    await firestore.collection('users').doc('uid-1').set({
      'timezone': 'Europe/Berlin',
    });
    repo = FirestoreCheckInRepository(
      firestore: firestore,
      timezoneService: TimezoneService(),
      now: () => DateTime.utc(2026, 5, 17, 12, 0), // 14:00 Berlin (CEST)
    );
  });

  test('toggleCheckIn adds habit id to today', () async {
    await repo.toggleCheckIn('uid-1', 'map-1', 'h1');

    final doc = await firestore
        .collection('users')
        .doc('uid-1')
        .collection('maps')
        .doc('map-1')
        .collection('checkIns')
        .doc('2026-05-17')
        .get();

    expect(doc.exists, isTrue);
    expect((doc.data()!['habitIds'] as List), contains('h1'));
  });

  test('toggleCheckIn removes existing habit id', () async {
    await repo.toggleCheckIn('uid-1', 'map-1', 'h1');
    await repo.toggleCheckIn('uid-1', 'map-1', 'h2');
    await repo.toggleCheckIn('uid-1', 'map-1', 'h1');

    final doc = await firestore
        .collection('users')
        .doc('uid-1')
        .collection('maps')
        .doc('map-1')
        .collection('checkIns')
        .doc('2026-05-17')
        .get();

    expect((doc.data()!['habitIds'] as List), ['h2']);
  });

  test('toggleCheckIn deletes the day doc when set becomes empty', () async {
    await repo.toggleCheckIn('uid-1', 'map-1', 'h1');
    await repo.toggleCheckIn('uid-1', 'map-1', 'h1');

    final doc = await firestore
        .collection('users')
        .doc('uid-1')
        .collection('maps')
        .doc('map-1')
        .collection('checkIns')
        .doc('2026-05-17')
        .get();

    expect(doc.exists, isFalse);
  });

  test('todayCheckInsStream reflects toggles', () async {
    final emissions = <Set<String>>[];
    final sub =
        repo.todayCheckInsStream('uid-1', 'map-1').listen(emissions.add);

    await Future<void>.delayed(const Duration(milliseconds: 50));
    await repo.toggleCheckIn('uid-1', 'map-1', 'h1');
    await Future<void>.delayed(const Duration(milliseconds: 50));
    await repo.toggleCheckIn('uid-1', 'map-1', 'h2');
    await Future<void>.delayed(const Duration(milliseconds: 50));

    expect(emissions.first, isEmpty);
    expect(emissions.last, containsAll(<String>{'h1', 'h2'}));
    await sub.cancel();
  });

  test('checkInsInRangeStream returns past days in range', () async {
    await firestore
        .collection('users')
        .doc('uid-1')
        .collection('maps')
        .doc('map-1')
        .collection('checkIns')
        .doc('2026-05-15')
        .set({
      'date': '2026-05-15',
      'habitIds': ['h1', 'h2'],
    });

    final result = await repo
        .checkInsInRangeStream(
          'uid-1',
          'map-1',
          start: DateTime.utc(2026, 5, 14),
          end: DateTime.utc(2026, 5, 17),
        )
        .first;

    expect(result.keys, hasLength(1));
    final entry = result.entries.single;
    expect(entry.value, containsAll(<String>{'h1', 'h2'}));
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/data/check_in/firestore_check_in_repository_test.dart
```
Expected: FAIL — implementation file does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/data/check_in/firestore_check_in_repository.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

import '../../services/timezone_service.dart';
import 'check_in_repository.dart';

typedef NowProvider = DateTime Function();

class FirestoreCheckInRepository implements CheckInRepository {
  FirestoreCheckInRepository({
    required FirebaseFirestore firestore,
    required TimezoneService timezoneService,
    NowProvider? now,
  })  : _firestore = firestore,
        _timezones = timezoneService,
        _now = now ?? DateTime.now;

  final FirebaseFirestore _firestore;
  final TimezoneService _timezones;
  final NowProvider _now;

  Future<String> _userTimezone(String uid) async {
    final snap = await _firestore.collection('users').doc(uid).get();
    return (snap.data()?['timezone'] as String?) ?? 'UTC';
  }

  CollectionReference<Map<String, dynamic>> _checkInsCol(
    String uid,
    String mapId,
  ) =>
      _firestore
          .collection('users')
          .doc(uid)
          .collection('maps')
          .doc(mapId)
          .collection('checkIns');

  @override
  Stream<Set<String>> todayCheckInsStream(String uid, String mapId) async* {
    final tz = await _userTimezone(uid);
    final dayKey = _timezones.todayKey(timezone: tz, now: _now());
    yield* _checkInsCol(uid, mapId).doc(dayKey).snapshots().map((snap) {
      if (!snap.exists) return <String>{};
      final ids = (snap.data()!['habitIds'] as List?) ?? const [];
      return ids.cast<String>().toSet();
    });
  }

  @override
  Future<void> toggleCheckIn(String uid, String mapId, String habitId) async {
    final tz = await _userTimezone(uid);
    final dayKey = _timezones.todayKey(timezone: tz, now: _now());
    final ref = _checkInsCol(uid, mapId).doc(dayKey);

    await _firestore.runTransaction((tx) async {
      final snap = await tx.get(ref);
      final currentIds = snap.exists
          ? ((snap.data()!['habitIds'] as List).cast<String>().toSet())
          : <String>{};
      if (currentIds.contains(habitId)) {
        currentIds.remove(habitId);
      } else {
        currentIds.add(habitId);
      }

      if (currentIds.isEmpty) {
        if (snap.exists) tx.delete(ref);
        return;
      }

      tx.set(ref, {
        'date': dayKey,
        'habitIds': currentIds.toList()..sort(),
        'updatedAt': FieldValue.serverTimestamp(),
      });
    });
  }

  @override
  Stream<Map<DateTime, Set<String>>> checkInsInRangeStream(
    String uid,
    String mapId, {
    required DateTime start,
    required DateTime end,
  }) async* {
    final tz = await _userTimezone(uid);
    final startKey = _timezones
        .todayKey(timezone: tz, now: start)
        .toString();
    final endKey = _timezones.todayKey(timezone: tz, now: end).toString();

    yield* _checkInsCol(uid, mapId)
        .where(FieldPath.documentId, isGreaterThanOrEqualTo: startKey)
        .where(FieldPath.documentId, isLessThanOrEqualTo: endKey)
        .orderBy(FieldPath.documentId)
        .snapshots()
        .map((qs) {
      final result = <DateTime, Set<String>>{};
      for (final doc in qs.docs) {
        final ids =
            ((doc.data()['habitIds'] as List?) ?? const []).cast<String>();
        final date = _timezones.parseDayKey(doc.id, timezone: tz);
        result[date] = ids.toSet();
      }
      return result;
    });
  }
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/data/check_in/firestore_check_in_repository_test.dart
```
Expected: PASS, all 5 tests.

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter test
flutter analyze
```
Expected: all pass; zero issues.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/data/check_in/firestore_check_in_repository.dart hbit/test/data/check_in/firestore_check_in_repository_test.dart
git commit -m "feat: implement FirestoreCheckInRepository with day-keyed toggles"
```

---

## Sprint 2 — Done definition

- [ ] `flutter test` from `hbit/` passes (all original + Sprint 1 + Sprint 2 tests)
- [ ] `flutter analyze` from `hbit/` reports zero issues
- [ ] No file in `lib/screens/` imports anything from `lib/data/auth/firebase_*`, `lib/data/habit_map/firestore_*`, or `lib/data/check_in/firestore_*` (the screens still use `HabitStorage`; repos are not yet wired in)
- [ ] All 10 tasks committed in sequence
- [ ] Tag the sprint completion: `git tag sprint-2-repositories`

Next: [Sprint 3 — Auth screens](2026-05-17-auth-cloud-storage-sprint-3-auth-screens.md).
