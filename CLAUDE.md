# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Google's Android Architecture Samples TODO app (Compose branch): a reference sample, not a
production app. Modules: `:app` (everything) and `:shared-test` (fakes and Hilt test modules
used by both `test` and `androidTest`). Requires JDK 17.

## Commands

Verified locally:

```bash
./gradlew assembleDebug
./gradlew :app:testDebugUnitTest
./gradlew :app:testDebugUnitTest --tests "*.TasksViewModelTest"            # one class
./gradlew :app:testDebugUnitTest --tests "*.TasksViewModelTest.<method>"   # one method
./gradlew :app:lintDebug          # report: app/build/reports/lint-results-debug.html
```

Needs a device/emulator (not verified locally; CI runs this on API 29):

```bash
./gradlew :app:connectedCheck
```

Notes:
- No product flavors exist. The README's `mock`/`prod` flavors are outdated; use `debug`/`release`.
- Lint has a baseline of ~170 warnings and 0 errors. Don't introduce new errors.
- `gradle/init.gradle.kts` (Spotless) is currently broken (`userData` was removed in Spotless
  6.25.0). Don't rely on it, and don't fix it unless asked.

## Architecture

Single activity, Compose only: `TodoApplication` (Hilt) → `TodoActivity` → `TodoNavGraph`.

- **Navigation**: string routes in `TodoNavigation.kt`. Add/edit/delete results go back to the
  tasks screen as the `userMessage` route argument, not as a shared result API.
- **UI state**: each screen has a `@HiltViewModel` that exposes a single `StateFlow<XUiState>`,
  built with `combine(...)` + `stateIn(viewModelScope, WhileUiSubscribed, ...)`. Loading/error
  use `util/Async`. One-off messages are `userMessage: Int?` in the state, cleared when the screen
  calls `snackbarMessageShown()`. Follow this pattern in new screens.
- **Data**: `TaskRepository` → `DefaultTaskRepository`, offline-first. The UI observes Room. `refresh()`
  replaces local data with network data. Writes call `saveTasksToNetwork()` without waiting, on the
  `@ApplicationScope` scope. The "network" (`TaskNetworkDataSource`) is an in-memory fake;
  there is no real networking.
- **Models**: `Task` (external) / `LocalTask` (Room) / `NetworkTask`, mapped in
  `data/ModelMappingExt.kt`. A new field must go through all three plus the mappings.
- **DI**: Hilt. Get dispatchers through `@IoDispatcher` / `@DefaultDispatcher`, never `Dispatchers.*`
  directly, so tests can replace them.

## Testing

- JVM tests use JUnit4, Truth, coroutines-test, `MainCoroutineRule` and the fakes in
  `:shared-test`. No Robolectric or Mockito.
- Device tests are `@HiltAndroidTest`. `CustomTestRunner` installs `HiltTestApplication`.
- `:shared-test` replaces `RepositoryModule` and `DatabaseModule` with `@TestInstallIn` modules.
  If you rename or split a production Hilt module, update its test replacement too.

## Rules for changes

- Keep changes scoped to the task. No unrelated refactors, renames, reformatting, file moves or
  "cleanup" of nearby code unless asked.
- Don't change the architecture (layers, state pattern, navigation style, DI layout) unless
  explicitly asked.
- Don't add, remove or upgrade dependencies, Gradle plugins, the Gradle wrapper or SDK levels in
  `gradle/libs.versions.toml` or the build scripts without explicit approval. If a task seems to need
  one, stop and ask. (Renovate handles routine version bumps.)
- Don't change build config (`gradle.properties`, build types, ProGuard rules, CI workflows)
  unless the task is about them.
- New source files need the Apache 2.0 license header used in the other files.
- Before finishing, run `:app:testDebugUnitTest` and `:app:lintDebug` and report the results.