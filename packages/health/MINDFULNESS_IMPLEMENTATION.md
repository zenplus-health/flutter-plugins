# Mindfulness Support Fork Implementation Guide

**Created:** 2 October 2025  
**For:** Zen+ Health Android Mindfulness Session Support  
**Target Package:** flutter-plugins/packages/health (fork)

---

## Table of Contents
1. [Background & Context](#background--context)
2. [Why This Fork is Necessary](#why-this-fork-is-necessary)
3. [Fork Setup Instructions](#fork-setup-instructions)
4. [Implementation Details](#implementation-details)
5. [Testing Strategy](#testing-strategy)
6. [Integration with Zen+ App](#integration-with-zen-app)
7. [Maintenance & Future](#maintenance--future)

---

## Background & Context

### The Problem

The Zen+ Health app needs to write mindfulness session data to Android Health Connect. Currently, the app uses the `health` package (v13.2.0) which:

- ✅ **iOS**: Properly writes to `MINDFULNESS` data type in Apple HealthKit
- ❌ **Android**: Uses a workaround writing to `WORKOUT` with `GUIDED_BREATHING` activity type

This workaround has several issues:
1. Mindfulness sessions appear as "exercise" instead of "mindfulness" in Health Connect
2. No support for mindfulness subtypes (meditation, breathing exercises, body awareness)
3. Inconsistent with Android's native `MindfulnessSessionRecord` API introduced in Health Connect SDK 1.1.0

### Android's Native API

As of Health Connect SDK 1.1.0-alpha10 (and stable in later versions), Android provides:

**`MindfulnessSessionRecord`** class with:
- Proper categorization as mindfulness (not exercise/workout)
- Support for session types:
  - `MINDFULNESS_SESSION_TYPE_MEDITATION` (0)
  - `MINDFULNESS_SESSION_TYPE_BREATHING_EXERCISE` (1)
  - `MINDFULNESS_SESSION_TYPE_BODY_AWARENESS` (2)
  - `MINDFULNESS_SESSION_TYPE_UNSPECIFIED` (3)
- Metadata support (title, recording method, device type)

**Documentation:**
- API Guide: https://developer.android.com/health-and-fitness/guides/health-connect/develop/mindfulness
- API Reference: https://developer.android.com/reference/android/health/connect/datatypes/MindfulnessSessionRecord

### Why It's Critical

For Zen+ Health:
1. **Accurate categorization**: Users see "Mindfulness" not "Guided Breathing" or "Exercise"
2. **Subtype support required**: Different task types (meditation, breathing, body scan) map to different mindfulness subtypes
3. **Platform consistency**: iOS already uses native `MINDFULNESS` type; Android should match semantically
4. **Future-proofing**: Health Connect is moving towards specific record types, away from generic workouts

---

## Why This Fork is Necessary

### Why Not Upstream Package?

The current `health` package (https://github.com/cph-cachet/flutter-plugins/tree/master/packages/health) v13.2.0:

1. **Does not include `MindfulnessSessionRecord` support**
   - Only supports older Health Connect record types
   - No enum for `MindfulnessSessionType`
   - No method to write mindfulness sessions

2. **Package is stable but slow-moving**
   - Last major update: Several months ago
   - Community PRs take time to merge
   - Can't wait for upstream to accept contribution

3. **Requires API additions**
   - New Dart enum: `MindfulnessSessionType`
   - New Dart method: `writeMindfulnessSession()`
   - New Kotlin constants and writer method
   - Method channel routing

### Why Fork vs Custom Channel?

**Fork Advantages:**
- ✅ Cleaner architecture (single health API surface)
- ✅ Leverages existing permission handling
- ✅ Consistent with iOS implementation pattern
- ✅ Reusable if open-sourced
- ✅ Can be contributed back upstream later

**Custom Channel Disadvantages:**
- ❌ Dual-channel complexity
- ❌ Separate permission handling needed
- ❌ More native code in app repo
- ❌ Not reusable across projects

---

## Fork Setup Instructions

### 1. Fork the Repository

**Upstream:** https://github.com/cph-cachet/flutter-plugins

**Steps:**
```bash
# Fork via GitHub UI or gh CLI
gh repo fork cph-cachet/flutter-plugins --org zenplus-health --fork-name flutter-plugins

# Clone your fork
cd ~/projects/zenplus-health  # or wherever you keep repos
git clone https://github.com/zenplus-health/flutter-plugins.git
cd flutter-plugins

# Add upstream remote for future updates
git remote add upstream https://github.com/cph-cachet/flutter-plugins.git

# Create feature branch
git checkout -b mindfulness-support
```

### 2. Repository Structure

The `flutter-plugins` repo is a monorepo with multiple packages. We only care about:

```
flutter-plugins/
├── packages/
│   ├── health/                    # ← THIS IS WHAT WE'RE FORKING
│   │   ├── lib/
│   │   │   ├── health.dart
│   │   │   └── src/
│   │   │       ├── health_plugin.dart       # ← ADD: writeMindfulnessSession()
│   │   │       ├── heath_data_types.dart    # ← ADD: MindfulnessSessionType enum
│   │   │       └── ...
│   │   ├── android/
│   │   │   └── src/main/kotlin/cachet/plugins/health/
│   │   │       ├── HealthPlugin.kt          # ← UPDATE: Add method routing
│   │   │       ├── HealthDataWriter.kt      # ← ADD: writeMindfulnessSession()
│   │   │       ├── HealthConstants.kt       # ← ADD: Mindfulness type mappings
│   │   │       └── ...
│   │   ├── ios/
│   │   │   └── ...                          # NO CHANGES NEEDED (already works)
│   │   ├── pubspec.yaml
│   │   └── CHANGELOG.md                     # ← UPDATE: Document changes
│   └── ... (other packages we don't care about)
```

---

## Implementation Details

### Phase 1: Dart API Layer

#### File: `packages/health/lib/src/heath_data_types.dart`

**Location to add:** After the existing enums (around line 150-200, after `HealthWorkoutActivityType`)

**Add new enum:**

```dart
/// Mindfulness session types for Health Connect (Android) and HealthKit (iOS).
/// 
/// These map to:
/// - Android: MindfulnessSessionRecord.MINDFULNESS_SESSION_TYPE_* constants
/// - iOS: HKCategoryTypeIdentifierMindfulSession (type is implicit in iOS)
enum MindfulnessSessionType {
  /// Meditation session (default).
  /// Android value: MINDFULNESS_SESSION_TYPE_MEDITATION (0)
  meditation,

  /// Breathing exercise session.
  /// Android value: MINDFULNESS_SESSION_TYPE_BREATHING_EXERCISE (1)
  breathingExercise,

  /// Body awareness or body scan session.
  /// Android value: MINDFULNESS_SESSION_TYPE_BODY_AWARENESS (2)
  bodyAwareness,

  /// Unspecified mindfulness session type.
  /// Android value: MINDFULNESS_SESSION_TYPE_UNSPECIFIED (3)
  unspecified,
}
```

**Why these values:**
- Enum ordering matches Android's integer constants (0, 1, 2, 3)
- Names follow Dart naming conventions (camelCase)
- Comments explain mapping for future maintainers

---

#### File: `packages/health/lib/src/health_plugin.dart`

**Location to add:** In the `Health` class, after existing write methods (around line 400-500, after `writeWorkoutData`)

**Add new method:**

```dart
/// Writes a mindfulness session to the platform's health database.
///
/// On iOS, writes to HealthKit's MINDFULNESS category type.
/// On Android, writes to Health Connect's MindfulnessSessionRecord.
///
/// Parameters:
/// - [type]: The type of mindfulness session (meditation, breathing, etc.)
/// - [startTime]: The start time of the session
/// - [endTime]: The end time of the session  
/// - [title]: Optional title/description for the session
/// - [recordingMethod]: How the data was recorded (defaults to automatic)
///
/// Returns true if the write was successful, false otherwise.
///
/// Example:
/// ```dart
/// final success = await Health().writeMindfulnessSession(
///   type: MindfulnessSessionType.meditation,
///   startTime: DateTime.now().subtract(Duration(minutes: 10)),
///   endTime: DateTime.now(),
///   title: "Morning meditation",
/// );
/// ```
Future<bool> writeMindfulnessSession({
  required MindfulnessSessionType type,
  required DateTime startTime,
  required DateTime endTime,
  String? title,
  RecordingMethod recordingMethod = RecordingMethod.automatic,
}) async {
  // iOS implementation
  if (Platform.isIOS) {
    // iOS uses the existing MINDFULNESS data type
    // Session type is implicit (iOS doesn't categorize mindfulness subtypes)
    final minutes = endTime.difference(startTime).inSeconds / 60.0;
    return writeHealthData(
      value: minutes,
      type: HealthDataType.MINDFULNESS,
      startTime: startTime,
      endTime: endTime,
      recordingMethod: recordingMethod,
    );
  }

  // Android implementation
  if (Platform.isAndroid) {
    try {
      final arguments = {
        'mindfulnessType': type.index, // 0, 1, 2, or 3
        'startTime': startTime.millisecondsSinceEpoch,
        'endTime': endTime.millisecondsSinceEpoch,
        'title': title,
        'recordingMethod': recordingMethod.index,
      };

      final success = await _channel.invokeMethod(
        'writeMindfulnessSession',
        arguments,
      );

      return success == true;
    } catch (e) {
      debugPrint('Error writing mindfulness session: $e');
      return false;
    }
  }

  // Unsupported platform
  return false;
}
```

**Why this implementation:**
- Reuses iOS's existing `writeHealthData` path (no iOS changes needed)
- Creates new method channel call for Android
- Matches existing method patterns in the package
- Includes comprehensive documentation
- Error handling follows package conventions

---

### Phase 2: Android Native Layer

#### File: `packages/health/android/src/main/kotlin/cachet/plugins/health/HealthConstants.kt`

**Location to add:** In the companion object, after existing constants (around line 50-100)

**Add mindfulness type mappings:**

```kotlin
// Mindfulness session types (Android Health Connect)
// These map to MindfulnessSessionRecord.MINDFULNESS_SESSION_TYPE_* constants
// Listed here: https://developer.android.com/reference/android/health/connect/datatypes/MindfulnessSessionRecord
const val MINDFULNESS_SESSION_TYPE_UNKNOWN = 0
const val MINDFULNESS_SESSION_TYPE_MEDITATION = 1
const val MINDFULNESS_SESSION_TYPE_OTHER = 2
const val MINDFULNESS_SESSION_TYPE_BREATHING = 3
const val MINDFULNESS_SESSION_TYPE_MUSIC = 4
const val MINDFULNESS_SESSION_TYPE_MOVEMENT = 5
const val MINDFULNESS_SESSION_TYPE_UNGUIDED = 6

// Recording method constants (if not already present)
const val RECORDING_METHOD_ACTIVELY_RECORDED = 0
const val RECORDING_METHOD_AUTOMATIC = 1
const val RECORDING_METHOD_MANUAL_ENTRY = 2
const val RECORDING_METHOD_UNKNOWN = 3
```

**Why these constants:**
- Match Android's official constant values exactly
- Provide clear documentation for mapping
- Follow existing constant naming pattern in the file

---

#### File: `packages/health/android/src/main/kotlin/cachet/plugins/health/HealthPlugin.kt`

**Location to modify:** In `onMethodCall` method (around line 120-150)

**Find the existing method routing:**

```kotlin
override fun onMethodCall(call: MethodCall, result: Result) {
    when (call.method) {
        "requestAuthorization" -> requestAuthorization(call, result)
        "writeWorkoutData" -> writeWorkoutData(call, result)
        "writeHealthData" -> writeHealthData(call, result)
        // ... other methods ...
```

**Add new routing:**

```kotlin
override fun onMethodCall(call: MethodCall, result: Result) {
    when (call.method) {
        "requestAuthorization" -> requestAuthorization(call, result)
        "writeWorkoutData" -> writeWorkoutData(call, result)
        "writeHealthData" -> writeHealthData(call, result)
        "writeMindfulnessSession" -> healthDataWriter.writeMindfulnessSession(call, result)  // ← ADD THIS
        // ... other methods ...
```

**Why here:**
- Central routing point for all method channel calls
- Delegates to `healthDataWriter` (follows existing pattern)
- Simple one-line addition

---

#### File: `packages/health/android/src/main/kotlin/cachet/plugins/health/HealthDataWriter.kt`

**Location to add:** After existing write methods (around line 200-300, after `writeWorkoutData`)

**Add import at top of file:**

```kotlin
import androidx.health.connect.client.records.MindfulnessSessionRecord
import androidx.health.connect.client.ExperimentalHealthConnectApi
```

**Add new method:**

```kotlin
/**
 * Writes a mindfulness session to Health Connect using MindfulnessSessionRecord.
 * 
 * This is a newer record type that properly categorizes mindfulness activities
 * separately from exercise/workout activities.
 * 
 * @param call Method call containing:
 *   - 'mindfulnessType': Int (0=meditation, 1=breathing, 2=body awareness, 3=unspecified)
 *   - 'startTime': Long (milliseconds since epoch)
 *   - 'endTime': Long (milliseconds since epoch)
 *   - 'title': String? (optional session title)
 *   - 'recordingMethod': Int (0=active, 1=automatic, 2=manual, 3=unknown)
 * @param result Flutter result callback
 */
@OptIn(ExperimentalHealthConnectApi::class)
fun writeMindfulnessSession(call: MethodCall, result: Result) {
    // Extract parameters with defaults
    val mindfulnessType = call.argument<Int>("mindfulnessType") ?: MINDFULNESS_SESSION_TYPE_MEDITATION
    val startTimeMillis = call.argument<Long>("startTime")
    val endTimeMillis = call.argument<Long>("endTime")
    val title = call.argument<String>("title")
    val recordingMethodIndex = call.argument<Int>("recordingMethod") ?: RECORDING_METHOD_AUTOMATIC

    // Validate required parameters
    if (startTimeMillis == null || endTimeMillis == null) {
        result.error("INVALID_ARGUMENT", "startTime and endTime are required", null)
        return
    }

    // Convert to Instant
    val startTime = Instant.ofEpochMilli(startTimeMillis)
    val endTime = Instant.ofEpochMilli(endTimeMillis)

    // Build metadata (reuse existing buildMetadata helper if available, or create inline)
    val metadata = Metadata(
        recordingMethod = when (recordingMethodIndex) {
            RECORDING_METHOD_ACTIVELY_RECORDED -> Metadata.RECORDING_METHOD_ACTIVELY_RECORDED
            RECORDING_METHOD_AUTOMATIC -> Metadata.RECORDING_METHOD_AUTOMATICALLY_RECORDED
            RECORDING_METHOD_MANUAL_ENTRY -> Metadata.RECORDING_METHOD_MANUAL_ENTRY
            else -> Metadata.RECORDING_METHOD_UNKNOWN
        }
    )

    // Launch coroutine to write data
    scope.launch {
        try {
            val record = MindfulnessSessionRecord(
                startTime = startTime,
                startZoneOffset = null, // Use device default timezone
                endTime = endTime,
                endZoneOffset = null,
                mindfulnessSessionType = mindfulnessType,
                title = title,
                metadata = metadata
            )

            healthConnectClient.insertRecords(listOf(record))
            
            Log.i(
                "FLUTTER_HEALTH::SUCCESS",
                "Mindfulness session written: type=$mindfulnessType, " +
                "duration=${(endTimeMillis - startTimeMillis) / 1000}s, title=$title"
            )
            
            result.success(true)
        } catch (e: Exception) {
            Log.e(
                "FLUTTER_HEALTH::ERROR",
                "Failed to write mindfulness session: ${e.message}",
                e
            )
            result.success(false)
        }
    }
}
```

**Why this implementation:**
- Uses `@OptIn` for experimental API (MindfulnessSessionRecord is still experimental in some SDK versions)
- Comprehensive parameter validation
- Proper error handling and logging
- Follows existing method patterns in the file
- Includes detailed documentation

---

#### File: `packages/health/android/src/main/kotlin/cachet/plugins/health/HealthConstants.kt` (Part 2)

**If permission handling needs updates:**

Find the permissions mapping section and ensure mindfulness permissions are included:

```kotlin
// In the dataTypeToHealthPermission or similar mapping
when (dataType) {
    // ... existing mappings ...
    "MINDFULNESS" -> listOf(
        HealthPermission.createReadPermission(MindfulnessSessionRecord::class),
        HealthPermission.createWritePermission(MindfulnessSessionRecord::class)
    )
    // ... more mappings ...
}
```

**Note:** The existing `requestAuthorization` method may need updates if it doesn't already handle dynamic permission types. Check the current implementation.

---

### Phase 3: Documentation & Changelog

#### File: `packages/health/CHANGELOG.md`

**Add at the top:**

```markdown
## [UNRELEASED] - Mindfulness Support Branch

### Added
- **Android**: Native `MindfulnessSessionRecord` support for Health Connect
  - New `MindfulnessSessionType` enum with 4 session types
  - New `writeMindfulnessSession()` method for writing mindfulness sessions
  - Proper categorization as mindfulness (not exercise/workout)
- **Documentation**: Added comprehensive docs for mindfulness session types

### Changed
- **Android**: Mindfulness sessions now use native Health Connect API instead of workout workaround
- **iOS**: No changes - already uses native MINDFULNESS data type

### Notes
This is a fork maintained by Zen+ Health (zenplus-health) with mindfulness support.
Intended to be contributed back to upstream `cph-cachet/flutter-plugins`.
```

---

#### File: `packages/health/README.md`

**Add section after existing "Writing Data" section:**

```markdown
### Writing Mindfulness Sessions

Write mindfulness sessions to the platform's health database:

```dart
// Write a meditation session
final success = await Health().writeMindfulnessSession(
  type: MindfulnessSessionType.meditation,
  startTime: DateTime.now().subtract(Duration(minutes: 10)),
  endTime: DateTime.now(),
  title: "Morning meditation",
);

// Write a breathing exercise
await Health().writeMindfulnessSession(
  type: MindfulnessSessionType.breathingExercise,
  startTime: startTime,
  endTime: endTime,
  title: "Box breathing exercise",
);
```

**Supported Mindfulness Types:**
- `MindfulnessSessionType.meditation` - Meditation sessions
- `MindfulnessSessionType.breathingExercise` - Breathing exercises
- `MindfulnessSessionType.bodyAwareness` - Body scan, body awareness practices
- `MindfulnessSessionType.unspecified` - Other mindfulness activities

**Platform Behavior:**
- **iOS**: Writes to HealthKit's `MINDFULNESS` category (session type is not categorized in iOS)
- **Android**: Writes to Health Connect's `MindfulnessSessionRecord` (supports session subtypes)
```

---

### Phase 4: Export & Versioning

#### File: `packages/health/lib/health.dart`

**Ensure new types are exported:**

```dart
library health;

export 'src/health_plugin.dart';
export 'src/heath_data_types.dart'; // This should export MindfulnessSessionType
export 'src/health_data_point.dart';
// ... other exports
```

**Verify `MindfulnessSessionType` is exported.**

---

#### File: `packages/health/pubspec.yaml`

**Update version:**

```yaml
name: health
description: Wrapper for Apple HealthKit and Google Health Connect APIs
version: 13.2.0+mindfulness.1  # or 13.3.0-mindfulness.1
# ... rest of file
```

**Version strategy:**
- Option 1: `13.2.0+mindfulness.1` (build number suffix)
- Option 2: `13.3.0-mindfulness.1` (pre-release)
- Option 3: Wait to bump version until ready to contribute upstream

---

## Testing Strategy

### 1. Unit Tests

**File to create:** `packages/health/test/mindfulness_test.dart`

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:health/health.dart';

void main() {
  group('MindfulnessSessionType', () {
    test('enum values match Android constants', () {
      expect(MindfulnessSessionType.meditation.index, 0);
      expect(MindfulnessSessionType.breathingExercise.index, 1);
      expect(MindfulnessSessionType.bodyAwareness.index, 2);
      expect(MindfulnessSessionType.unspecified.index, 3);
    });
  });

  group('writeMindfulnessSession', () {
    // Add mock tests as needed
    test('constructs proper arguments for Android', () async {
      // Mock channel and verify arguments
    });
  });
}
```

**Run tests:**
```bash
cd packages/health
flutter test
```

---

### 2. Integration Testing (Android Device)

**Test app to create in fork:** `packages/health/example/`

Update the example app to include mindfulness testing:

```dart
// In example/lib/main.dart
ElevatedButton(
  onPressed: () async {
    final now = DateTime.now();
    final start = now.subtract(Duration(minutes: 10));
    
    final success = await health.writeMindfulnessSession(
      type: MindfulnessSessionType.meditation,
      startTime: start,
      endTime: now,
      title: "Test meditation",
    );
    
    print('Mindfulness write success: $success');
  },
  child: Text('Test Mindfulness Write'),
),
```

**Testing checklist:**
- [ ] Request permissions for mindfulness data
- [ ] Write meditation session
- [ ] Write breathing exercise session  
- [ ] Write body awareness session
- [ ] Verify in Health Connect app that sessions appear as "Mindfulness" not "Guided breathing"
- [ ] Verify session types are correct in Health Connect
- [ ] Test with missing optional parameters (title)
- [ ] Test error cases (invalid timestamps, etc.)

---

### 3. iOS Testing (Verify No Regression)

Even though we're not changing iOS code, verify it still works:

```bash
cd packages/health/example
flutter run -d "iPhone Simulator"
```

**Test checklist:**
- [ ] Existing mindfulness writes still work
- [ ] New `writeMindfulnessSession()` works on iOS
- [ ] Permissions still work
- [ ] No new warnings or errors

---

## Integration with Zen+ App

Once the fork is complete and tested, integrate it into the Zen+ Health app.

### Step 1: Update pubspec.yaml

**File:** `zen-apps/pubspec.yaml`

**Replace:**
```yaml
dependencies:
  health: ^13.2.0
```

**With:**
```yaml
dependencies:
  health:
    git:
      url: https://github.com/zenplus-health/flutter-plugins.git
      ref: mindfulness-support
      path: packages/health
```

**Run:**
```bash
cd ~/Documents/projects/zp/zen-apps
flutter pub get
```

---

### Step 2: Update HealthSyncService

**File:** `lib/services/health_sync_service.dart`

**Add import:**
```dart
import 'package:health/health.dart'; // Now includes MindfulnessSessionType
```

**Add enum for task mapping:**
```dart
/// Maps Zen+ task types to mindfulness session types
enum ZenTaskType {
  meditation,
  breathing,
  bodyAwareness,
  grounding,
  other,
}
```

**Update writeMindfulnessMinutes method:**

Find the current implementation (around line 247) and replace:

```dart
Future<bool> writeMindfulnessMinutes({
  required int durationSeconds,
  DateTime? startTime,
  DateTime? endTime,
  ZenTaskType taskType = ZenTaskType.meditation, // NEW PARAMETER
}) async {
  // Early returns for disabled/unavailable states
  if (!isAvailable) {
    Logger.log(LogLevel.debug, 'HealthSyncService',
        'Platform not supported, skipping write');
    return false;
  }

  await _ensureInitialized();

  if (!_enabled) {
    Logger.log(LogLevel.debug, 'HealthSyncService',
        'Health sync disabled, skipping write');
    return false;
  }

  if (!_hasPermission) {
    Logger.log(
        LogLevel.debug, 'HealthSyncService', 'No permission, skipping write');
    return false;
  }

  // Calculate time range
  final now = DateTime.now();
  final end = endTime ?? now;
  final start = startTime ?? end.subtract(Duration(seconds: durationSeconds));

  try {
    bool success = false;

    if (Platform.isIOS) {
      // iOS: Use existing MINDFULNESS data type (no subtype support)
      final minutes = durationSeconds / 60.0;
      success = await _health.writeHealthData(
        value: minutes,
        type: HealthDataType.MINDFULNESS,
        startTime: start,
        endTime: end,
        recordingMethod: RecordingMethod.automatic,
      );

      Logger.log(LogLevel.debug, 'HealthSyncService',
          'Wrote $minutes minutes of MINDFULNESS to Apple Health: $success');
    } else if (Platform.isAndroid) {
      // Android: Use new native MindfulnessSessionRecord API
      final mindfulnessType = _mapTaskTypeToMindfulness(taskType);
      
      success = await _health.writeMindfulnessSession(
        type: mindfulnessType,
        startTime: start,
        endTime: end,
        title: "Zen+ mindfulness session",
        recordingMethod: RecordingMethod.automatic,
      );

      Logger.log(LogLevel.debug, 'HealthSyncService',
          'Wrote mindfulness session to Health Connect: $success (type: $mindfulnessType, duration: ${durationSeconds}s)');
    }

    return success;
  } catch (e, stackTrace) {
    // Fail silently as per requirements
    Logger.log(LogLevel.debug, 'HealthSyncService',
        'Failed to write health data: $e',
        stackTrace: stackTrace);
    return false;
  }
}

/// Maps Zen+ task types to Health Connect mindfulness session types
MindfulnessSessionType _mapTaskTypeToMindfulness(ZenTaskType taskType) {
  switch (taskType) {
    case ZenTaskType.meditation:
      return MindfulnessSessionType.meditation;
    case ZenTaskType.breathing:
      return MindfulnessSessionType.breathingExercise;
    case ZenTaskType.bodyAwareness:
    case ZenTaskType.grounding:
      return MindfulnessSessionType.bodyAwareness;
    case ZenTaskType.other:
    default:
      return MindfulnessSessionType.unspecified;
  }
}
```

**Why this approach:**
- iOS behavior unchanged (already works correctly)
- Android now uses proper native API
- Task type mapping is explicit and documented
- Backward compatible (taskType parameter has default)

---

### Step 3: Update Task Screens

**Files to update:** `lib/screens/tasks/*.dart`

Each task screen that calls `writeMindfulnessMinutes()` should pass the appropriate task type.

**Example - Meditation Screen:**

```dart
// In lib/screens/tasks/meditation.dart
await healthSync.writeMindfulnessMinutes(
  durationSeconds: durationSeconds,
  taskType: ZenTaskType.meditation, // NEW
);
```

**Example - Breathing Screen:**

```dart
// In lib/screens/tasks/breathing.dart
await healthSync.writeMindfulnessMinutes(
  durationSeconds: durationSeconds,
  taskType: ZenTaskType.breathing, // NEW
);
```

**Example - Grounding/Body Scan Screen:**

```dart
// In lib/screens/tasks/grounding.dart (or body_scan.dart)
await healthSync.writeMindfulnessMinutes(
  durationSeconds: durationSeconds,
  taskType: ZenTaskType.bodyAwareness, // NEW
);
```

**Or in base screen:**

If called from `BaseTaskScreenState`, determine task type from task metadata:

```dart
// In lib/screens/tasks/base.dart - in reportTaskComplete()
ZenTaskType getTaskType() {
  final taskName = task.catalogTask.taskName?.toLowerCase() ?? '';
  if (taskName.contains('meditat')) return ZenTaskType.meditation;
  if (taskName.contains('breath')) return ZenTaskType.breathing;
  if (taskName.contains('body') || taskName.contains('ground')) {
    return ZenTaskType.bodyAwareness;
  }
  return ZenTaskType.other;
}

// Then call:
await healthSync.writeMindfulnessMinutes(
  durationSeconds: durationSeconds,
  taskType: getTaskType(),
);
```

---

### Step 4: Update Android Permissions

**File:** `android/app/src/main/AndroidManifest.xml`

**Replace existing permissions:**

```xml
<!-- OLD: Exercise permissions -->
<uses-permission android:name="android.permission.health.READ_EXERCISE"/>
<uses-permission android:name="android.permission.health.WRITE_EXERCISE"/>
```

**With:**

```xml
<!-- Mindfulness permissions -->
<uses-permission android:name="android.permission.health.READ_MINDFULNESS"/>
<uses-permission android:name="android.permission.health.WRITE_MINDFULNESS"/>

<!-- Note: EXERCISE permissions may still be needed if reading workout data -->
```

---

### Step 5: Test Integration

**Test on Android device:**

```bash
cd ~/Documents/projects/zp/zen-apps
flutter clean
flutter pub get
flutter run -d <android-device>
```

**Test checklist:**
- [ ] Go to Settings → Enable health sync
- [ ] Permission dialog shows "Mindfulness" not "Guided breathing"
- [ ] Complete a meditation task (80%+)
- [ ] Check Health Connect app - verify "Mindfulness" entry appears
- [ ] Verify session type is "Meditation"
- [ ] Complete a breathing exercise task
- [ ] Check Health Connect - verify "Breathing exercise" subtype
- [ ] Complete a body awareness/grounding task
- [ ] Check Health Connect - verify "Body awareness" subtype
- [ ] Verify iOS still works (no regression)

---

### Step 6: Update Documentation

**File:** `zen-apps/HEALTH_SYNC_IMPLEMENTATION.md`

Update the Android section:

```markdown
### Android (Health Connect)
- **Data Type**: MindfulnessSessionRecord (native mindfulness support)
- **Session Types**: 
  - Meditation
  - Breathing Exercise
  - Body Awareness
  - Unspecified
- **Recording Method**: AUTOMATIC
- **Package**: forked health package with mindfulness support

Note: Uses forked `health` package from zenplus-health/flutter-plugins with native
MindfulnessSessionRecord support added.
```

---

## Maintenance & Future

### Keeping Fork Updated

**Periodic sync with upstream:**

```bash
cd ~/projects/zenplus-health/flutter-plugins
git checkout mindfulness-support
git fetch upstream
git merge upstream/master

# Resolve any conflicts
# Test thoroughly after merge
git push origin mindfulness-support
```

**Recommended cadence:** Every 2-3 months or when major upstream changes occur

---

### Contributing Back to Upstream

Once stable and tested, consider submitting a PR to `cph-cachet/flutter-plugins`:

**PR Checklist:**
- [ ] All tests passing
- [ ] Documentation complete
- [ ] CHANGELOG updated
- [ ] Example app demonstrates new feature
- [ ] No breaking changes to existing API
- [ ] iOS behavior unchanged (no regression)
- [ ] Follows package code style

**PR Description Template:**

```markdown
## Add native MindfulnessSessionRecord support for Android Health Connect

### Summary
Adds support for writing mindfulness sessions to Health Connect using the native
`MindfulnessSessionRecord` API introduced in Health Connect SDK 1.1.0.

### Motivation
Currently, mindfulness sessions must be written as WORKOUT records with 
GUIDED_BREATHING activity type. This causes them to appear as "exercise" 
rather than "mindfulness" in Health Connect.

Android Health Connect now provides a dedicated MindfulnessSessionRecord 
API that properly categorizes mindfulness activities with support for subtypes
(meditation, breathing exercises, body awareness).

### Changes
- New `MindfulnessSessionType` enum
- New `writeMindfulnessSession()` method
- Android: Kotlin implementation using MindfulnessSessionRecord
- iOS: Delegates to existing MINDFULNESS data type (no changes)
- Documentation and tests

### Testing
- Unit tests added
- Integration tested on Android API 34
- iOS tested for no regression
- Example app updated

### Breaking Changes
None - this is a pure addition

### Related Issues
[Link to any related GitHub issues if they exist]
```

---

### Alternative: Keep Fork Private

If upstream doesn't accept the PR or you prefer to keep it private:

**Advantages:**
- ✅ Full control over timing and features
- ✅ Can add Zen+ specific customizations
- ✅ No waiting for upstream review

**Maintenance strategy:**
- Tag stable versions: `git tag v13.2.0-mindfulness.1`
- Use in pubspec: `ref: v13.2.0-mindfulness.1` (instead of branch)
- Document fork purpose in fork's README
- Keep fork public for transparency (but not published to pub.dev)

---

## Summary Checklist

### Fork Setup
- [ ] Fork cph-cachet/flutter-plugins to zenplus-health
- [ ] Clone fork locally
- [ ] Create `mindfulness-support` branch
- [ ] Add upstream remote

### Dart Changes
- [ ] Add `MindfulnessSessionType` enum to `heath_data_types.dart`
- [ ] Add `writeMindfulnessSession()` method to `health_plugin.dart`
- [ ] Export new types in `health.dart`
- [ ] Update version in `pubspec.yaml`

### Kotlin Changes
- [ ] Add mindfulness constants to `HealthConstants.kt`
- [ ] Add method routing in `HealthPlugin.kt`
- [ ] Add `writeMindfulnessSession()` to `HealthDataWriter.kt`
- [ ] Update permissions handling if needed

### Documentation
- [ ] Update `CHANGELOG.md`
- [ ] Update `README.md` with examples
- [ ] Add code comments

### Testing
- [ ] Write unit tests
- [ ] Test in example app (Android)
- [ ] Test in example app (iOS - no regression)
- [ ] Integration test on physical Android device

### Zen+ App Integration
- [ ] Update `pubspec.yaml` to use fork
- [ ] Update `HealthSyncService` implementation
- [ ] Add `ZenTaskType` enum
- [ ] Add task type mapping logic
- [ ] Update task screens to pass task type
- [ ] Update Android permissions
- [ ] Test full integration
- [ ] Update documentation

### Future
- [ ] Set up periodic upstream sync
- [ ] (Optional) Submit PR to upstream
- [ ] Document fork maintenance process

---

## Quick Reference

### Key Files to Modify in Fork

| File | Purpose | Lines to Add |
|------|---------|--------------|
| `heath_data_types.dart` | Add enum | ~30 lines |
| `health_plugin.dart` | Add method | ~50 lines |
| `HealthConstants.kt` | Add constants | ~10 lines |
| `HealthPlugin.kt` | Add routing | ~1 line |
| `HealthDataWriter.kt` | Add writer | ~60 lines |
| `CHANGELOG.md` | Document changes | ~10 lines |
| `README.md` | Add examples | ~30 lines |

**Total new code:** ~190 lines across 7 files

---

## Contact & Questions

**Fork Maintainer:** Zen+ Health Development Team  
**Fork Repository:** https://github.com/zenplus-health/flutter-plugins  
**Branch:** `mindfulness-support`  
**Upstream:** https://github.com/cph-cachet/flutter-plugins

For questions about this implementation, refer back to this document or the commit history in the fork.

---

**Document Version:** 1.0  
**Last Updated:** 2 October 2025  
**Status:** Ready for implementation
