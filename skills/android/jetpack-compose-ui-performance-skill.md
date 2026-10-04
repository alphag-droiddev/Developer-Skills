# Jetpack Compose Performance Audit & Code Quality Skill

An actionable rule-set, architectural guideline, and decision workflow for AI agents and Android developers to audit Jetpack Compose code, detect improper side effects, eliminate performance anti-patterns, enforce code quality standards, and verify UI performance using profiling tools and Android Developer Options.

---

## 🎯 Skill Objectives
1. **Audit Side Effects & Lifecycle Safety**: Detect misused Compose side-effect APIs (`LaunchedEffect`, `SideEffect`, `DisposableEffect`, `rememberUpdatedState`) and hoist business logic to the `ViewModel`.
2. **Detect Performance Anti-Patterns**: Identify unnecessary recompositions, main-thread blocking, missing list keys, unlambda-ified modifier reads, stability leaks, and backwards writes.
3. **Enforce Stability & Compiler Configuration**: Utilize Strong Skipping Mode, Compose Compiler Stability Configuration files, and Immutable Collections.
4. **Enforce Compose Code Quality Standards**: Enforce Unidirectional Data Flow (UDF), lifecycle-aware state collection (`collectAsStateWithLifecycle`), and `@NonRestartableComposable` optimizations.
5. **Guide Performance Testing & Profiling**: Explain how to measure frame render times, recomposition counts, overdraw, and compiler stability reports using Android Studio and device Developer Options.

---

## 🔍 Phase 1: Static Code Audit & Code Quality Checklist

### 1. Side Effects & Logic Hoisting
| Anti-Pattern / Smell | Problem | Action / Remediation |
| :--- | :--- | :--- |
| **Business logic inside `LaunchedEffect`** | Keeps state management inside UI composition, making it non-testable and prone to lifecycle bugs. | **Hoist to ViewModel**: Move state mutations, network requests, and domain logic into ViewModel `viewModelScope`. |
| **State modification directly in Composition body** | Modifying `MutableState` during composition body execution causes infinite recomposition loops. | Wrap mutation inside an event handler (e.g., `onClick`) or inside `LaunchedEffect(key)` / ViewModel action. |
| **Backwards Writes** | Writing to a state object during composition after it has already been read in that frame causes composition invalidation loops. | Never mutate state during composition execution. Move updates to event callbacks or ViewModel. |
| **Stale closures in `LaunchedEffect`** | Capturing parameters/lambdas in `LaunchedEffect` without keying on them or using `rememberUpdatedState`. | Use `rememberUpdatedState` for long-lived effect callbacks, or key the effect appropriately. |
| **Missing cleanup in `DisposableEffect`** | Setting up observers or listeners without removing them in `onDispose`. | Always register unbind/unregister logic inside `onDispose { ... }`. |
| **Collecting Flow via `collectAsState()` on Android** | Keeps Flow collection active even when app is in the background, wasting CPU and battery. | Use **`collectAsStateWithLifecycle()`** from `androidx.lifecycle.compose`. |

---

### 2. Deferred State Reads & Layout Efficiency
| Anti-Pattern / Smell | Problem | Action / Remediation |
| :--- | :--- | :--- |
| **Direct state read in layout modifiers** (`Modifier.offset(x = state.value.dp)`) | Reading rapidly updating state (drag offset, scroll position, animations) forces full **Recomposition** on every frame. | **Use Lambda Modifiers**: `Modifier.offset { IntOffset(...) }` or `Modifier.graphicsLayer { ... }` to defer reads to **Layout/Draw phase**. |
| **High-frequency calculation without `derivedStateOf`** | Reading fast-changing state (`LazyListState.firstVisibleItemIndex`) repeatedly executes calculations during recomposition. | Wrap calculation in `remember { derivedStateOf { ... } }` so UI re-triggers only when the final result changes. |
| **Overusing Intrinsic Measurement (`IntrinsicSize.Min/Max`)** | Intrinsic measurements force two measure passes on child layout nodes, compounding depth penalty. | Limit intrinsic measurements to shallow, necessary UI branches (e.g., matching row heights). |

---

### 3. Stability, Strong Skipping & Compiler Reports
| Anti-Pattern / Smell | Problem | Action / Remediation |
| :--- | :--- | :--- |
| **Unstable class parameters** (e.g., `Bitmap`, `java.time.LocalDateTime`, standard `List`/`Set`/`Map` interfaces from 3rd party or external modules) | Compose compiler marks composable as **unskippable** (unless Strong Skipping mode compares by instance equality). | 1. Annotate custom data classes with `@Stable` or `@Immutable`.<br>2. Use Kotlin Immutable Collections (`ImmutableList`).<br>3. Enable Compose Compiler **Stability Configuration File** for external classes. |
| **Multi-module stability breakdown** | Data layer model classes defined in non-Compose modules are treated as unstable by the Compose compiler. | Add `runtime` dependency to data modules, enable Compose compiler plugin, or list types in `stability_config.conf`. |
| **Unstable lambda allocations in lists** | Instantiating new lambda instances every recomposition breaks child skipping when Strong Skipping is off. | Pass stable function references (`onAction = viewModel::onAction`) or rely on Kotlin 2.0+ Strong Skipping. |

---

### 4. Lazy Layouts (`LazyColumn`, `LazyRow`, `LazyVerticalGrid`)
| Anti-Pattern / Smell | Problem | Action / Remediation |
| :--- | :--- | :--- |
| **Missing `key` in `items()`** | Compose tracks items by positional index. Inserting/deleting/reordering items causes mass recompositions and state loss. | Always specify a stable, unique key: `items(list, key = { it.id })`. |
| **Heavy calculations inside list item items scope** | Re-runs heavy operations during list scroll. | Pre-compute or map data in ViewModel before emitting to StateFlow. |
| **Nested scrolling without explicit constraints** | Placing a `LazyColumn` inside a `ScrollView` or unbounded `Column` breaks layout recycling and causes memory lag. | Use a single `LazyColumn` with multiple `item { }` / `items { }` blocks. |

---

### 5. Code Quality & Architecture Standards
| Guideline / Standard | Benefit | Implementation |
| :--- | :--- | :--- |
| **Single Immutable UI State Class** | Ensures predictable Unidirectional Data Flow (UDF) and easier unit testing. | Expose a single `StateFlow<MyScreenUiState>` from `ViewModel` rather than multiple discrete state flows. |
| **`@NonRestartableComposable` for Leaf Composables** | Saves runtime composition tracking overhead for simple leaf composables (e.g. static icons, text wrappers). | Annotate composables that directly delegate to another skippable composable without reading state. |
| **Main-Thread Safety in Repositories/ViewModels** | Prevents UI freezing during bitmap processing, disk I/O, or JSON parsing. | Always execute blocking work on `withContext(Dispatchers.IO)`. |

---

## 🛠️ Phase 2: Refactoring Patterns & Examples

### Rule A: Deferring State Reads
❌ **BEFORE (Triggers Recomposition Every Frame)**
```kotlin
// BAD: Recomposes on every pixel change during dragging/animating
val offset by animateFloatAsState(...)
Box(
    modifier = Modifier.offset(x = offset.dp, y = 0.dp)
)
```

✅ **AFTER (Defers to Layout Phase — 0 Recompositions)**
```kotlin
// GOOD: Skips composition, reads state directly in layout phase
val offset by animateFloatAsState(...)
Box(
    modifier = Modifier.offset {
        IntOffset(x = offset.roundToInt(), y = 0)
    }
)
```

---

### Rule B: Capturing Callback State in Effects (`rememberUpdatedState`)
❌ **BEFORE (Restarts LaunchedEffect when callback changes or loses latest value)**
```kotlin
@Composable
fun Timer(onTimeout: () -> Unit) {
    LaunchedEffect(Unit) {
        delay(5000)
        onTimeout() // Might call stale callback if onTimeout changed during delay!
    }
}
```

✅ **AFTER (Always calls updated callback without restarting effect)**
```kotlin
@Composable
fun Timer(onTimeout: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)
    LaunchedEffect(Unit) {
        delay(5000)
        currentOnTimeout() // Guaranteed to execute latest callback instance
    }
}
```

---

### Rule C: Stabilizing External Types via Compiler Configuration
❌ **BEFORE (Requires wrapping 3rd-party types in custom data classes)**
```kotlin
// java.time.LocalDateTime is treated as unstable by default
data class Event(
    val id: String,
    val date: LocalDateTime // Unstable parameter!
)
```

✅ **AFTER (Use Stability Configuration File `stability_config.conf`)**
```conf
// stability_config.conf in project root
java.time.LocalDateTime
kotlin.collections.*
com.example.domain.model.**
```

In `build.gradle.kts`:
```kotlin
composeCompiler {
    stabilityConfigurationFile = rootProject.layout.projectDirectory.file("stability_config.conf")
}
```

---

### Rule D: Strong Skipping Mode (Kotlin 2.0+ Default)
Strong Skipping is enabled by default in Kotlin 2.0.20+.
- **Unstable parameters**: Compared using instance equality (`===`).
- **Stable parameters**: Compared using object equality (`.equals()`).
- **Lambdas**: Automatically wrapped in `remember(captures)`.
- **Opt-out**: Use `@NonSkippableComposable` or `@DontMemoize` when explicit re-evaluation is needed.

---

## 🧪 Phase 3: Testing, Diagnostics & Verifying Compose Performance

### 1. Compose Compiler Stability Reports
Enable stability report generation in `build.gradle.kts`:
```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    metricsDestination = layout.buildDirectory.dir("compose_compiler")
}
```
Run `./gradlew assembleRelease` to generate:
- **`modulename-classes.txt`**: Reports stability of classes (`stable` vs `unstable`).
- **`modulename-composables.txt`**: Reports whether composables are `restartable` and `skippable`.

---

### 2. Android Studio Layout Inspector (Recomposition Counts)
* **Goal**: Identify components that recompose unnecessarily during user interactions.
* **How to use**:
  1. Connect a device/emulator and run app in **debug mode**.
  2. Open **Tools -> Layout Inspector**.
  3. Turn on **Recomposition Counts** overlay.
  4. Perform UI actions (scroll, drag, swipe).
  5. Check **Recomposition Count** vs **Skip Count**:
     - ⚠️ **High Recomposition Count + Low Skip Count** = Unstable parameters or unlambda-ified state reads.
     - ✅ **0 Recomposition Count during scrolling/dragging** = State reads successfully deferred to layout/draw.

---

### 3. Android Studio CPU Profiler & System Trace
* **Goal**: Detect Main thread jank, dropped frames, and long composition times.
* **How to use**:
  1. Open **View -> Tool Windows -> Profiler**.
  2. Start a CPU Trace: Choose **System Trace**.
  3. Perform UI interaction (e.g. fast scrolling).
  4. Look for long frames (bars exceeding 16.6ms for 60Hz or 8.3ms for 120Hz).
  5. Inspect Compose trace sections:
     - `Recompose`: Time spent executing composables.
     - `Measure/Layout`: Time spent measuring layout node dimensions.
     - `Draw`: Time spent issuing GPU drawing commands.

---

### 4. On-Device Developer Options
Enable under `Settings -> System -> Developer Options`:

#### A. Profile HWUI Rendering (GPU Rendering Bars)
* **Path**: `Developer Options -> Profile HWUI Rendering -> On screen as bars`.
* **Interpretation**:
  - Horizontal **green line** represents the 16.6ms target frame time budget (60 FPS).
  - Vertical bars represent time taken per frame.
  - Bars exceeding the green line indicate dropped frames (jank).

#### B. Debug GPU Overdraw
* **Path**: `Developer Options -> Debug GPU Overdraw -> Show overdraw areas`.
* **Interpretation**:
  - **No color**: Pixel painted once (ideal).
  - **Blue**: Overdrawn 1x (acceptable).
  - **Green**: Overdrawn 2x.
  - **Pink/Red**: Overdrawn 3x or 4x+ (Requires removal of unnecessary container backgrounds).

---

### 5. Production Measurement: JankStats & Release Builds
* **Release Mode**: Never measure final performance on a Debug build! Debug builds disable R8 optimizations and Compose compiler runtime optimizations. Run `./gradlew assembleRelease`.
* **JankStats**: Integrate Jetpack `JankStats` library to log frame drop metrics in production alongside analytics/bug reporting.
* **Baseline Profiles**: Generate Baseline Profiles via `androidx.profileinstaller` and Macrobenchmark to eliminate cold startup jank and first-frame compilation delays.
