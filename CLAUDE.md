# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

Use the Gradle wrapper (run from project root):

```powershell
.\gradlew assembleDebug          # Build debug APK
.\gradlew build                  # Full build
.\gradlew clean                  # Clean build artifacts
.\gradlew installDebug           # Build and install on connected device/emulator
.\gradlew test                   # Run unit tests
.\gradlew connectedAndroidTest   # Run instrumented tests (requires device/emulator)
.\gradlew check                  # Run lint + all checks
```

## Architecture

**MVVM + Jetpack Compose** on Android (minSdk 24, targetSdk 36).

- `MainActivity` is the single activity; all UI is Compose-based
- Navigation is declared in `ui/navigation/NavGraph.kt` using Compose Navigation; routes are defined as sealed class entries in `ui/navigation/Screen.kt`
- Screens live under `ui/view/component/screen/<feature>/` — each feature folder contains the screen composable, a `component/` subfolder for sub-composables, and a `util/` subfolder for state models and validators
- ViewModels live in `ui/view/view_model/`; currently only `LoginViewModel` exists
- Theme (colors, typography, dark/light) is in `ui/theme/`

### Screen inventory

| Route | Screen | Notes |
|---|---|---|
| `login_screen` | `LoginScreen` | Email + password, Google sign-in button (not wired) |
| `register_screen` | `RegisterScreen` | Full name, email, confirm password |
| `home_screen` | `HomeScreen` | Stats cards (jobs applied, XP, available jobs) |
| `job_listing_screen` | `JobListingScreen` | Searchable list of job applications |
| `job_details_screen` | `JobDetailsScreen` | Detail view for a single job |
| `dashboard_screen` | `DashboardScreen` | Metrics cards + add-job form |
| `profile_screen` | `ProfileScreen` | Theme toggle, user prefs |

## Key Technical Details

- **Compose BOM**: `2024.09.00` — use BOM-managed versions for all `androidx.compose.*` dependencies
- **Kotlin**: 2.2.10; **AGP**: 9.1.0
- **Version catalog**: `gradle/libs.versions.toml` is the single source of truth for dependency versions
- Input validation lives in `ui/view/component/screen/LoginScreen/util/AuthValidator.kt`
- App string resources are in `res/values/strings.xml` (59 entries)

## Known Issues to Be Aware Of

- `app/build.gradle.kts` has a typo on the navigation dependency: `"adnroidx.navigation:..."` — should be `"androidx.navigation:navigation-compose:..."`. Fix this before navigation will compile.
- `NavGraph.kt` references `hiltViewModel()` but Hilt is not in the dependency list; replace with a plain `viewModel()` call or add Hilt setup.
- Data is entirely hardcoded — no Room database, Retrofit, or other persistence layer exists yet.
