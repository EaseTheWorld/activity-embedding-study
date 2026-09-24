# Presentation Outline: Android Activity Embedding Deep Dive

**Target Audience:** Mobile Engineers / Android Platform Developers  
**Estimated Time:** 30–45 minutes  

---

### Slide 1: Title & Motivation
- **Title:** Deep Dive: How Android Activity Embedding Works Under the Hood
- **The Problem:** Modern Android devices (Foldables, Tablets, Pixel Fold, Galaxy Fold) offer large screens. Legacy apps built with multi-activity architecture appear stretched or letterboxed.
- **The Solution:** Activity Embedding allows displaying multiple independent activities side-by-side without refactoring the entire app into Fragments or Compose layouts.

---

### Slide 2: The 4-Layer Architecture
- Show diagram:
  1. **Jetpack WindowManager (`androidx.window.embedding`)**: App-facing API, rules, callbacks.
  2. **WindowManager Extensions (`androidx.window.extensions.embedding`)**: Platform/OEM implementation in app process (`SplitController`, `SplitPresenter`).
  3. **Framework Client (`android.window`)**: `TaskFragmentOrganizer`, `TaskFragmentTransaction`, `TaskFragmentOperation`.
  4. **System Server WM (`com.android.server.wm`)**: `TaskFragmentOrganizerController`, `TaskFragment`, `Task`.
- **Key Insight:** Clean separation of concerns between developer rules, OEM/device adaptations, and OS window hierarchy.

---

### Slide 3: The Paradigm Shift in WindowManager Hierarchy
- **Before Android 12L:**
  - `DisplayContent` ➔ `TaskDisplayArea` ➔ `Task` ➔ `ActivityRecord`
- **Android 12L & Beyond:**
  - `DisplayContent` ➔ `TaskDisplayArea` ➔ `Task` ➔ **`TaskFragment`** ➔ `ActivityRecord`
- **Architectural revelation in source code:**
  - `Task extends TaskFragment`!
  - A `Task` is simply a top-level container mapping to a Recents entry.
  - Sub-panes inside the task are individual `TaskFragment` containers.

---

### Slide 4: Lifecycle & Visibility in Splits
- How do activities behave in a split?
  - Both activities are `RESUMED` on Android 10+ (multi-resume).
  - Each `TaskFragment` independently manages visibility (`TASK_FRAGMENT_VISIBILITY_VISIBLE`, `TASK_FRAGMENT_VISIBILITY_INVISIBLE`, etc.).
  - What happens when a split collapses (e.g. device folded)?
    - Handled dynamically: `SplitPresenter` recalculates split rules; fragments either stack or collapse into a single visible pane.

---

### Slide 5: The Lifecycle of a Split Launch (Step-by-Step)
- Trace what happens when `startActivity()` is called:
  1. `SplitRule` / `SplitPairRule` matches the Intent.
  2. `SplitController` & `SplitPresenter` compute target bounds (e.g., 50/50 split).
  3. `JetpackTaskFragmentOrganizer` builds a `TaskFragmentTransaction`:
     - `OP_TYPE_CREATE_TASK_FRAGMENT`
     - `OP_TYPE_SET_RELATIVE_BOUNDS`
     - `OP_TYPE_START_ACTIVITY_IN_TASK_FRAGMENT`
  4. Binder call to `ITaskFragmentOrganizerController` in `system_server`.
  5. `TaskFragmentOrganizerController` atomically updates the WM hierarchy and dispatches Shell transitions.

---

### Slide 6: Security: Trusted vs. Untrusted Embedding
- Why can't any app embed any other app? **Tapjacking & Clickjacking protection.**
- **Trusted Embedding:**
  - Same UID or application package. Permitted automatically.
- **Untrusted Embedding:**
  - Different UIDs.
  - Target activity must explicitly opt in via manifest:
    ```xml
    <activity
        android:name=".TargetActivity"
        android:allowUntrustedActivityEmbedding="true" />
    ```
  - Or calling app requires privileged permission: `EMBED_ANY_APP_IN_UNTRUSTED_MODE`.

---

### Slide 7: Interactive Dividers & Decor Surfaces
- How do users drag to resize the split?
  - `OP_TYPE_CREATE_OR_MOVE_TASK_FRAGMENT_DECOR_SURFACE`: OS allocates a decor SurfaceControl owned by the Task.
  - `DividerPresenter`: Draws the interactive handle, listens to touch gestures, and pushes updated relative bounds to the organizer.

---

### Slide 8: Summary & Best Practices
- Takeaways:
  - Declarative XML rules vs. programmatic `RuleController`.
  - Back-navigation strategies (`finishPrimaryWithSecondary`, `finishSecondaryWithPrimary`).
  - Hinge and fold awareness (`FoldingFeature` support).
  - Where to explore in code: Check `aosp-frameworks-base/` and `androidx-window/`.
- Q&A.
