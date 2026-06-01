# Defense Study Notes

---

## Q1: Why does LoginScreen have 3 functions, two with the same name where one is private?

This is the **State Hoisting + Separation of Concerns** pattern in Jetpack Compose.

### The three functions:

**1. `LoginScreen` (public, line 39)** — the "smart" composable
- Takes a `viewModel` and navigation callbacks
- Collects the ViewModel's state flows (`uiState`, `emailError`, etc.)
- Handles side effects (`LaunchedEffect` to navigate on success)
- Delegates all rendering to the private overload

**2. `private fun LoginScreen` (line 72)** — the "dumb" / stateless composable
- Takes only plain data and lambdas — no ViewModel, no flows
- Pure UI: just renders what it's given
- Has no knowledge of where data comes from

**3. `LoginScreenPreview` (line 151)** — Android Studio preview
- Calls the private overload directly with hardcoded values
- Lets you see the UI in the IDE without running the app

### Why two functions with the same name?

The public one is what the navigation graph wires up — it knows about ViewModels and navigation. The private one is intentionally isolated so it can be:
- **Previewed** without a ViewModel (which requires an Activity/Hilt to exist)
- **Tested** in isolation with just plain values
- **Reused** if the same layout is ever needed driven by a different data source

This pattern is called **"Hoisted State"** — state management is lifted to the public composable, keeping the rendering layer pure and stateless.

---

## Q2: What is Hilt?

**Hilt** is a dependency injection (DI) framework for Android, built on top of Dagger.

### What is Dependency Injection?

Instead of a class creating its own dependencies, something external provides them.

Without DI:
```kotlin
class LoginViewModel {
    val repo = UserRepository() // creates its own dependency
}
```

With DI (Hilt provides it):
```kotlin
class LoginViewModel @Inject constructor(
    val repo: UserRepository // Hilt creates and injects this
) : ViewModel()
```

### Why use Hilt?
- You don't manually create ViewModels, repositories, or services — Hilt does it for you
- Makes testing easier — you can swap real implementations for fakes
- Manages the lifecycle of objects (e.g. a repository lives as long as the app, not recreated every screen)

### In this project

`LoginScreen.kt` calls `hiltViewModel()` to get the `LoginViewModel` — it tells Hilt: *"give me the ViewModel for this screen, and inject all its dependencies."*

However, **Hilt is not actually set up in the project dependencies**, so this will crash at runtime. It either needs to be properly added or replaced with a plain `viewModel()` call.

---

## Q3: What is a ViewModel? What is it used for? Walk me through LoginViewModel.

### What is a ViewModel?

A `ViewModel` is an Android Architecture Component that **holds and manages UI-related data** across configuration changes (like screen rotation). When you rotate the phone, the Activity/Composable is destroyed and recreated — but the ViewModel survives.

- **Without ViewModel:** User types email → rotates phone → email is gone
- **With ViewModel:** User types email → rotates phone → email is still there

### Role in MVVM

- **Model** — `AuthRepository` (data layer, talks to Firebase)
- **ViewModel** — `LoginViewModel` (business logic, holds state)
- **View** — `LoginScreen` (Compose UI, just renders)

The ViewModel sits between the UI and the data. The UI never directly touches the repository.

---

### LoginViewModel — piece by piece

#### 1. Sealed class `LoginUiState`
```kotlin
sealed class LoginUiState {
    object Init : LoginUiState()
    object Loading : LoginUiState()
    object Success : LoginUiState()
    data class Error(val message: String) : LoginUiState()
}
```
Represents the **4 possible states** of a login operation. `sealed` means no class outside this file can extend it — the compiler knows every possible state, so `when` expressions are exhaustive.

#### 2. `@HiltViewModel` + `@Inject constructor`
```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authRepository: AuthRepository
) : ViewModel()
```
Tells Hilt to create this ViewModel and inject `AuthRepository` automatically. You never call `LoginViewModel(...)` yourself.

#### 3. Two separate state flows
```kotlin
private val _uiState = MutableStateFlow<LoginUiState>(LoginUiState.Init)
val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

private val _state = MutableStateFlow(LoginState())
val state = _state.asStateFlow()
```
- `_state` — holds **form data**: email, password, passwordVisible
- `_uiState` — holds **operation status**: Init / Loading / Success / Error

Separated because they represent different concerns. `_` prefix = private mutable, the public one is read-only (`StateFlow`).

#### 4. Derived flows — validation
```kotlin
val emailError = _state.map {
    AuthValidators.validateEmail(it.email)
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
```
Every time `_state` changes, `emailError` automatically recomputes. The UI doesn't call validate manually — it just observes `emailError`.

`stateIn` converts a cold Flow into a `StateFlow`. `WhileSubscribed(5000)` keeps the flow alive 5 seconds after the last subscriber disappears (handles screen rotation gracefully).

```kotlin
val isLoginEnabled = combine(_state, emailError, passwordError) { s, eErr, pErr ->
    eErr == null && pErr == null && s.email.isNotBlank() && s.password.isNotBlank()
}
```
`combine` merges 3 flows — the button is only enabled when both fields are filled and both validations pass.

#### 5. Event functions
```kotlin
fun onEmailChange(email: String) = _state.update { it.copy(email = email) }
fun onPasswordChange(password: String) = _state.update { it.copy(password = password) }
fun togglePasswordVisibility() = _state.update { it.copy(passwordVisible = !it.passwordVisible) }
```
`update` + `copy` is the standard way to update a `data class` inside a `StateFlow` immutably — you make a modified copy rather than mutating in place.

#### 6. The `login()` function
```kotlin
fun login() {
    viewModelScope.launch {
        _uiState.update { LoginUiState.Loading }
        when (val result = authRepository.signIn(...)) {
            is NetworkResult.Success -> _uiState.update { LoginUiState.Success }
            is NetworkResult.Error   -> _uiState.update { LoginUiState.Error(result.message) }
            else -> Unit
        }
    }
}
```
- `viewModelScope.launch` runs a coroutine tied to the ViewModel's lifecycle — cancelled automatically if the ViewModel is destroyed
- Sets state to `Loading` first so the UI can show a spinner
- Calls the repository, updates state based on the result

---

### Likely defense questions

| Question | Key answer |
|---|---|
| Why does ViewModel survive rotation? | Android framework holds it separately from the Activity |
| Why two state flows instead of one? | Separation of concerns — form data vs. operation status |
| What is `sealed class` for? | Exhaustive, compile-safe state representation |
| What is `MutableStateFlow` vs `StateFlow`? | Mutable = writable (private), StateFlow = read-only (exposed to UI) |
| Why `viewModelScope`? | Coroutine tied to ViewModel lifecycle, auto-cancelled on destroy |
| What does `stateIn` do? | Converts a Flow to a StateFlow so the UI can collect it |
| What does `combine` do? | Merges multiple flows into one derived value |
| Why `copy()` instead of direct mutation? | `LoginState` is a `data class` — immutable updates via copy |
| What is `WhileSubscribed(5000)`? | Keeps flow active 5s after last subscriber, handles rotation |

---

## Q4: What is the Model layer? Walk me through it.

The model layer has three distinct parts: Remote (Firebase), Local (Room), and Repositories that bridge them.

---

### Part 1: Repository Pattern

A **Repository** abstracts data sources. The ViewModel doesn't know or care whether data comes from Firebase, Room, or a network API — it just calls the repository.

Always: **interface** first, then **implementation**:
```
AuthRepository (interface)  ←  LoginViewModel talks to this
      ↑
AuthRepositoryImpl          ←  actual Firebase logic lives here
```

**Why interface + impl?**
- Swap the implementation without touching the ViewModel
- Inject a fake implementation in tests
- Hilt knows to bind `AuthRepositoryImpl` whenever `AuthRepository` is requested (wired in `RepositoryModule`)

---

### Part 2: `NetworkResult<T>` — the result wrapper

```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val message: String, val code: Int? = null) : NetworkResult<Nothing>()
    object Loading : NetworkResult<Nothing>()
}
```

Instead of throwing exceptions across layers, every async operation returns a `NetworkResult`. The ViewModel `when`-switches on it safely. `out T` makes it covariant — a `NetworkResult<FirebaseUser>` is also a `NetworkResult<Any>`.

---

### Part 3: `AuthRepositoryImpl` — Firebase Auth

```kotlin
override suspend fun signIn(email: String, password: String): NetworkResult<FirebaseUser> =
    try {
        val result = firebaseAuth.signInWithEmailAndPassword(email, password).await()
        val user = result.user ?: throw Exception("Sign-in succeeded but user is null")
        NetworkResult.Success(user)
    } catch (e: Exception) {
        NetworkResult.Error(e.message ?: "Sign-in failed")
    }
```

- `suspend` — coroutine function, can be paused without blocking a thread
- `.await()` — converts Firebase's `Task<T>` (callback-based) into a suspendable call
- `try/catch` wraps everything into `NetworkResult` so no exception leaks to the ViewModel

---

### Part 4: Room — Local Database

Room is Android's SQLite ORM. Three parts:

**Entity** = a database table:
```kotlin
@Entity(tableName = "job_applications")
data class JobApplicationEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String, val company: String,
    val status: String, val dateApplied: String
)
```

**DAO** = Data Access Object, defines the queries:
```kotlin
@Dao
interface JobApplicationDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertJob(job: JobApplicationEntity): Long

    @Query("SELECT * FROM job_applications ORDER BY id DESC")
    fun getAllJobs(): Flow<List<JobApplicationEntity>>
}
```
- `suspend` functions = one-shot operations (insert, update, delete)
- `Flow` return = live stream — UI automatically updates when DB changes

**Database** = ties it all together:
```kotlin
@Database(entities = [UserEntity::class, JobApplicationEntity::class, ...], version = 2)
abstract class AppDatabase : RoomDatabase() {
    abstract fun jobApplicationDao(): JobApplicationDao
}
```

---

### Part 5: Hilt DI Modules

**`DatabaseModule`** — tells Hilt how to create Room:
```kotlin
@Provides @Singleton
fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
    Room.databaseBuilder(context, AppDatabase::class.java, "job_manager_db").build()
```
`@Singleton` = only one instance for the whole app lifetime.

**`RepositoryModule`** — tells Hilt which impl to use for each interface:
```kotlin
@Binds @Singleton
abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
```
Whenever a ViewModel asks for `AuthRepository`, Hilt injects `AuthRepositoryImpl`.

---

### Full data flow for login

```
User taps Login
  → LoginScreen calls viewModel.login()
  → LoginViewModel sets uiState = Loading
  → LoginViewModel calls authRepository.signIn(email, password)
  → AuthRepositoryImpl calls Firebase signInWithEmailAndPassword().await()
  → Firebase responds
  → AuthRepositoryImpl returns NetworkResult.Success or .Error
  → LoginViewModel updates uiState = Success or Error
  → LoginScreen observes uiState, navigates or shows error
```

---

### Likely defense questions

| Question | Key answer |
|---|---|
| Why use a Repository? | Abstracts data source from ViewModel; swap impl without changing ViewModel |
| Why interface + implementation? | Decoupling; enables testing with fake data |
| What is `suspend`? | Marks a function that can pause without blocking a thread (coroutine) |
| What is `.await()`? | Converts Firebase Task (callback) into a suspendable call |
| What is Room? | SQLite ORM — maps Kotlin data classes to database tables |
| What is a DAO? | Interface defining SQL queries; Room generates the implementation |
| What is a `Flow` in a DAO? | Live stream — emits a new value every time the DB row changes |
| Why `@Singleton` on the DB? | One shared DB instance across the whole app |
| What does `@Binds` do in Hilt? | Tells Hilt: when interface X is requested, inject implementation Y |
| What does `@Provides` do? | Tells Hilt how to construct an object it can't construct automatically |
| What is `NetworkResult`? | Sealed class wrapping async results — avoids throwing exceptions across layers |

---

## Q5: What are Coroutines and Flow?

### Coroutines

#### What is a coroutine?
A coroutine is a piece of code that can be **paused and resumed** without blocking a thread.

**Without coroutines (blocking):**
```kotlin
val result = firebaseAuth.signIn(email, password) // freezes UI thread
```
**With coroutines (non-blocking):**
```kotlin
val result = firebaseAuth.signInWithEmailAndPassword(email, password).await() // suspends, UI stays responsive
```
The thread is free while waiting. When Firebase responds, the coroutine resumes.

#### `suspend` keyword
Marks a function that can be paused. Can only be called from another `suspend` function or a coroutine.
```kotlin
// One-shot operations — do the thing, return, done
override suspend fun signIn(email: String, password: String): NetworkResult<FirebaseUser>
suspend fun insertJob(job: JobApplicationEntity): Long
```

#### `viewModelScope`
Built-in scope on every ViewModel. Coroutines launched in it are **automatically cancelled** when the ViewModel is destroyed — no memory leaks.
```kotlin
fun login() {
    viewModelScope.launch {   // fire and forget
        _uiState.update { LoginUiState.Loading }
        when (val result = authRepository.signIn(...)) {
            is NetworkResult.Success -> _uiState.update { LoginUiState.Success }
            is NetworkResult.Error   -> _uiState.update { LoginUiState.Error(result.message) }
            else -> Unit
        }
    }
}
```

#### `.await()`
Firebase uses a callback-based `Task<T>`. `.await()` converts it into a suspending call so the code looks sequential:
```kotlin
val result = firebaseAuth.signInWithEmailAndPassword(email, password).await()
```

---

### Flow

#### What is a Flow?
A stream of values over time — emits a new value whenever something changes.

| | `suspend fun` | `Flow` |
|---|---|---|
| Emits | Once | Multiple times |
| Example | Insert a job | Observe all jobs, re-emit on every DB change |

```kotlin
suspend fun insertJob(job: JobApplicationEntity): Long       // one-shot
fun getAllJobs(): Flow<List<JobApplicationEntity>>            // live stream
```

When a job is inserted, `getAllJobs()` automatically emits the updated list. No polling — it's reactive.

#### `MutableStateFlow` vs `StateFlow`
`StateFlow` is a Flow that always has a current value and emits on change — perfect for UI state.
```kotlin
private val _state = MutableStateFlow(LoginState())   // writable (private)
val state: StateFlow<LoginState> = _state.asStateFlow() // read-only (public)
```
The `_` prefix is convention for the mutable backing field.

#### `.map{}` — transforming a Flow
```kotlin
val emailError = _state.map { loginState ->
    AuthValidators.validateEmail(loginState.email)
}
```
Every time `_state` emits, `.map` transforms it by running `validateEmail`. `emailError` stays in sync automatically.

#### `combine{}` — merging multiple Flows
```kotlin
val isLoginEnabled = combine(_state, emailError, passwordError) { state, eErr, pErr ->
    eErr == null && pErr == null && state.email.isNotBlank() && state.password.isNotBlank()
}
```
Any time **any** of the 3 flows change, the block re-runs and emits a new `Boolean`.

#### `stateIn` — converting Flow → StateFlow
`.map` and `combine` return a cold Flow (does nothing until collected). `stateIn` converts it to a hot `StateFlow`:
```kotlin
.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000),
    initialValue = null
)
```
`WhileSubscribed(5000)` — 5 second grace period after last subscriber disappears. Survives screen rotation without restarting the flow.

#### `collectAsStateWithLifecycle` — in the UI
```kotlin
val emailError by viewModel.emailError.collectAsStateWithLifecycle()
```
Collects the `StateFlow` in a Composable, converts to Compose `State`. When the flow emits, only the affected UI parts re-render. Stops collecting automatically when the screen is backgrounded.

---

### Full reactive chain — login button enabling

```
User types in email field
  → onEmailChange() called
  → _state.update { it.copy(email = newEmail) }
  → _state emits new LoginState
  → emailError.map{} runs validateEmail → emits null or error string
  → isLoginEnabled.combine{} re-evaluates → emits true or false
  → LoginScreen recomposes → button enabled/disabled
```
All automatic — zero manual wiring.

---

### Likely defense questions

| Question | Key answer |
|---|---|
| What is a coroutine? | Code that can pause and resume without blocking a thread |
| What does `suspend` mean? | Function can be paused; only callable from a coroutine or another suspend fun |
| What is `viewModelScope`? | Coroutine scope tied to ViewModel — auto-cancelled when ViewModel is destroyed |
| What is `launch`? | Starts a coroutine and returns immediately (fire and forget) |
| What does `.await()` do? | Converts Firebase Task callback into a suspending call |
| What is a Flow? | A stream that emits multiple values over time |
| `suspend fun` vs `Flow` in DAO? | `suspend` = one-shot operation; `Flow` = live stream that re-emits on DB change |
| What is `StateFlow`? | A Flow with a current value — perfect for UI state |
| Why `MutableStateFlow` private? | Encapsulation — only ViewModel mutates state, UI only reads |
| What does `.map{}` do on a Flow? | Transforms each emission into a new value |
| What does `combine` do? | Merges multiple flows — re-runs block when any input changes |
| What does `stateIn` do? | Converts a cold Flow into a hot StateFlow with a current value |
| What is `WhileSubscribed(5000)`? | 5s grace period after last subscriber — survives screen rotation without restarting |
| What is `collectAsStateWithLifecycle`? | Collects a Flow in Compose, stops collecting when screen is backgrounded |

---
