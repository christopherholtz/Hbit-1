# Sprint 5 — History, Security Rules, Setup Docs

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** [`../specs/2026-05-17-auth-cloud-storage-design.md`](../specs/2026-05-17-auth-cloud-storage-design.md)
**Index:** [`2026-05-17-auth-cloud-storage-INDEX.md`](2026-05-17-auth-cloud-storage-INDEX.md)
**Prerequisites:** Sprints 1, 2, 3, and 4 complete.

**Goal:** Surface the archived-map history to the user, deploy and test Firestore Security Rules, and document the Firebase setup so a fresh clone of the repo can be run end-to-end without tribal knowledge.

**Architecture:** Two new read-only screens (`MapHistoryScreen`, `ArchivedMapViewScreen`) consume `archivedMapsStream` and `getArchivedMap` from the existing `HabitMapRepository`. A separate Node-based test harness exercises the deployed Firestore Security Rules against an in-memory rules emulator. The README documents the full Firebase project setup.

**Tech Stack:** Same as previous sprints. Additionally, `firebase-tools` and `@firebase/rules-unit-testing` (Node).

---

## File structure

| Path | Role |
|---|---|
| `lib/screens/map_history_screen.dart` | NEW — list of archived maps |
| `lib/screens/archived_map_view_screen.dart` | NEW — read-only single archived map |
| `lib/screens/profile_screen.dart` | MODIFY — add "Map history" link |
| `lib/main.dart` | MODIFY — register `/history` and `/history/view` routes |
| `test/screens/map_history_screen_test.dart` | NEW |
| `test/screens/archived_map_view_screen_test.dart` | NEW |
| `firestore.rules` | NEW — top-level (repo root) |
| `firestore-tests/package.json` | NEW |
| `firestore-tests/rules.test.js` | NEW |
| `firestore-tests/.gitignore` | NEW |
| `firebase.json` | NEW — top-level, points the Firebase CLI at `firestore.rules` |
| `README.md` | MODIFY — add Firebase setup section |

---

## Task 1: MapHistoryScreen

**Files:**
- Create: `hbit/lib/screens/map_history_screen.dart`
- Test: `hbit/test/screens/map_history_screen_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/screens/map_history_screen_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/app/repositories_scope.dart';
import 'package:hbit/data/repositories.dart';
import 'package:hbit/models/app_user.dart';
import 'package:hbit/models/aspiration.dart';
import 'package:hbit/models/habit_map.dart';
import 'package:hbit/models/tiny_habit.dart';
import 'package:hbit/screens/map_history_screen.dart';

import '../fakes/fake_auth_repository.dart';
import '../fakes/fake_check_in_repository.dart';
import '../fakes/fake_habit_map_repository.dart';

AppUser sampleUser() => AppUser(
      uid: 'uid-1',
      email: 'a@b.test',
      timezone: 'Europe/Berlin',
      createdAt: DateTime.utc(2026, 1, 1),
      lastSignInAt: DateTime.utc(2026, 5, 17),
    );

TinyHabit anyHabit() => TinyHabit(
      id: 'h1',
      domain: 'engineering',
      domainLabel: 'Engineering',
      behavior: 'Daily review',
      anchorPrompt: 'After I sit',
      tinyAction: 'I read one PR',
      celebration: 'I say nice',
      impact: 4,
      ease: 4,
    );

Widget _harness(Repositories repos) => MaterialApp(
      home: RepositoriesScope(
        repositories: repos,
        child: const MapHistoryScreen(),
      ),
    );

void main() {
  late FakeHabitMapRepository habitMaps;
  late Repositories repos;

  setUp(() {
    habitMaps = FakeHabitMapRepository();
    repos = Repositories(
      auth: FakeAuthRepository(initialUser: sampleUser()),
      habitMaps: habitMaps,
      checkIns: FakeCheckInRepository(),
    );
  });

  testWidgets('shows empty state when no archived maps', (tester) async {
    await tester.pumpWidget(_harness(repos));
    await tester.pumpAndSettle();

    expect(find.text('No archived maps yet.'), findsOneWidget);
  });

  testWidgets('lists archived maps newest first', (tester) async {
    // Seed two archived maps via the create+archive flow on the fake.
    await habitMaps.createMap(
      'uid-1',
      aspiration: Aspiration(
        identity: 'a writer',
        timeframe: '2 years',
        motivation: 'I want to publish',
        createdAt: DateTime.utc(2026, 1, 1),
      ),
      habits: [anyHabit()],
    );
    await habitMaps.createMap(
      'uid-1',
      aspiration: Aspiration(
        identity: 'a leader',
        timeframe: '3 years',
        motivation: 'I want to manage',
        createdAt: DateTime.utc(2026, 2, 1),
      ),
      habits: [anyHabit()],
    );
    await habitMaps.archiveActiveMap('uid-1');

    await tester.pumpWidget(_harness(repos));
    await tester.pumpAndSettle();

    expect(find.text('a leader'), findsWidgets);
    expect(find.text('a writer'), findsWidgets);
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/screens/map_history_screen_test.dart
```
Expected: FAIL — `map_history_screen.dart` does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/screens/map_history_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:intl/intl.dart';

import '../app/repositories_scope.dart';
import '../models/habit_map_summary.dart';
import 'archived_map_view_screen.dart';

class MapHistoryScreen extends StatelessWidget {
  const MapHistoryScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final repos = RepositoriesScope.of(context);
    final user = repos.auth.currentUser;
    if (user == null) {
      return const Scaffold(body: Center(child: Text('Signed out.')));
    }

    return Scaffold(
      appBar: AppBar(title: const Text('Map history')),
      body: SafeArea(
        child: StreamBuilder<List<HabitMapSummary>>(
          stream: repos.habitMaps.archivedMapsStream(user.uid),
          builder: (context, snapshot) {
            if (snapshot.connectionState == ConnectionState.waiting) {
              return const Center(child: CircularProgressIndicator());
            }
            final maps = snapshot.data ?? const [];
            if (maps.isEmpty) {
              return Center(
                child: Text(
                  'No archived maps yet.',
                  style: theme.textTheme.bodyLarge,
                ),
              );
            }
            return ListView.separated(
              padding: const EdgeInsets.all(16),
              itemCount: maps.length,
              separatorBuilder: (_, __) => const SizedBox(height: 8),
              itemBuilder: (context, i) {
                final s = maps[i];
                return Card(
                  child: ListTile(
                    title: Text(
                      s.identity,
                      style: theme.textTheme.titleMedium
                          ?.copyWith(fontWeight: FontWeight.bold),
                    ),
                    subtitle: Text(
                      '${s.habitCount} habits · archived '
                      '${DateFormat.yMMMd().format(s.archivedAt)}',
                    ),
                    trailing: const Icon(Icons.chevron_right),
                    onTap: () => Navigator.of(context).push(
                      MaterialPageRoute(
                        builder: (_) =>
                            ArchivedMapViewScreen(mapId: s.id),
                      ),
                    ),
                  ),
                );
              },
            );
          },
        ),
      ),
    );
  }
}
```

The test still requires `ArchivedMapViewScreen` to compile. Create a placeholder before running:

Create `hbit/lib/screens/archived_map_view_screen.dart`:

```dart
import 'package:flutter/material.dart';

class ArchivedMapViewScreen extends StatelessWidget {
  const ArchivedMapViewScreen({super.key, required this.mapId});

  final String mapId;

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Archived map')),
        body: Center(child: Text('TODO Task 2: render $mapId')),
      );
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/screens/map_history_screen_test.dart
```
Expected: PASS, both tests.

- [ ] **Step 5: Run full test suite + analyzer**

```powershell
flutter analyze
flutter test
```
Expected: all pass.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/screens/map_history_screen.dart hbit/lib/screens/archived_map_view_screen.dart hbit/test/screens/map_history_screen_test.dart
git commit -m "feat: add MapHistoryScreen and ArchivedMapViewScreen placeholder"
```

---

## Task 2: ArchivedMapViewScreen

**Files:**
- Modify: `hbit/lib/screens/archived_map_view_screen.dart` (replace placeholder)
- Test: `hbit/test/screens/archived_map_view_screen_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/screens/archived_map_view_screen_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/app/repositories_scope.dart';
import 'package:hbit/data/repositories.dart';
import 'package:hbit/models/app_user.dart';
import 'package:hbit/models/aspiration.dart';
import 'package:hbit/models/tiny_habit.dart';
import 'package:hbit/screens/archived_map_view_screen.dart';

import '../fakes/fake_auth_repository.dart';
import '../fakes/fake_check_in_repository.dart';
import '../fakes/fake_habit_map_repository.dart';

AppUser sampleUser() => AppUser(
      uid: 'uid-1',
      email: 'a@b.test',
      timezone: 'Europe/Berlin',
      createdAt: DateTime.utc(2026, 1, 1),
      lastSignInAt: DateTime.utc(2026, 5, 17),
    );

void main() {
  testWidgets('renders the archived map identity and habits',
      (tester) async {
    final habitMaps = FakeHabitMapRepository();
    await habitMaps.createMap(
      'uid-1',
      aspiration: Aspiration(
        identity: 'a senior writer',
        timeframe: '2 years',
        motivation: 'I want to publish a book',
        createdAt: DateTime.utc(2026, 1, 1),
      ),
      habits: [
        TinyHabit(
          id: 'h1',
          domain: 'writing',
          domainLabel: 'Writing Craft',
          behavior: 'Daily writing',
          anchorPrompt: 'After my morning coffee',
          tinyAction: 'I write one sentence',
          celebration: 'I smile',
          impact: 5,
          ease: 4,
        ),
      ],
    );
    await habitMaps.archiveActiveMap('uid-1');
    final summaries = await habitMaps.archivedMapsStream('uid-1').first;
    final mapId = summaries.first.id;

    final repos = Repositories(
      auth: FakeAuthRepository(initialUser: sampleUser()),
      habitMaps: habitMaps,
      checkIns: FakeCheckInRepository(),
    );

    await tester.pumpWidget(MaterialApp(
      home: RepositoriesScope(
        repositories: repos,
        child: ArchivedMapViewScreen(mapId: mapId),
      ),
    ));
    await tester.pumpAndSettle();

    expect(find.text('a senior writer'), findsOneWidget);
    expect(find.text('Daily writing'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/screens/archived_map_view_screen_test.dart
```
Expected: FAIL — the placeholder renders only "TODO" text.

- [ ] **Step 3: Replace the placeholder**

Replace `hbit/lib/screens/archived_map_view_screen.dart`:

```dart
import 'package:flutter/material.dart';

import '../app/repositories_scope.dart';
import '../models/habit_map.dart';

class ArchivedMapViewScreen extends StatelessWidget {
  const ArchivedMapViewScreen({super.key, required this.mapId});

  final String mapId;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final repos = RepositoriesScope.of(context);
    final user = repos.auth.currentUser;
    if (user == null) {
      return const Scaffold(body: Center(child: Text('Signed out.')));
    }

    return Scaffold(
      appBar: AppBar(title: const Text('Archived map')),
      body: SafeArea(
        child: FutureBuilder<HabitMap>(
          future: repos.habitMaps.getArchivedMap(user.uid, mapId),
          builder: (context, snapshot) {
            if (snapshot.connectionState != ConnectionState.done) {
              return const Center(child: CircularProgressIndicator());
            }
            if (snapshot.hasError) {
              return Center(child: Text('Error: ${snapshot.error}'));
            }
            final map = snapshot.data!;
            return ListView(
              padding: const EdgeInsets.all(20),
              children: [
                Text(
                  'I was becoming',
                  style: theme.textTheme.labelSmall
                      ?.copyWith(letterSpacing: 1.2),
                ),
                const SizedBox(height: 4),
                Text(
                  map.aspiration.identity,
                  style: theme.textTheme.headlineSmall
                      ?.copyWith(fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 6),
                Text('"${map.aspiration.motivation}"',
                    style: theme.textTheme.bodyMedium
                        ?.copyWith(fontStyle: FontStyle.italic)),
                const Divider(height: 32),
                for (final entry in map.habitsByDomain.entries) ...[
                  Text(
                    entry.key.toUpperCase(),
                    style: theme.textTheme.labelLarge?.copyWith(
                      fontWeight: FontWeight.bold,
                      color: theme.colorScheme.primary,
                    ),
                  ),
                  const SizedBox(height: 8),
                  for (final habit in entry.value) ...[
                    Padding(
                      padding: const EdgeInsets.only(bottom: 12),
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Text(habit.behavior,
                              style: theme.textTheme.titleSmall
                                  ?.copyWith(fontWeight: FontWeight.bold)),
                          const SizedBox(height: 4),
                          Text(habit.recipe,
                              style: theme.textTheme.bodyMedium),
                        ],
                      ),
                    ),
                  ],
                  const SizedBox(height: 12),
                ],
              ],
            );
          },
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Wire `ProfileScreen` to link to map history**

In `hbit/lib/screens/profile_screen.dart`, add a "Map history" tile above the "Sign out" button. After the timezone dropdown padding block and before `const SizedBox(height: 32)`, insert:

```dart
ListTile(
  contentPadding: EdgeInsets.zero,
  title: const Text('Map history'),
  trailing: const Icon(Icons.chevron_right),
  onTap: () => Navigator.of(context).pushNamed('/history'),
),
const Divider(),
```

- [ ] **Step 5: Register the `/history` route in main.dart**

Update the `routes:` map in `hbit/lib/main.dart`:

```dart
routes: {
  '/profile': (_) => const ProfileScreen(),
  '/history': (_) => const MapHistoryScreen(),
},
```

Add the import:

```dart
import 'screens/map_history_screen.dart';
```

- [ ] **Step 6: Run all tests**

```powershell
flutter analyze
flutter test
```
Expected: all pass; zero issues.

- [ ] **Step 7: Commit**

```powershell
git add hbit/lib/screens/archived_map_view_screen.dart hbit/lib/screens/profile_screen.dart hbit/lib/main.dart hbit/test/screens/archived_map_view_screen_test.dart
git commit -m "feat: implement ArchivedMapViewScreen and wire from ProfileScreen"
```

---

## Task 3: Firestore Security Rules + Node test harness

**Files:**
- Create: `firestore.rules` (repo root)
- Create: `firebase.json` (repo root)
- Create: `firestore-tests/package.json`
- Create: `firestore-tests/rules.test.js`
- Create: `firestore-tests/.gitignore`

- [ ] **Step 1: Create `firestore.rules` at the repo root**

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

        allow delete: if false;

        match /checkIns/{date} {
          allow read: if request.auth != null && request.auth.uid == uid;

          allow create, update: if request.auth != null
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
      return next.status in ['active', 'archived']
          && next.id == prev.id
          && next.aspiration == prev.aspiration
          && next.habits == prev.habits;
    }

    function isValidCheckIn(d, date) {
      return d.date == date
          && d.habitIds is list
          && d.habitIds.size() > 0;
    }
  }
}
```

- [ ] **Step 2: Create `firebase.json` at the repo root**

```json
{
  "firestore": {
    "rules": "firestore.rules"
  }
}
```

- [ ] **Step 3: Create `firestore-tests/package.json`**

```json
{
  "name": "hbit-firestore-tests",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "node --experimental-vm-modules node_modules/jest/bin/jest.js"
  },
  "devDependencies": {
    "@firebase/rules-unit-testing": "^3.0.4",
    "firebase-admin": "^12.0.0",
    "jest": "^29.7.0"
  }
}
```

- [ ] **Step 4: Create `firestore-tests/.gitignore`**

```
node_modules/
coverage/
```

- [ ] **Step 5: Create `firestore-tests/rules.test.js`**

```javascript
import {
  initializeTestEnvironment,
  assertSucceeds,
  assertFails,
} from "@firebase/rules-unit-testing";
import { readFileSync } from "node:fs";
import { resolve, dirname } from "node:path";
import { fileURLToPath } from "node:url";

const __dirname = dirname(fileURLToPath(import.meta.url));

let env;

beforeAll(async () => {
  env = await initializeTestEnvironment({
    projectId: "hbit-test",
    firestore: {
      rules: readFileSync(resolve(__dirname, "../firestore.rules"), "utf8"),
      host: "127.0.0.1",
      port: 8080,
    },
  });
});

afterAll(async () => {
  await env.cleanup();
});

beforeEach(async () => {
  await env.clearFirestore();
});

function authedDb(uid) {
  return env.authenticatedContext(uid).firestore();
}

function unauthedDb() {
  return env.unauthenticatedContext().firestore();
}

const validMap = {
  id: "m1",
  status: "active",
  createdAt: new Date(),
  aspiration: {
    identity: "an engineer",
    timeframe: "3 years",
    motivation: "build things",
    createdAt: new Date(),
  },
  habits: [
    {
      id: "h1",
      domain: "engineering",
      domainLabel: "Engineering",
      behavior: "Review PRs",
      anchorPrompt: "After I sit down",
      tinyAction: "I open one PR",
      celebration: "I smile",
      impact: 4,
      ease: 4,
    },
  ],
};

const validCheckIn = (date) => ({
  date,
  habitIds: ["h1"],
});

describe("Firestore Security Rules", () => {
  test("unauthenticated user cannot read users/{uid}", async () => {
    const db = unauthedDb();
    await assertFails(db.doc("users/alice").get());
  });

  test("user A cannot read user B's profile", async () => {
    const db = authedDb("alice");
    await assertFails(db.doc("users/bob").get());
  });

  test("user A can read and write own profile", async () => {
    const db = authedDb("alice");
    await assertSucceeds(
      db.doc("users/alice").set({ email: "a@b.test" })
    );
    await assertSucceeds(db.doc("users/alice").get());
  });

  test("user A cannot read user B's maps", async () => {
    const db = authedDb("alice");
    await assertFails(db.doc("users/bob/maps/m1").get());
  });

  test("create with valid map succeeds", async () => {
    const db = authedDb("alice");
    await assertSucceeds(
      db.doc("users/alice/maps/m1").set(validMap)
    );
  });

  test("create with status outside enum is rejected", async () => {
    const db = authedDb("alice");
    await assertFails(
      db.doc("users/alice/maps/m1").set({ ...validMap, status: "WAT" })
    );
  });

  test("create with too many habits is rejected", async () => {
    const db = authedDb("alice");
    const bad = {
      ...validMap,
      habits: Array.from({ length: 31 }, (_, i) => ({
        ...validMap.habits[0],
        id: `h${i}`,
      })),
    };
    await assertFails(db.doc("users/alice/maps/m1").set(bad));
  });

  test("delete on map is rejected", async () => {
    const db = authedDb("alice");
    await assertSucceeds(db.doc("users/alice/maps/m1").set(validMap));
    await assertFails(db.doc("users/alice/maps/m1").delete());
  });

  test("check-in with empty habitIds is rejected", async () => {
    const db = authedDb("alice");
    await assertSucceeds(db.doc("users/alice/maps/m1").set(validMap));
    await assertFails(
      db
        .doc("users/alice/maps/m1/checkIns/2026-05-17")
        .set({ ...validCheckIn("2026-05-17"), habitIds: [] })
    );
  });

  test("valid check-in succeeds", async () => {
    const db = authedDb("alice");
    await assertSucceeds(db.doc("users/alice/maps/m1").set(validMap));
    await assertSucceeds(
      db
        .doc("users/alice/maps/m1/checkIns/2026-05-17")
        .set(validCheckIn("2026-05-17"))
    );
  });

  test("check-in date field must match doc id", async () => {
    const db = authedDb("alice");
    await assertSucceeds(db.doc("users/alice/maps/m1").set(validMap));
    await assertFails(
      db
        .doc("users/alice/maps/m1/checkIns/2026-05-17")
        .set({ date: "2026-05-18", habitIds: ["h1"] })
    );
  });
});
```

- [ ] **Step 6: Install and verify the Firebase CLI emulators**

Run from repo root:

```powershell
npm install -g firebase-tools
firebase --version
```
Expected: CLI version printed. (One-time setup; safe to skip if already installed.)

- [ ] **Step 7: Install Node dependencies in `firestore-tests/`**

Run from repo root:

```powershell
cd firestore-tests
npm install
cd ..
```
Expected: `node_modules/` populated.

- [ ] **Step 8: Start the Firestore emulator in a separate terminal**

```powershell
firebase emulators:start --only firestore
```
Expected: emulator runs at `127.0.0.1:8080`. Keep this terminal open.

- [ ] **Step 9: Run the rules tests**

In a second terminal, from repo root:

```powershell
cd firestore-tests
npm test
```
Expected: all 11 tests pass.

- [ ] **Step 10: Deploy the rules to the live Firebase project**

```powershell
firebase deploy --only firestore:rules
```
Expected: rules upload succeeds.

- [ ] **Step 11: Commit**

```powershell
git add firestore.rules firebase.json firestore-tests/package.json firestore-tests/package-lock.json firestore-tests/rules.test.js firestore-tests/.gitignore
git commit -m "feat: add Firestore Security Rules with Node test harness"
```

---

## Task 4: README setup instructions

**Files:**
- Modify: `README.md`
- Modify: `hbit/pubspec.yaml` (update `description`)

- [ ] **Step 1: Update the pubspec description**

In `hbit/pubspec.yaml`, change line 2:

```yaml
description: "Hbit — a Tiny Habits career habit map, backed by Firebase."
```

- [ ] **Step 2: Replace the README**

Replace `README.md` at repo root:

```markdown
# Hbit

A Flutter app that turns a career aspiration into a curated set of BJ Fogg-style
tiny habits and tracks daily check-ins. Cloud-backed via Firebase Auth +
Cloud Firestore.

## Project layout

- `hbit/` — Flutter application
- `firestore.rules` — Firestore Security Rules
- `firebase.json` — Firebase CLI configuration
- `firestore-tests/` — Node-based test harness for the security rules
- `docs/superpowers/specs/` — design specifications
- `docs/superpowers/plans/` — implementation plans

## Prerequisites

- Flutter SDK 3.10 or newer
- Dart 3.10 or newer
- Node.js 18 or newer (for the security-rules test harness)
- Firebase CLI: `npm install -g firebase-tools`
- FlutterFire CLI: `dart pub global activate flutterfire_cli`
- A Google account with access to the Firebase Console

## First-time setup

1. **Create a Firebase project**
   Go to https://console.firebase.google.com and create a new project named
   something like `hbit-dev`.

2. **Enable authentication providers**
   In the Firebase Console, open Authentication → Sign-in method and enable:
   - Email/Password
   - Google

3. **Configure the Flutter app**
   From the repo root:
   ```bash
   cd hbit
   flutterfire configure
   ```
   Pick the Firebase project from step 1 and confirm the target platforms.
   This generates `hbit/lib/firebase_options.dart`.

4. **Place platform credential files**
   - Android: download `google-services.json` from the Firebase Console
     (Project Settings → General → Your apps → Android) and place it at
     `hbit/android/app/google-services.json`.
   - iOS: download `GoogleService-Info.plist` and place it at
     `hbit/ios/Runner/GoogleService-Info.plist`. Open `hbit/ios/Runner/Info.plist`
     and ensure the Google Sign-In reversed-client-id URL scheme is registered.

5. **Install Flutter dependencies**
   ```bash
   cd hbit
   flutter pub get
   ```

6. **Deploy Firestore Security Rules**
   From the repo root:
   ```bash
   firebase login
   firebase use --add        # pick the Firebase project
   firebase deploy --only firestore:rules
   ```

## Running the app

```bash
cd hbit
flutter run
```

The app launches into the sign-in screen. Create an account, enter an
aspiration, and you're in.

## Running tests

### Flutter tests
```bash
cd hbit
flutter test
flutter analyze
```

### Firestore Security Rules tests
The rules tests require the Firestore emulator. In one terminal:

```bash
firebase emulators:start --only firestore
```

In a second terminal:

```bash
cd firestore-tests
npm install      # first run only
npm test
```

## Architecture

See `docs/superpowers/specs/2026-05-17-auth-cloud-storage-design.md` for the
full subsystem #1 specification.

Key conventions:
- Screens depend on abstract `AuthRepository` / `HabitMapRepository` /
  `CheckInRepository`, never on Firebase SDKs directly.
- Firestore-backed implementations live in `hbit/lib/data/<topic>/firebase_*.dart`.
- All check-ins are day-keyed under `users/{uid}/maps/{mapId}/checkIns/{YYYY-MM-DD}`
  using the user's stored IANA timezone.
- Each habit map has a single active status; previous maps are archived
  (not deleted) and viewable in Map History.

## Roadmap

- Subsystem #1: Auth + Cloud Storage (this sprint set)
- Subsystem #2: RAG/LLM-driven habit engine
- Subsystem #3: Gamification (streaks, calendar, scores)
- Subsystem #4: Onboarding redesign + visual refresh

## License

Private.
```

- [ ] **Step 3: Run analyzer and tests one last time**

```powershell
cd hbit
flutter analyze
flutter test
cd ..
```
Expected: all pass; zero issues.

- [ ] **Step 4: Commit**

```powershell
git add README.md hbit/pubspec.yaml
git commit -m "docs: document Firebase setup, testing, and architecture"
```

---

## Sprint 5 — Done definition

- [ ] `flutter test` from `hbit/` passes (Sprints 1–5 cumulative)
- [ ] `flutter analyze` from `hbit/` reports zero issues
- [ ] `cd firestore-tests && npm test` passes (with the Firestore emulator running)
- [ ] `firebase deploy --only firestore:rules` succeeds against the Firebase project
- [ ] App flow: sign in → generate map → toggle habits → archive → view in `MapHistoryScreen` → tap to open `ArchivedMapViewScreen` (read-only)
- [ ] `README.md` documents the full setup from a fresh clone
- [ ] All 4 tasks committed in sequence
- [ ] Tag the sprint and subsystem completion:
  ```powershell
  git tag sprint-5-history-security
  git tag subsystem-1-auth-cloud-storage-complete
  ```

---

## Subsystem #1 — Wrap-up

At the end of Sprint 5, the foundation for the rest of the roadmap is in place:

- ✅ Auth + cloud storage (this subsystem)
- ⏭ Next: Subsystem #2 — RAG/LLM-driven habit engine (the Cloud Functions backend can now be added with confidence, because Firestore + Auth are already wired in and the data model is stable)
- ⏭ Then: Subsystem #3 — Gamification (the day-keyed check-in collection is already in place and ready for streak/calendar features)
- ⏭ Then: Subsystem #4 — Onboarding redesign + visual refresh
