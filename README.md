# Android Activity Embedding Deep Dive

[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-Live%20Presentation-brightgreen)](https://easetheworld.github.io/activity-embedding-study/)
[![Android](https://img.shields.io/badge/Android-12L%20%2B-3DDC84?logo=android&logoColor=white)](https://developer.android.com/guide/topics/large-screens/activity-embedding)
[![Jetpack WindowManager](https://img.shields.io/badge/Jetpack-WindowManager%201.3+-4285F4?logo=google&logoColor=white)](https://developer.android.com/jetpack/androidx/releases/window)

A comprehensive architectural deep-dive into how **Android Activity Embedding** works across all 4 system layers: Jetpack WindowManager, OEM WindowManager Extensions, Framework Core (`android.window`), and System Server (`com.android.server.wm`).

---

## 🌐 Live GitHub Pages Documentation & Presentation

👉 **[View the Interactive Web Guide & Slide Deck](https://easetheworld.github.io/activity-embedding-study/)**

Features included in the GitHub Pages site:
* **Interactive 4-Layer Architecture Visualizer**
* **WindowManager Hierarchy Comparison (`Task extends TaskFragment`)**
* **End-to-End Execution Sequence Flow**
* **Security Model (Trusted vs. Untrusted Embedding)**
* **Built-in Presentation Slide Mode** (Press `Presentation Slide Mode` or use `←` / `→` arrow keys to present directly to colleagues)

---

## 🏛️ Architecture at a Glance

Activity Embedding is coordinated across four distinct tiers:

```
┌────────────────────────────────────────────────────────────────────────┐
│ 1. App / Jetpack WindowManager (androidx.window.embedding)             │
│    - SplitController, RuleController, ActivityEmbeddingController      │
├───────────────────────────────────┬────────────────────────────────────┤
                                    │ (Dynamic Loading / Reflection)
┌───────────────────────────────────▼────────────────────────────────────┐
│ 2. WindowManager Extensions (AOSP / OEM Implementation)                │
│    - SplitController, SplitPresenter, JetpackTaskFragmentOrganizer     │
├───────────────────────────────────┬────────────────────────────────────┤
                                    │ (IPC: ITaskFragmentOrganizerController)
┌───────────────────────────────────▼────────────────────────────────────┐
│ 3. Framework Client Core (android.window)                              │
│    - TaskFragmentOrganizer, TaskFragmentTransaction, Operations        │
├───────────────────────────────────┬────────────────────────────────────┤
                                    │ (Binder IPC to system_server)
┌───────────────────────────────────▼────────────────────────────────────┐
│ 4. System Server / WindowManager Core (com.android.server.wm)          │
│    - TaskFragmentOrganizerController, TaskFragment, Task               │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 📂 Source Code Included in this Repository

This repository includes targeted, complete source code from both AndroidX and AOSP to enable fast offline study:

### 1. [`androidx-window/`](androidx-window)
Extracted from `platform/frameworks/support` (AndroidX):
- `window/window/src/main/java/androidx/window/embedding/`: High-level developer APIs, rule controllers, and backend bridges.
- `window/window-demos/demo/src/main/java/androidx/window/demo/embedding/`: Real-world working sample applications demonstrating split rules, IME, PiP, and fold transitions.

### 2. [`aosp-frameworks-base/`](aosp-frameworks-base)
Extracted from `platform/frameworks/base` (AOSP):
- `libs/WindowManager/Jetpack/src/androidx/window/extensions/embedding/`: Platform default implementation of the OEM extensions (`SplitController.java`, `SplitPresenter.java`, `JetpackTaskFragmentOrganizer.java`).
- `core/java/android/window/`: Client-side framework contracts (`TaskFragmentOrganizer.java`, `TaskFragmentTransaction.java`, `TaskFragmentOperation.java`).
- `services/core/java/com/android/server/wm/`: WindowManager system server core (`TaskFragmentOrganizerController.java`, `TaskFragment.java`, `Task.java`).
- `libs/WindowManager/Shell/`: Shell transition handlers and organizer animations.

---

## 🔑 Key Technical Insights

1. **Hierarchy Refactoring (`Task extends TaskFragment`)**:
   In Android 12L+, the WindowManager hierarchy was redesigned. `Task` is no longer a direct leaf container of `ActivityRecord`s; rather, it extends `TaskFragment`. A `Task` can contain multiple `TaskFragment`s, which can be placed side-by-side or stacked.
2. **Atomic Batch Transactions (`TaskFragmentTransaction`)**:
   Container creation, bounds assignment, and activity reparenting are dispatched to ATMS in a single parcelable transaction, ensuring zero visual flickering during transitions.
3. **Security (Trusted vs Untrusted)**:
   Same-UID activities are trusted by default. Cross-app embedding requires target activities to declare `android:allowUntrustedActivityEmbedding="true"` in their `AndroidManifest.xml` to prevent tapjacking.
4. **Inversion of Control (IoC) via `compileOnly` & System Shared Library**:
   The unbundled Jetpack library (`androidx.window:window`) links against the specification interface (`WindowExtensions`, `ActivityEmbeddingComponent`) as `compileOnly`, stripping the `.class` files from the final APK. The device's `/system_ext/framework/androidx.window.extensions.jar` provides both the interface and the concrete implementation (`SplitController`) at runtime via `<uses-library>`, eliminating duplicate class conflicts and preventing `ClassCastException`s.

---

## 📑 Study & Presentation Guides

- [ACTIVITY_EMBEDDING_ARCHITECTURE.md](ACTIVITY_EMBEDDING_ARCHITECTURE.md): Full technical breakdown and sequence tracing.
- [SLIDE_DECK_OUTLINE.md](SLIDE_DECK_OUTLINE.md): Structured slide outline for technical presentations.
