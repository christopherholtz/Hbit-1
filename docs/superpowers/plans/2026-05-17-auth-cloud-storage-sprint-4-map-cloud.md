# Sprint 4 — Cloud-Backed Map and Check-Ins

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** [`../specs/2026-05-17-auth-cloud-storage-design.md`](../specs/2026-05-17-auth-cloud-storage-design.md)
**Index:** [`2026-05-17-auth-cloud-storage-INDEX.md`](2026-05-17-auth-cloud-storage-INDEX.md)
**Prerequisites:** Sprints 1, 2, and 3 complete.

**Goal:** Replace the local `HabitStorage` flow with Firestore-backed maps and per-day check-ins. After this sprint, all habit data is in the cloud, syncs across devices, and the per-habit `done` boolean is removed from the data model. A new `ProfileScreen` exposes sign-out and timezone selection.

**Architecture:** `AspirationScreen` calls `habitMapRepository.createMap`. `HabitMapScreen` consumes `activeMapStream` and `todayCheckInsStream`, calling `toggleCheckIn` on tap. A new `HomeRoot` replaces `PostAuthRoot` and routes between the two screens based on the active-map stream. `TinyHabit.done` is removed; "done today" is derived from the check-in set.

**Tech Stack:** Same as previous sprints. `shared_preferences` is removed at the end of the sprint.

---

## File structure

| Path | Role |
|---|---|
| `lib/app/home_root.dart` | NEW — replaces `PostAuthRoot`; routes based on `activeMapStream` |
| `lib/app/post_auth_root.dart` | DELETE |
| `lib/screens/aspiration_screen.dart` | MODIFY — submit calls `createMap` |
| `lib/screens/habit_map_screen.dart` | MODIFY — reads streams; toggles call `toggleCheckIn`; archive calls `archiveActiveMap` |
| `lib/screens/profile_screen.dart` | NEW |
| `lib/models/tiny_habit.dart` | MODIFY — remove `done` field |
| `lib/models/habit_map.dart` | MODIFY — remove `doneCount` (no longer meaningful at the model level) |
| `lib/services/habit_storage.dart` | DELETE |
| `lib/app/auth_gate.dart` | MODIFY — show `HomeRoot` instead of `PostAuthRoot` |
| `lib/main.dart` | unchanged (still wires `AuthGate`) |
| `test/widget_test.dart` | MODIFY — remove the `done` round-trip assertion |
| `pubspec.yaml` | MODIFY — drop `shared_preferences` |

---

## Task 1: Refactor AspirationScreen to use HabitMapRepository

The screen no longer takes a `onGenerated` callback. It reads the current user from `AuthRepository` and calls `habitMapRepository.createMap`. After success, the parent stream-based router will switch automatically to `HabitMapScreen`.

**Files:**
- Modify: `hbit/lib/screens/aspiration_screen.dart`
- Test: `hbit/test/screens/aspiration_screen_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/screens/aspiration_screen_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/app/repositories_scope.dart';
import 'package:hbit/data/repositories.dart';
import 'package:hbit/models/app_user.dart';
import 'package:hbit/screens/aspiration_screen.dart';

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

Widget _harness(Repositories repos) => MaterialApp(
      home: RepositoriesScope(
        repositories: repos,
        child: const AspirationScreen(),
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

  testWidgets('submitting valid form calls createMap', (tester) async {
    await tester.pumpWidget(_harness(repos));

    await tester.enterText(
      find.widgetWithText(TextFormField, 'e.g. a senior engineering leader'),
      'a senior engineering leader',
    );
    await tester.enterText(
      find.widgetWithText(
          TextFormField,
          'The deeper reason behind the goal. '
          'Be specific — it helps us pick the right habits.'),
      'I want to lead and build great products',
    );

    await tester.tap(find.text('Generate my habit map'));
    await tester.pump();
    await tester.pump(const Duration(milliseconds: 100));

    final activeMap = await habitMaps.activeMapStream('uid-1').first;
    expect(activeMap, isNotNull);
    expect(activeMap!.aspiration.identity, 'a senior engineering leader');
    expect(activeMap.habits, isNotEmpty);
  });

  testWidgets('blocks submit when identity is empty', (tester) async {
    await tester.pumpWidget(_harness(repos));

    await tester.tap(find.text('Generate my habit map'));
    await tester.pump();

    expect(
      find.text("Tell us the role or identity you're aiming for"),
      findsOneWidget,
    );
    final active = await habitMaps.activeMapStream('uid-1').first;
    expect(active, isNull);
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/screens/aspiration_screen_test.dart
```
Expected: FAIL — `AspirationScreen` still requires `onGenerated`.

- [ ] **Step 3: Rewrite AspirationScreen**

Replace `hbit/lib/screens/aspiration_screen.dart`:

```dart
import 'package:flutter/material.dart';

import '../app/repositories_scope.dart';
import '../engine/habit_engine.dart';
import '../models/aspiration.dart';

class AspirationScreen extends StatefulWidget {
  const AspirationScreen({super.key});

  @override
  State<AspirationScreen> createState() => _AspirationScreenState();
}

class _AspirationScreenState extends State<AspirationScreen> {
  final _formKey = GlobalKey<FormState>();
  final _identityController = TextEditingController();
  final _timeframeController = TextEditingController(text: '3 years');
  final _motivationController = TextEditingController();

  bool _submitting = false;

  @override
  void dispose() {
    _identityController.dispose();
    _timeframeController.dispose();
    _motivationController.dispose();
    super.dispose();
  }

  Future<void> _generate() async {
    if (!_formKey.currentState!.validate()) return;

    final repos = RepositoriesScope.of(context);
    final user = repos.auth.currentUser;
    if (user == null) return;

    final aspiration = Aspiration(
      identity: _identityController.text.trim(),
      timeframe: _timeframeController.text.trim().isEmpty
          ? '3 years'
          : _timeframeController.text.trim(),
      motivation: _motivationController.text.trim(),
      createdAt: DateTime.now(),
    );

    setState(() => _submitting = true);
    try {
      final habits = HabitEngine.generateHabits(aspiration);
      await repos.habitMaps.createMap(
        user.uid,
        aspiration: aspiration,
        habits: habits,
      );
      // The active-map stream will drive HomeRoot to switch screens.
    } finally {
      if (mounted) setState(() => _submitting = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Your Aspiration'),
        backgroundColor: theme.colorScheme.inversePrimary,
      ),
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(20),
          child: Form(
            key: _formKey,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                Text(
                  'Build your career habit map',
                  style: theme.textTheme.headlineSmall
                      ?.copyWith(fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 8),
                Text(
                  'Based on BJ Fogg\'s Tiny Habits. Tell us your ultimate '
                  'goal and we\'ll break it into tiny, anchored habits you '
                  'can actually keep.',
                  style: theme.textTheme.bodyMedium
                      ?.copyWith(color: theme.colorScheme.onSurfaceVariant),
                ),
                const SizedBox(height: 28),
                _FieldLabel('What do you want to become?'),
                TextFormField(
                  controller: _identityController,
                  textCapitalization: TextCapitalization.sentences,
                  decoration: const InputDecoration(
                    hintText: 'e.g. a senior engineering leader',
                    border: OutlineInputBorder(),
                  ),
                  validator: (value) =>
                      (value == null || value.trim().isEmpty)
                          ? "Tell us the role or identity you're aiming for"
                          : null,
                ),
                const SizedBox(height: 20),
                _FieldLabel('By when?'),
                TextFormField(
                  controller: _timeframeController,
                  decoration: const InputDecoration(
                    hintText: 'e.g. 3 years, 18 months',
                    border: OutlineInputBorder(),
                  ),
                ),
                const SizedBox(height: 20),
                _FieldLabel('Why does this matter to you?'),
                TextFormField(
                  controller: _motivationController,
                  textCapitalization: TextCapitalization.sentences,
                  minLines: 3,
                  maxLines: 5,
                  decoration: const InputDecoration(
                    hintText: 'The deeper reason behind the goal. '
                        'Be specific — it helps us pick the right habits.',
                    border: OutlineInputBorder(),
                  ),
                  validator: (value) =>
                      (value == null || value.trim().isEmpty)
                          ? 'Your motivation anchors the whole map'
                          : null,
                ),
                const SizedBox(height: 32),
                FilledButton.icon(
                  onPressed: _submitting ? null : _generate,
                  icon: const Icon(Icons.auto_awesome),
                  style: FilledButton.styleFrom(
                    padding: const EdgeInsets.symmetric(vertical: 16),
                  ),
                  label: _submitting
                      ? const Text('Generating...')
                      : const Text('Generate my habit map'),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}

class _FieldLabel extends StatelessWidget {
  const _FieldLabel(this.text);

  final String text;

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.only(bottom: 8),
      child: Text(
        text,
        style: Theme.of(context)
            .textTheme
            .titleSmall
            ?.copyWith(fontWeight: FontWeight.w600),
      ),
    );
  }
}
```

- [ ] **Step 4: The change above breaks `PostAuthRoot` (which passes `onGenerated`). Open `lib/app/post_auth_root.dart` and temporarily mark it as deprecated**

Replace the body of `PostAuthRoot._PostAuthRootState.build()` with:

```dart
@override
Widget build(BuildContext context) {
  return const Scaffold(
    body: Center(
      child: Text('Replaced by HomeRoot in Sprint 4 Task 4. '
          'This widget is unreachable.'),
    ),
  );
}
```

Remove all imports and fields that are no longer referenced. The file becomes:

```dart
import 'package:flutter/material.dart';

/// Deprecated in Sprint 4. Will be deleted in Task 7 once [AuthGate] points
/// to [HomeRoot]. Kept temporarily so `AuthGate` still compiles.
class PostAuthRoot extends StatelessWidget {
  const PostAuthRoot({super.key});

  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: Center(
        child: Text('Replaced by HomeRoot in Sprint 4. Unreachable.'),
      ),
    );
  }
}
```

- [ ] **Step 5: Run the test to confirm it passes**

```powershell
flutter test test/screens/aspiration_screen_test.dart
```
Expected: PASS, both tests.

- [ ] **Step 6: Run full test suite + analyzer**

```powershell
flutter analyze
flutter test
```
The existing `auth_gate_test` checks for `'Build your career habit map'` after sign-in — it will fail because `PostAuthRoot` is now a placeholder. Expected fail; we fix it in Task 4 when `HomeRoot` is introduced.

- [ ] **Step 7: Mark the broken test as skipped temporarily**

Edit `hbit/test/app/auth_gate_test.dart`. For the test cases `'shows PostAuthRoot when signed in'` and `'navigates from SignIn to PostAuthRoot after sign-in'`, add `skip: 'rewired in Sprint 4 Task 4'` as the last argument to `testWidgets`. Example:

```dart
testWidgets('shows PostAuthRoot when signed in', (tester) async {
  // ... unchanged body ...
}, skip: 'rewired in Sprint 4 Task 4');
```

- [ ] **Step 8: Re-run tests**

```powershell
flutter test
```
Expected: all unskipped tests pass.

- [ ] **Step 9: Commit**

```powershell
git add hbit/lib/screens/aspiration_screen.dart hbit/lib/app/post_auth_root.dart hbit/test/screens/aspiration_screen_test.dart hbit/test/app/auth_gate_test.dart
git commit -m "refactor: AspirationScreen calls HabitMapRepository.createMap"
```

---

## Task 2: Refactor HabitMapScreen to use streams

`HabitMapScreen` now takes a `HabitMap` (loaded from the stream by `HomeRoot`) and reads `todayCheckInsStream` for the per-habit done state. Toggling a habit calls `toggleCheckIn`. The "New map" action calls `archiveActiveMap`.

**Files:**
- Modify: `hbit/lib/screens/habit_map_screen.dart`
- Test: `hbit/test/screens/habit_map_screen_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/screens/habit_map_screen_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/app/repositories_scope.dart';
import 'package:hbit/data/repositories.dart';
import 'package:hbit/models/app_user.dart';
import 'package:hbit/models/aspiration.dart';
import 'package:hbit/models/habit_map.dart';
import 'package:hbit/models/tiny_habit.dart';
import 'package:hbit/screens/habit_map_screen.dart';

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

HabitMap sampleMap() => HabitMap(
      id: 'map-1',
      aspiration: Aspiration(
        identity: 'a senior engineer',
        timeframe: '3 years',
        motivation: 'build great products',
        createdAt: DateTime.utc(2026, 5, 17),
      ),
      habits: [
        TinyHabit(
          id: 'h1',
          domain: 'engineering',
          domainLabel: 'Engineering',
          behavior: 'Daily review',
          anchorPrompt: 'After I sit at my desk',
          tinyAction: 'I read one PR',
          celebration: 'I say nice',
          impact: 4,
          ease: 4,
        ),
      ],
    );

Widget _harness({
  required Repositories repos,
  required HabitMap map,
}) =>
    MaterialApp(
      home: RepositoriesScope(
        repositories: repos,
        child: HabitMapScreen(habitMap: map),
      ),
    );

void main() {
  late FakeCheckInRepository checkIns;
  late FakeHabitMapRepository habitMaps;
  late Repositories repos;

  setUp(() {
    checkIns = FakeCheckInRepository();
    habitMaps = FakeHabitMapRepository();
    repos = Repositories(
      auth: FakeAuthRepository(initialUser: sampleUser()),
      habitMaps: habitMaps,
      checkIns: checkIns,
    );
  });

  testWidgets('renders habits from the provided map', (tester) async {
    await tester.pumpWidget(_harness(repos: repos, map: sampleMap()));
    await tester.pump();
    expect(find.text('Daily review'), findsOneWidget);
  });

  testWidgets('tapping a habit toggles the check-in', (tester) async {
    await tester.pumpWidget(_harness(repos: repos, map: sampleMap()));
    await tester.pump();

    await tester.tap(find.byKey(const Key('habitCard.h1')));
    await tester.pump();
    await tester.pump(const Duration(milliseconds: 50));

    final today = await checkIns.todayCheckInsStream('uid-1', 'map-1').first;
    expect(today, contains('h1'));
  });

  testWidgets('archive confirms and calls archiveActiveMap', (tester) async {
    await tester.pumpWidget(_harness(repos: repos, map: sampleMap()));
    await tester.pump();

    // Seed an active map so archive has something to archive.
    habitMaps.seedActive('uid-1', sampleMap());

    await tester.tap(find.byTooltip('New map'));
    await tester.pumpAndSettle();
    await tester.tap(find.text('New map'));
    await tester.pumpAndSettle();

    final active = await habitMaps.activeMapStream('uid-1').first;
    expect(active, isNull);
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/screens/habit_map_screen_test.dart
```
Expected: FAIL — constructor of `HabitMapScreen` no longer matches; key `habitCard.h1` does not exist.

- [ ] **Step 3: Rewrite HabitMapScreen**

Replace `hbit/lib/screens/habit_map_screen.dart`:

```dart
import 'package:flutter/material.dart';

import '../app/repositories_scope.dart';
import '../models/habit_map.dart';
import '../models/tiny_habit.dart';

class HabitMapScreen extends StatelessWidget {
  const HabitMapScreen({super.key, required this.habitMap});

  final HabitMap habitMap;

  Future<void> _confirmArchive(BuildContext context) async {
    final confirmed = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Start a new map?'),
        content: const Text(
          'This archives your current aspiration and habits, then lets you '
          'enter a new goal. Your check-in history is preserved in Map '
          'History.',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Cancel'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('New map'),
          ),
        ],
      ),
    );
    if (confirmed != true) return;
    if (!context.mounted) return;
    final repos = RepositoriesScope.of(context);
    final user = repos.auth.currentUser;
    if (user == null) return;
    await repos.habitMaps.archiveActiveMap(user.uid);
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final repos = RepositoriesScope.of(context);
    final user = repos.auth.currentUser;
    final mapId = habitMap.id;

    if (user == null || mapId == null) {
      return const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }

    final grouped = habitMap.habitsByDomain;

    return Scaffold(
      appBar: AppBar(
        title: const Text('My Habit Map'),
        backgroundColor: theme.colorScheme.inversePrimary,
        actions: [
          IconButton(
            tooltip: 'Profile',
            icon: const Icon(Icons.account_circle_outlined),
            onPressed: () =>
                Navigator.of(context).pushNamed('/profile'),
          ),
          IconButton(
            tooltip: 'New map',
            icon: const Icon(Icons.refresh),
            onPressed: () => _confirmArchive(context),
          ),
        ],
      ),
      body: SafeArea(
        child: StreamBuilder<Set<String>>(
          stream:
              repos.checkIns.todayCheckInsStream(user.uid, mapId),
          builder: (context, snapshot) {
            final doneIds = snapshot.data ?? const <String>{};
            return ListView(
              padding: const EdgeInsets.all(16),
              children: [
                _AspirationCard(habitMap: habitMap, doneIds: doneIds),
                const SizedBox(height: 20),
                Text(
                  'Your tiny habits',
                  style: theme.textTheme.titleLarge
                      ?.copyWith(fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 4),
                Text(
                  'Each habit is anchored to a routine you already have. Do '
                  'the tiny version, then celebrate immediately.',
                  style: theme.textTheme.bodySmall?.copyWith(
                    color: theme.colorScheme.onSurfaceVariant,
                  ),
                ),
                const SizedBox(height: 12),
                for (final entry in grouped.entries) ...[
                  _DomainHeader(label: entry.key, count: entry.value.length),
                  for (final habit in entry.value)
                    _HabitCard(
                      habit: habit,
                      done: doneIds.contains(habit.id),
                      onToggle: () => repos.checkIns
                          .toggleCheckIn(user.uid, mapId, habit.id),
                    ),
                  const SizedBox(height: 8),
                ],
              ],
            );
          },
        ),
      ),
    );
  }
}

class _AspirationCard extends StatelessWidget {
  const _AspirationCard({required this.habitMap, required this.doneIds});

  final HabitMap habitMap;
  final Set<String> doneIds;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final aspiration = habitMap.aspiration;
    final total = habitMap.habits.length;
    final done = habitMap.habits.where((h) => doneIds.contains(h.id)).length;

    return Card(
      elevation: 0,
      color: theme.colorScheme.primaryContainer,
      child: Padding(
        padding: const EdgeInsets.all(18),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'I AM BECOMING',
              style: theme.textTheme.labelSmall?.copyWith(
                color: theme.colorScheme.onPrimaryContainer,
                letterSpacing: 1.2,
              ),
            ),
            const SizedBox(height: 6),
            Text(
              aspiration.identity,
              style: theme.textTheme.headlineSmall?.copyWith(
                fontWeight: FontWeight.bold,
                color: theme.colorScheme.onPrimaryContainer,
              ),
            ),
            const SizedBox(height: 10),
            Row(
              children: [
                Icon(Icons.flag_outlined,
                    size: 16,
                    color: theme.colorScheme.onPrimaryContainer),
                const SizedBox(width: 6),
                Text(
                  'Horizon: ${aspiration.timeframe}',
                  style: theme.textTheme.bodyMedium?.copyWith(
                    color: theme.colorScheme.onPrimaryContainer,
                  ),
                ),
              ],
            ),
            const SizedBox(height: 6),
            Text(
              '"${aspiration.motivation}"',
              style: theme.textTheme.bodyMedium?.copyWith(
                fontStyle: FontStyle.italic,
                color: theme.colorScheme.onPrimaryContainer,
              ),
            ),
            const SizedBox(height: 14),
            ClipRRect(
              borderRadius: BorderRadius.circular(8),
              child: LinearProgressIndicator(
                value: total == 0 ? 0 : done / total,
                minHeight: 8,
                backgroundColor: theme.colorScheme.surface.withValues(
                  alpha: 0.4,
                ),
              ),
            ),
            const SizedBox(height: 6),
            Text(
              '$done of $total habits done today',
              style: theme.textTheme.bodySmall?.copyWith(
                color: theme.colorScheme.onPrimaryContainer,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _DomainHeader extends StatelessWidget {
  const _DomainHeader({required this.label, required this.count});

  final String label;
  final int count;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Padding(
      padding: const EdgeInsets.only(top: 12, bottom: 6),
      child: Row(
        children: [
          Text(
            label.toUpperCase(),
            style: theme.textTheme.labelLarge?.copyWith(
              fontWeight: FontWeight.bold,
              color: theme.colorScheme.primary,
              letterSpacing: 0.8,
            ),
          ),
          const SizedBox(width: 8),
          Text(
            '$count',
            style: theme.textTheme.labelMedium
                ?.copyWith(color: theme.colorScheme.onSurfaceVariant),
          ),
        ],
      ),
    );
  }
}

class _HabitCard extends StatelessWidget {
  const _HabitCard({
    required this.habit,
    required this.done,
    required this.onToggle,
  });

  final TinyHabit habit;
  final bool done;
  final VoidCallback onToggle;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Card(
      key: Key('habitCard.${habit.id}'),
      margin: const EdgeInsets.symmetric(vertical: 5),
      child: InkWell(
        onTap: onToggle,
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(14),
          child: Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Checkbox(value: done, onChanged: (_) => onToggle()),
              const SizedBox(width: 4),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Wrap(
                      spacing: 6,
                      runSpacing: 4,
                      crossAxisAlignment: WrapCrossAlignment.center,
                      children: [
                        Text(
                          habit.behavior,
                          style: theme.textTheme.titleSmall?.copyWith(
                            fontWeight: FontWeight.bold,
                            decoration:
                                done ? TextDecoration.lineThrough : null,
                          ),
                        ),
                        if (habit.isGolden) const _GoldenBadge(),
                      ],
                    ),
                    const SizedBox(height: 8),
                    Text(
                      habit.recipe,
                      style: theme.textTheme.bodyMedium?.copyWith(
                        color: theme.colorScheme.onSurface,
                        height: 1.35,
                      ),
                    ),
                    const SizedBox(height: 8),
                    Row(
                      children: [
                        Icon(Icons.celebration_outlined,
                            size: 15,
                            color: theme.colorScheme.tertiary),
                        const SizedBox(width: 6),
                        Expanded(
                          child: Text(
                            'Celebrate: ${habit.celebration}',
                            style: theme.textTheme.bodySmall?.copyWith(
                              color: theme.colorScheme.onSurfaceVariant,
                            ),
                          ),
                        ),
                      ],
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class _GoldenBadge extends StatelessWidget {
  const _GoldenBadge();

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
      decoration: BoxDecoration(
        color: theme.colorScheme.tertiaryContainer,
        borderRadius: BorderRadius.circular(20),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(Icons.star,
              size: 12, color: theme.colorScheme.onTertiaryContainer),
          const SizedBox(width: 3),
          Text(
            'Golden',
            style: theme.textTheme.labelSmall?.copyWith(
              color: theme.colorScheme.onTertiaryContainer,
              fontWeight: FontWeight.bold,
            ),
          ),
        ],
      ),
    );
  }
}
```

- [ ] **Step 4: Run the test to confirm it passes**

```powershell
flutter test test/screens/habit_map_screen_test.dart
```
Expected: PASS, 3 tests.

- [ ] **Step 5: Existing legacy `test/widget_test.dart` references `habit.done`. Replace the test file with the new shape (the legacy `done`-coupled test for the engine is removed; engine tests now live in `test/engine/habit_engine_test.dart`)**

Replace `hbit/test/widget_test.dart`:

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

  test('engine matches domain keywords and always adds growth habits', () {
    final habits = HabitEngine.generateHabits(
      aspirationFor('a senior software engineer',
          'I want to build great products and lead a team'),
    );

    expect(habits, isNotEmpty);
    expect(habits.any((h) => h.domain == 'engineering'), isTrue);
    expect(habits.any((h) => h.domain == 'growth'), isTrue);
  });

  test('engine falls back to growth foundations with no domain match', () {
    final habits = HabitEngine.generateHabits(
      aspirationFor('a happier person', 'I want more balance in my life'),
    );

    expect(habits, isNotEmpty);
    expect(habits.every((h) => h.domain == 'growth'), isTrue);
  });

  test('shorter horizon produces a tighter map', () {
    const identity = 'a senior engineering leader';
    const motivation =
        'I want to manage a team and one day found my own startup';

    final short =
        HabitEngine.generateHabits(aspirationFor(identity, motivation, '1 year'));
    final long =
        HabitEngine.generateHabits(aspirationFor(identity, motivation, '10 years'));

    expect(short.length, lessThan(long.length));
  });
}
```

- [ ] **Step 6: Run full test suite + analyzer**

```powershell
flutter analyze
flutter test
```
Expected: all pass.

- [ ] **Step 7: Commit**

```powershell
git add hbit/lib/screens/habit_map_screen.dart hbit/test/screens/habit_map_screen_test.dart hbit/test/widget_test.dart
git commit -m "refactor: HabitMapScreen reads streams and calls CheckInRepository"
```

---

## Task 3: Remove `done` field from TinyHabit

`TinyHabit.done` is no longer read by any screen. Remove it from the model and from `HabitMap.doneCount`.

**Files:**
- Modify: `hbit/lib/models/tiny_habit.dart`
- Modify: `hbit/lib/models/habit_map.dart`
- Modify: `hbit/test/models/habit_map_test.dart`

- [ ] **Step 1: Update the existing habit_map_test to drop the `done` reference**

In `hbit/test/models/habit_map_test.dart`, replace the `habitsByDomain` test (the last test in the file) with:

```dart
test('habitsByDomain groups by label preserving order', () {
  final map = HabitMap(
    aspiration: anyAspiration(),
    habits: [anyHabit(id: 'a'), anyHabit(id: 'b')],
  );
  expect(map.habitsByDomain['Engineering'], hasLength(2));
});
```

(The previous version had `anyHabit(id: 'a')..done = false` — remove the `..done = false`.)

- [ ] **Step 2: Rewrite `lib/models/tiny_habit.dart` without `done`**

Replace the file contents:

```dart
/// A single behavior shrunk to its tiniest form, following BJ Fogg's recipe:
/// "After I [anchor], I will [tiny action]" — then celebrate immediately.
class TinyHabit {
  const TinyHabit({
    required this.id,
    required this.domain,
    required this.domainLabel,
    required this.behavior,
    required this.anchorPrompt,
    required this.tinyAction,
    required this.celebration,
    required this.impact,
    required this.ease,
  });

  final String id;
  final String domain;
  final String domainLabel;

  /// The full-size behavior this tiny habit grows into.
  final String behavior;

  /// The existing routine the habit is anchored to ("After I ...").
  final String anchorPrompt;

  /// The tiny action, doable in under 30 seconds ("I will ...").
  final String tinyAction;

  /// The immediate celebration that wires the habit in.
  final String celebration;

  /// Impact toward the aspiration, 1-5.
  final int impact;

  /// Feasibility / ease of doing today, 1-5.
  final int ease;

  /// The Fogg recipe as one readable sentence.
  String get recipe => '$anchorPrompt, $tinyAction.';

  /// High impact AND high feasibility — a Tiny Habits "Golden Behavior".
  bool get isGolden => impact >= 4 && ease >= 4;

  Map<String, dynamic> toJson() => {
        'id': id,
        'domain': domain,
        'domainLabel': domainLabel,
        'behavior': behavior,
        'anchorPrompt': anchorPrompt,
        'tinyAction': tinyAction,
        'celebration': celebration,
        'impact': impact,
        'ease': ease,
      };

  factory TinyHabit.fromJson(Map<String, dynamic> json) => TinyHabit(
        id: json['id'] as String,
        domain: json['domain'] as String,
        domainLabel: json['domainLabel'] as String,
        behavior: json['behavior'] as String,
        anchorPrompt: json['anchorPrompt'] as String,
        tinyAction: json['tinyAction'] as String,
        celebration: json['celebration'] as String,
        impact: json['impact'] as int,
        ease: json['ease'] as int,
      );
}
```

- [ ] **Step 3: Remove `doneCount` from `HabitMap`**

In `hbit/lib/models/habit_map.dart`, delete the `doneCount` getter:

```dart
// Delete this block:
int get doneCount => habits.where((h) => h.done).length;
```

- [ ] **Step 4: Run the analyzer to surface every remaining reference**

```powershell
flutter analyze
```
Expected: any compile errors are in files that still reference `habit.done` or `habitMap.doneCount`. After the previous tasks, none should remain. If any surface, remove them.

- [ ] **Step 5: Run full tests**

```powershell
flutter test
```
Expected: all pass.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/models/tiny_habit.dart hbit/lib/models/habit_map.dart hbit/test/models/habit_map_test.dart
git commit -m "refactor: remove TinyHabit.done and HabitMap.doneCount"
```

---

## Task 4: HomeRoot widget — routes between AspirationScreen and HabitMapScreen

**Files:**
- Create: `hbit/lib/app/home_root.dart`
- Test: `hbit/test/app/home_root_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/app/home_root_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/app/home_root.dart';
import 'package:hbit/app/repositories_scope.dart';
import 'package:hbit/data/repositories.dart';
import 'package:hbit/models/app_user.dart';
import 'package:hbit/models/aspiration.dart';
import 'package:hbit/models/habit_map.dart';
import 'package:hbit/models/tiny_habit.dart';

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

HabitMap sampleMap() => HabitMap(
      id: 'map-1',
      aspiration: Aspiration(
        identity: 'a senior engineer',
        timeframe: '3 years',
        motivation: 'build great products',
        createdAt: DateTime.utc(2026, 5, 17),
      ),
      habits: [
        TinyHabit(
          id: 'h1',
          domain: 'engineering',
          domainLabel: 'Engineering',
          behavior: 'Daily review',
          anchorPrompt: 'After I sit at my desk',
          tinyAction: 'I read one PR',
          celebration: 'I say nice',
          impact: 4,
          ease: 4,
        ),
      ],
    );

Widget _harness(Repositories repos) => MaterialApp(
      home: RepositoriesScope(
        repositories: repos,
        child: const HomeRoot(),
      ),
    );

void main() {
  testWidgets('shows AspirationScreen when no active map', (tester) async {
    final repos = Repositories(
      auth: FakeAuthRepository(initialUser: sampleUser()),
      habitMaps: FakeHabitMapRepository(),
      checkIns: FakeCheckInRepository(),
    );
    await tester.pumpWidget(_harness(repos));
    await tester.pumpAndSettle();

    expect(find.text('Build your career habit map'), findsOneWidget);
  });

  testWidgets('shows HabitMapScreen when active map exists',
      (tester) async {
    final habitMaps = FakeHabitMapRepository();
    habitMaps.seedActive('uid-1', sampleMap());

    final repos = Repositories(
      auth: FakeAuthRepository(initialUser: sampleUser()),
      habitMaps: habitMaps,
      checkIns: FakeCheckInRepository(),
    );

    await tester.pumpWidget(_harness(repos));
    await tester.pumpAndSettle();

    expect(find.text('My Habit Map'), findsOneWidget);
    expect(find.text('Daily review'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/app/home_root_test.dart
```
Expected: FAIL — `home_root.dart` does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/app/home_root.dart`:

```dart
import 'package:flutter/material.dart';

import '../models/habit_map.dart';
import '../screens/aspiration_screen.dart';
import '../screens/habit_map_screen.dart';
import 'repositories_scope.dart';

class HomeRoot extends StatelessWidget {
  const HomeRoot({super.key});

  @override
  Widget build(BuildContext context) {
    final repos = RepositoriesScope.of(context);
    final user = repos.auth.currentUser;

    if (user == null) {
      // Should not happen — AuthGate guards this. Render a spinner just in case.
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }

    return StreamBuilder<HabitMap?>(
      stream: repos.habitMaps.activeMapStream(user.uid),
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const Scaffold(
            body: Center(child: CircularProgressIndicator()),
          );
        }
        final map = snapshot.data;
        if (map == null) return const AspirationScreen();
        return HabitMapScreen(habitMap: map);
      },
    );
  }
}
```

- [ ] **Step 4: Update AuthGate to use HomeRoot**

In `hbit/lib/app/auth_gate.dart`, change the import and the post-auth widget:

```dart
import 'package:flutter/material.dart';

import '../models/app_user.dart';
import '../screens/sign_in_screen.dart';
import 'home_root.dart';
import 'repositories_scope.dart';

class AuthGate extends StatelessWidget {
  const AuthGate({super.key});

  @override
  Widget build(BuildContext context) {
    final repos = RepositoriesScope.of(context);

    return StreamBuilder<AppUser?>(
      stream: repos.auth.userStream,
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting &&
            repos.auth.currentUser == null) {
          return const Scaffold(
            body: Center(child: CircularProgressIndicator()),
          );
        }
        final user = snapshot.data ?? repos.auth.currentUser;
        if (user == null) return const SignInScreen();
        return const HomeRoot();
      },
    );
  }
}
```

- [ ] **Step 5: Remove the `skip:` annotations from `auth_gate_test.dart` and update them**

In `hbit/test/app/auth_gate_test.dart`:

- Replace `'shows PostAuthRoot when signed in'` test name with `'shows HomeRoot when signed in'`.
- Replace `'navigates from SignIn to PostAuthRoot after sign-in'` test name with `'navigates from SignIn to HomeRoot after sign-in'`.
- Remove all `skip: ...` parameters.
- Both tests assert `find.text('Build your career habit map')` — that text comes from `AspirationScreen`, which `HomeRoot` shows when no map exists. They should pass.

- [ ] **Step 6: Run all tests**

```powershell
flutter analyze
flutter test
```
Expected: all pass; zero issues.

- [ ] **Step 7: Commit**

```powershell
git add hbit/lib/app/home_root.dart hbit/lib/app/auth_gate.dart hbit/test/app/home_root_test.dart hbit/test/app/auth_gate_test.dart
git commit -m "feat: add HomeRoot and wire AuthGate -> HomeRoot"
```

---

## Task 5: ProfileScreen with sign-out and timezone selector

**Files:**
- Create: `hbit/lib/screens/profile_screen.dart`
- Test: `hbit/test/screens/profile_screen_test.dart`

- [ ] **Step 1: Write the failing test**

Create `hbit/test/screens/profile_screen_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hbit/app/repositories_scope.dart';
import 'package:hbit/data/repositories.dart';
import 'package:hbit/models/app_user.dart';
import 'package:hbit/screens/profile_screen.dart';
import 'package:timezone/data/latest.dart' as tzdata;

import '../fakes/fake_auth_repository.dart';
import '../fakes/fake_check_in_repository.dart';
import '../fakes/fake_habit_map_repository.dart';

AppUser sampleUser({String tz = 'Europe/Berlin'}) => AppUser(
      uid: 'uid-1',
      email: 'a@b.test',
      displayName: 'Alice',
      timezone: tz,
      createdAt: DateTime.utc(2026, 1, 1),
      lastSignInAt: DateTime.utc(2026, 5, 17),
    );

Widget _harness(Repositories repos) => MaterialApp(
      home: RepositoriesScope(
        repositories: repos,
        child: const ProfileScreen(),
      ),
    );

void main() {
  setUpAll(tzdata.initializeTimeZones);

  late FakeAuthRepository auth;
  late Repositories repos;

  setUp(() {
    auth = FakeAuthRepository(initialUser: sampleUser());
    repos = Repositories(
      auth: auth,
      habitMaps: FakeHabitMapRepository(),
      checkIns: FakeCheckInRepository(),
    );
  });

  testWidgets('displays display name, email, and timezone', (tester) async {
    await tester.pumpWidget(_harness(repos));
    expect(find.text('Alice'), findsOneWidget);
    expect(find.text('a@b.test'), findsOneWidget);
    expect(find.text('Europe/Berlin'), findsOneWidget);
  });

  testWidgets('Sign out triggers AuthRepository.signOut', (tester) async {
    await tester.pumpWidget(_harness(repos));
    await tester.tap(find.text('Sign out'));
    await tester.pump();

    expect(auth.recordedCalls, contains('signOut'));
  });

  testWidgets('changing timezone via dropdown updates repository',
      (tester) async {
    await tester.pumpWidget(_harness(repos));
    await tester.tap(find.byKey(const Key('profile.timezone')));
    await tester.pumpAndSettle();

    // Find a known IANA zone in the menu and pick it.
    await tester.tap(find.text('America/Los_Angeles').last);
    await tester.pumpAndSettle();

    expect(
      auth.recordedCalls,
      contains('updateTimezone:America/Los_Angeles'),
    );
  });
}
```

- [ ] **Step 2: Run the test to confirm it fails**

```powershell
flutter test test/screens/profile_screen_test.dart
```
Expected: FAIL — `profile_screen.dart` does not exist.

- [ ] **Step 3: Write the implementation**

Create `hbit/lib/screens/profile_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:timezone/timezone.dart' as tz;

import '../app/repositories_scope.dart';

class ProfileScreen extends StatefulWidget {
  const ProfileScreen({super.key});

  @override
  State<ProfileScreen> createState() => _ProfileScreenState();
}

class _ProfileScreenState extends State<ProfileScreen> {
  late List<String> _zones;

  @override
  void initState() {
    super.initState();
    _zones = tz.timeZoneDatabase.locations.keys.toList()..sort();
  }

  Future<void> _signOut() async {
    await RepositoriesScope.of(context).auth.signOut();
  }

  Future<void> _changeTimezone(String? newZone) async {
    if (newZone == null) return;
    await RepositoriesScope.of(context).auth.updateTimezone(newZone);
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final user = RepositoriesScope.of(context).auth.currentUser;
    if (user == null) {
      return const Scaffold(body: Center(child: Text('Signed out.')));
    }

    return Scaffold(
      appBar: AppBar(title: const Text('Profile')),
      body: SafeArea(
        child: ListView(
          padding: const EdgeInsets.all(20),
          children: [
            _row('Display name', user.displayName ?? '—'),
            const Divider(),
            _row('Email', user.email),
            const Divider(),
            Padding(
              padding: const EdgeInsets.symmetric(vertical: 12),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    'Timezone',
                    style: theme.textTheme.titleSmall,
                  ),
                  const SizedBox(height: 8),
                  DropdownButtonFormField<String>(
                    key: const Key('profile.timezone'),
                    initialValue: user.timezone,
                    isExpanded: true,
                    decoration: const InputDecoration(
                      border: OutlineInputBorder(),
                    ),
                    items: _zones
                        .map((z) => DropdownMenuItem(value: z, child: Text(z)))
                        .toList(),
                    onChanged: _changeTimezone,
                  ),
                ],
              ),
            ),
            const SizedBox(height: 32),
            FilledButton.tonal(
              onPressed: _signOut,
              style: FilledButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 14),
              ),
              child: const Text('Sign out'),
            ),
          ],
        ),
      ),
    );
  }

  Widget _row(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 12),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          SizedBox(
            width: 120,
            child: Text(label, style: Theme.of(context).textTheme.titleSmall),
          ),
          Expanded(child: Text(value)),
        ],
      ),
    );
  }
}
```

- [ ] **Step 4: Register the `/profile` named route in `main.dart`**

Update `hbit/lib/main.dart` — replace the body of `HbitApp.build` to include named routes:

```dart
@override
Widget build(BuildContext context) {
  return RepositoriesScope(
    repositories: repositories,
    child: MaterialApp(
      title: 'Hbit',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const AuthGate(),
      routes: {
        '/profile': (_) => const ProfileScreen(),
      },
    ),
  );
}
```

Also add the import at the top:

```dart
import 'screens/profile_screen.dart';
```

- [ ] **Step 5: Run the tests**

```powershell
flutter analyze
flutter test
```
Expected: all pass.

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/screens/profile_screen.dart hbit/lib/main.dart hbit/test/screens/profile_screen_test.dart
git commit -m "feat: add ProfileScreen with sign-out and timezone selector"
```

---

## Task 6: Delete HabitStorage, PostAuthRoot, and shared_preferences

**Files:**
- Delete: `hbit/lib/services/habit_storage.dart`
- Delete: `hbit/lib/app/post_auth_root.dart`
- Modify: `hbit/pubspec.yaml`

- [ ] **Step 1: Delete the files**

```powershell
Remove-Item hbit\lib\services\habit_storage.dart
Remove-Item hbit\lib\app\post_auth_root.dart
```

- [ ] **Step 2: Remove the `shared_preferences` dependency from pubspec.yaml**

In `hbit/pubspec.yaml`, delete the line:

```yaml
  shared_preferences: ^2.5.5
```

- [ ] **Step 3: Run `flutter pub get` and the analyzer**

```powershell
flutter pub get
flutter analyze
```
Expected: no references to `HabitStorage`, `PostAuthRoot`, or `shared_preferences` remain. Zero issues.

- [ ] **Step 4: Run the full test suite**

```powershell
flutter test
```
Expected: all pass.

- [ ] **Step 5: Manually verify on device or emulator**

```powershell
flutter run
```
Verify:
1. App opens to `SignInScreen`
2. Sign in or register → reach `AspirationScreen`
3. Generate a map → see it appear (Firestore round-trip)
4. Toggle a habit → check-in persisted (kill app, reopen — habit still checked)
5. Tap profile icon → `ProfileScreen`
6. Change timezone → no error, dropdown updates
7. Sign out → return to `SignInScreen`
8. Sign in on a second device → same map and check-ins visible

- [ ] **Step 6: Commit**

```powershell
git add hbit/lib/services/habit_storage.dart hbit/lib/app/post_auth_root.dart hbit/pubspec.yaml hbit/pubspec.lock
git commit -m "chore: remove HabitStorage, PostAuthRoot, shared_preferences"
```

---

## Sprint 4 — Done definition

- [ ] `flutter test` from `hbit/` passes (Sprints 1–4 cumulative)
- [ ] `flutter analyze` from `hbit/` reports zero issues
- [ ] `flutter run` shows the full cloud-backed flow described in Task 6 Step 5
- [ ] `lib/services/habit_storage.dart` and `lib/app/post_auth_root.dart` no longer exist
- [ ] `shared_preferences` is no longer in `pubspec.yaml`
- [ ] `TinyHabit.done` and `HabitMap.doneCount` no longer exist
- [ ] All 6 tasks committed in sequence
- [ ] Tag the sprint completion: `git tag sprint-4-map-cloud`

Next: [Sprint 5 — History, security, docs](2026-05-17-auth-cloud-storage-sprint-5-history-security.md).
