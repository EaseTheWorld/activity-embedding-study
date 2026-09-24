# Android Activity Embedding: Deep Dive Architecture & Study Guide

This document serves as your technical map and slide outline for presenting how **Android Activity Embedding** works under the hood.

---

## 1. Architectural Big Picture

Activity Embedding allows multi-activity Android apps to display two or more activities simultaneously on large-screen devices (foldables, tablets, chromebooks) without rewriting the activities into fragments.

The system is structured across **4 primary layers**:

```
+-------------------------------------------------------------------------------+
| 1. App / Jetpack WindowManager Layer (androidx.window.embedding)              |
|    - SplitController, RuleController, ActivityEmbeddingController             |
|    - EmbeddingAdapter, ExtensionEmbeddingBackend                              |
+---------------------------------------+---------------------------------------+
                                        | (Dynamic Loading / Reflection)
+---------------------------------------v---------------------------------------+
| 2. WindowManager Extensions (OEM Implementation in AOSP / Vendor)             |
|    - androidx.window.extensions.embedding.SplitController                    |
|    - SplitPresenter, TaskContainer, TaskFragmentContainer                     |
|    - JetpackTaskFragmentOrganizer (extends TaskFragmentOrganizer)             |
+---------------------------------------+---------------------------------------+
                                        | (IPC: ITaskFragmentOrganizerController)
+---------------------------------------v---------------------------------------+
| 3. Framework Core Client (android.window)                                     |
|    - TaskFragmentOrganizer, TaskFragmentTransaction, TaskFragmentOperation   |
|    - TaskFragmentInfo, TaskFragmentParentInfo                                 |
+---------------------------------------+---------------------------------------+
                                        | (Binder IPC to system_server)
+---------------------------------------v---------------------------------------+
| 4. System Server / WindowManager (com.android.server.wm)                     |
|    - TaskFragmentOrganizerController (ITaskFragmentOrganizerController.Stub)  |
|    - TaskFragment (extends WindowContainer<WindowContainer>)                 |
|    - Task (extends TaskFragment)                                              |
|    - ActivityRecord, SurfaceControl Transactions                              |
+-------------------------------------------------------------------------------+
```

---

## 2. Key Source Code Map (Local Repositories)

Both repositories have been checked out locally using lightweight shallow sparse checkouts:
- **`androidx-window/`** (~17 MB)
- **`aosp-frameworks-base/`** (~46 MB)

### Layer 1: Jetpack WindowManager API & Backend
* **`androidx-window/window/window/src/main/java/androidx/window/embedding/`**
  - `ActivityEmbeddingController.kt`: Public developer API to query embedded states.
  - `SplitController.kt`: Developer entry point for checking split support and observing split state.
  - `RuleController.kt`: Manages `SplitRule`, `SplitPairRule`, `SplitPlaceholderRule`.
  - `ExtensionEmbeddingBackend.kt`: Singleton backend bridge. Dynamically detects and binds the OEM extension.
  - `EmbeddingCompat.kt` & `SafeActivityEmbeddingComponentProvider.kt`: Reflection-safe loader that verifies API level support.
  - `EmbeddingAdapter.kt`: Translates Jetpack API classes into OEM extension interface representations.

### Layer 2: WindowManager Jetpack Extension (AOSP / OEM side)
* **`aosp-frameworks-base/libs/WindowManager/Jetpack/src/androidx/window/extensions/embedding/`**
  - `SplitController.java`: Central coordinator in the app process. Tracks activity launches, handles config changes, maintains split containers.
  - `SplitPresenter.java`: Calculates layout geometry, split ratios, min dimensions, and converts split configurations to system transactions.
  - `JetpackTaskFragmentOrganizer.java`: The bridge to the OS. Extends `android.window.TaskFragmentOrganizer`. Dispatches container changes to ATMS.
  - `TaskContainer.java` & `TaskFragmentContainer.java`: Client-side bookkeeping of the Task hierarchy and sub-containers.
  - `DividerPresenter.java`: Coordinates interactive split dividers and drag handles.

### Layer 3: Framework Client APIs (`android.window`)
* **`aosp-frameworks-base/core/java/android/window/`**
  - `TaskFragmentOrganizer.java`: Base class for organizing task fragments. Registers with system server.
  - `TaskFragmentTransaction.java`: Batch transaction container for atomic operations.
  - `TaskFragmentOperation.java`: Contains atomic operation types (`OP_TYPE_CREATE_TASK_FRAGMENT`, `OP_TYPE_START_ACTIVITY_IN_TASK_FRAGMENT`, `OP_TYPE_REPARENT_ACTIVITY_TO_TASK_FRAGMENT`, `OP_TYPE_SET_RELATIVE_BOUNDS`, etc.).
  - `ITaskFragmentOrganizerController.aidl`: IPC interface to the system server organizer controller.

### Layer 4: System Server / WindowManager Core
* **`aosp-frameworks-base/services/core/java/com/android/server/wm/`**
  - `TaskFragmentOrganizerController.java`: System server receiver. Manages registered organizers, validates security tokens, applies transaction batches.
  - `TaskFragment.java`: The core container. Inherits from `WindowContainer<WindowContainer>`. Controls activity lifecycle and visibility within a sub-pane.
  - `Task.java`: **Notice that `class Task extends TaskFragment`!** The Task itself is a specialized TaskFragment representing the entire Recents tile.

### Layer 5: Reference Samples & Window Demos
* **`androidx-window/window/window-demos/demo/src/main/java/androidx/window/demo/embedding/`**
  - `ExampleWindowInitializer.kt`: Demonstrates XML and programmatic rule initialization.
  - `SplitActivityBase.java`, `SplitActivityA.kt`, `SplitActivityB.kt`: Complete demo implementations illustrating split behavior, IME interactions, and PiP transitions.

---

## 3. End-to-End Execution Flow: How a Split Happens

```
App Code: startActivity(Intent B from Activity A)
   │
   ▼
[1] Jetpack WindowManager
   - Matches Intent against registered SplitPairRule / SplitPlaceholderRule
   │
   ▼
[2] Extension SplitController (OEM layer in app process)
   - Checks device metrics (width/height vs minDimensions / splitRatio)
   - Decides whether activities should be stacked or side-by-side
   │
   ▼
[3] JetpackTaskFragmentOrganizer
   - Creates a TaskFragmentTransaction with operations:
     * OP_TYPE_CREATE_TASK_FRAGMENT (token for primary & secondary fragments)
     * OP_TYPE_SET_RELATIVE_BOUNDS (e.g. left [0, 0, 500, 1000], right [500, 0, 1000, 1000])
     * OP_TYPE_START_ACTIVITY_IN_TASK_FRAGMENT (Intent B -> Secondary Fragment)
   │
   ▼ (Binder IPC: ITaskFragmentOrganizerController)
[4] TaskFragmentOrganizerController (in system_server ATMS)
   - Validates organizer ownership & security checks (trusted vs untrusted)
   - Creates TaskFragment nodes in the Task hierarchy
   - Launches Activity B directly into the target TaskFragment
   │
   ▼
[5] WindowManager & SurfaceFlinger
   - Hierarchy updated: Task -> [TaskFragment 1 (Activity A), TaskFragment 2 (Activity B)]
   - Shell Transition executes split animation atomically (no flickering)
```

---

## 4. Key Concepts to Highlight in Your Presentation

1. **Hierarchy Refactoring (`Task extends TaskFragment`)**:
   - In older Android versions: `Task` contained `ActivityRecord`s directly.
   - In Android 12L+: `Task` extends `TaskFragment`. A `Task` can contain multiple `TaskFragment`s, which each contain `ActivityRecord`s.
2. **Client-Side Orchestration vs Server Enforcement**:
   - The layout math (50/50, 33/67, hinge avoidance) is computed on the **client side** by `SplitPresenter`.
   - The **server side** (`TaskFragmentOrganizerController`) handles atomic container creation, reparenting, and security.
3. **Atomic Transactions (`TaskFragmentTransaction`)**:
   - Prevents visual glitching: creation of fragments, reparenting of activities, and bounds updates are bundled into a single transaction and animated together.
4. **Security Model (Trusted vs. Untrusted Embedding)**:
   - Activities within the same UID are trusted by default.
   - Cross-app embedding requires `android:allowUntrustedActivityEmbedding="true"` or `android.permission.EMBED_ANY_APP_IN_UNTRUSTED_MODE` to protect against tapjacking.
5. **Interactive Dividers**:
   - The OS creates a dedicated decor surface (`OP_TYPE_CREATE_OR_MOVE_TASK_FRAGMENT_DECOR_SURFACE`) above the fragments so the app/organizer can draw draggable split handles.
