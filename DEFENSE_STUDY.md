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

## Q6: Navigation — how NavGraph.kt routes between screens

### What is Jetpack Compose Navigation?
Instead of switching between Activities, Compose Navigation uses a single `NavHost` with named **routes**. A `NavController` manages the back stack and moves between them.

### `Screen.kt` — route definitions
```kotlin
sealed class Screen(val route: String) {
    data object Login    : Screen("login_screen")
    data object Register : Screen("register_screen")
    data object Home     : Screen("home_screen")
    data object JobDetails : Screen("job_details_screen/{jobId}") {
        fun createRoute(jobId: Long): String = "job_details_screen/$jobId"
    }
}
```
A `sealed class` with `data object` entries — each screen is a singleton with a string route. `{jobId}` is a path argument like a URL parameter. `createRoute(123)` produces `"job_details_screen/123"`.

### `NavGraph.kt` — the map of your app
```kotlin
NavHost(navController, startDestination = Screen.Login.route) {

    composable(Screen.Login.route) {
        LoginScreen(
            onLoginClick = {
                navController.navigate(Screen.Home.route) {
                    popUpTo(Screen.Login.route) { inclusive = true }
                }
            }
        )
    }

    composable(
        route = Screen.JobDetails.route,
        arguments = listOf(navArgument("jobId") { type = NavType.LongType })
    ) {
        JobDetailScreen(onBack = { navController.popBackStack() })
    }
}
```

- **`popUpTo(...) { inclusive = true }`** — removes the login screen from the back stack after login. Pressing Back on Home exits the app instead of returning to login.
- **`popBackStack()`** — goes back one screen.
- **`navArgument`** — extracts `jobId` from the route string as a typed `Long`.

### Why screens don't take a NavController directly
`LoginScreen` receives `onLoginClick: () -> Unit` instead of a `NavController`. The screen just says "I'm done" — the NavGraph decides where to go. Keeps screens decoupled, reusable, and testable.

---

## Q7: Firebase & Firestore — what you're using and how it's set up

### Services used

| Service | Purpose |
|---|---|
| **Firebase Auth** | User login / registration / sign out |
| **Firestore** | Cloud database — stores job applications per user |

### `FirebaseModule.kt` — wiring Firebase into Hilt
```kotlin
@Provides @Singleton
fun provideFirebaseAuth(): FirebaseAuth = FirebaseAuth.getInstance()

@Provides @Singleton
fun provideFirebaseFirestore(): FirebaseFirestore = FirebaseFirestore.getInstance()
```
`getInstance()` returns Firebase's own singletons. Wrapping in `@Provides @Singleton` means Hilt manages them — any class that needs `FirebaseAuth` just declares it in its constructor.

### Firestore data structure
```
Firestore
└── users/
    └── {userId}/
        └── jobs/
            └── {jobId}  →  { title, company, status, dateApplied, ... }
```
Each user has their own `jobs` subcollection. `userJobsCollection(userId)` builds this path.

### `callbackFlow` — bridging Firestore listeners into a Flow
```kotlin
override fun getJobsForUser(userId: String): Flow<NetworkResult<List<Job>>> = callbackFlow {
    trySend(NetworkResult.Loading)
    val listener = userJobsCollection(userId).addSnapshotListener { snapshot, error ->
        if (error != null) {
            trySend(NetworkResult.Error(error.message ?: "Firestore error"))
            return@addSnapshotListener
        }
        val jobs = snapshot?.documents?.mapNotNull { ... } ?: emptyList()
        trySend(NetworkResult.Success(jobs))
    }
    awaitClose { listener.remove() }
}
```
Firestore's `addSnapshotListener` fires every time data changes in the cloud. `callbackFlow` converts it into a Kotlin Flow:
- `trySend()` emits a value into the Flow
- Every cloud change fires the listener → emits a new value → UI updates automatically
- `awaitClose { listener.remove() }` — removes the listener when nobody is collecting, preventing memory leaks

---

## Q8: Jetpack Compose Basics — @Composable, recomposition, state

### What is Jetpack Compose?
Android's modern UI toolkit. Instead of XML layouts and `findViewById`, you write UI as Kotlin functions. The framework draws UI based on state.

### `@Composable`
```kotlin
@Composable
fun AuthHeader(title: String, subtitle: String) {
    Column {
        Text(text = title)
        Text(text = subtitle)
    }
}
```
- Can only be called from another `@Composable`
- Has no return value — it emits UI
- Should have no side effects (no network calls, no file writes)

### Recomposition
When state changes, Compose **re-runs only the composables that read that state** — not the whole screen.
```kotlin
val emailError by viewModel.emailError.collectAsStateWithLifecycle()

Text(text = emailError ?: "")  // only this recomposes when emailError changes
```
If `emailError` didn't change, `Text` won't re-render. No manual view updates needed.

### State in Compose
Only reads of `State<T>` objects trigger recomposition. `collectAsStateWithLifecycle()` converts a `StateFlow` into Compose `State`:
```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```
The `by` delegate means you use `uiState` directly (not `uiState.value`). Every emission schedules recomposition of anything that reads it.

### `LaunchedEffect` — side effects
```kotlin
LaunchedEffect(uiState) {
    if (uiState is LoginUiState.Success) onLoginClick()
}
```
Composables shouldn't trigger navigation directly in their body — that would run on every recompose. `LaunchedEffect(key)` runs its block in a coroutine **once when the key changes**. When `uiState` becomes `Success`, navigate — but only once.

### `Modifier`
```kotlin
Column(
    modifier = Modifier
        .fillMaxSize()
        .padding(16.dp)
)
```
Chains UI instructions — size, padding, background, click handlers. Order matters — `padding` then `fillMaxSize` ≠ `fillMaxSize` then `padding`.

### `@Preview`
```kotlin
@Preview(showBackground = true, showSystemUi = true)
@Composable
fun LoginScreenPreview() {
    LoginScreen(email = "test@example.com", ...)
}
```
Renders the composable in Android Studio without running the app. This is why the private stateless `LoginScreen` overload exists — preview it with hardcoded data, no ViewModel or Firebase needed.

---

### Likely defense questions — all three topics

| Question | Key answer |
|---|---|
| What is a route in navigation? | A string identifier for a screen, like a URL |
| What does `popUpTo` do? | Removes screens from the back stack up to the specified route |
| Why pass callbacks instead of NavController to screens? | Decouples screens from navigation — reusable and testable |
| How do you pass data between screens? | Path arguments in the route string, declared with `navArgument` |
| What is Firebase Auth? | Manages user accounts — sign in, register, sign out |
| What is Firestore? | Cloud NoSQL document database — stores data as collections of documents |
| What is `callbackFlow`? | Converts a callback-based API into a Kotlin Flow |
| What does `awaitClose` do? | Cleans up the listener when the Flow has no more collectors — prevents leaks |
| What is `@Composable`? | Annotation marking a function that emits UI |
| What is recomposition? | Compose re-runs composables that read changed state |
| What triggers recomposition? | Reading a `State<T>` object whose value changed |
| What is `LaunchedEffect`? | Runs a coroutine as a side effect when its key changes — not on every recompose |
| What does `collectAsStateWithLifecycle` do? | Converts a StateFlow into Compose State; stops collecting when screen is backgrounded |
| What is a `Modifier`? | Chains UI instructions (size, padding, clicks); order matters |

---

## Q9: MainActivity — the app entry point

### Single Activity Pattern
The entire app has **one Activity** — `MainActivity`. All screens are Compose composables swapped in and out by the NavController. No other Activities exist.

### `@AndroidEntryPoint`
```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity()
```
Required by Hilt on any Activity using injection. Without it, `hiltViewModel()` calls crash.

### Auth check on launch
```kotlin
val startDestination = remember {
    if (authViewModel.authState.value == AuthCheckState.LoggedIn)
        Screen.Home.route
    else
        Screen.Login.route
}
```
`AuthViewModel` checks `FirebaseAuth.currentUser` — **synchronous**, no network call, Firebase caches it locally. `remember` ensures `startDestination` is computed once and never recalculated on recomposition.

### Reacting to logout
```kotlin
val initialAuthState = remember { authViewModel.authState.value }
LaunchedEffect(authState) {
    if (authState == initialAuthState) return@LaunchedEffect
    if (authState == AuthCheckState.LoggedOut) {
        navController.navigate(Screen.Login.route) {
            popUpTo(0) { inclusive = true }
        }
    }
}
```
`popUpTo(0) { inclusive = true }` clears the **entire** back stack. After logout the user can't press Back to re-enter the app. The `initialAuthState` guard prevents navigating on the very first composition.

### Bottom Navigation Bar
```kotlin
val showBottomBar = currentRoute in bottomNavScreens.map { it.first.route }
```
Only shown on the 4 main screens — not on Login, Register, or JobDetails.

```kotlin
navController.navigate(screen.route) {
    popUpTo(Screen.Home.route) { saveState = true }
    launchSingleTop = true
    restoreState = true
}
```
- `launchSingleTop` — don't create a second copy of a screen already on top
- `saveState` / `restoreState` — remember scroll position when switching tabs

---

## Q10: Retrofit / NetworkModule — the REST API layer

### What is Retrofit?
Type-safe HTTP client for Android. You define an interface describing your API; Retrofit generates the implementation.

### `JobApiService` — the API contract
```kotlin
interface JobApiService {
    @GET("habits/")
    suspend fun getHabits(): Response<List<HabitDto>>

    @POST("habits/")
    suspend fun createHabit(@Body habit: CreateHabitDto): Response<HabitDto>

    @PUT("habits/{id}")
    suspend fun updateHabit(@Path("id") id: Int, @Body habit: CreateHabitDto): Response<HabitDto>

    @DELETE("habits/{id}")
    suspend fun deleteHabit(@Path("id") id: Int): Response<Unit>
}
```
- `@GET`, `@POST`, `@PUT`, `@DELETE` — HTTP methods
- `@Path("id")` — replaces `{id}` in the URL with the actual value
- `@Body` — serializes the Kotlin object to JSON as the request body
- `suspend` — Retrofit supports coroutines natively
- `Response<T>` — wraps result so you can check `isSuccessful`, `code()`, `body()`

### `NetworkModule` — building Retrofit with Hilt

```kotlin
private const val BASE_URL = "http://10.0.2.2:8000/"
```
`10.0.2.2` is the Android emulator's alias for `localhost` on the host machine. Your backend runs on `localhost:8000` on your PC; the emulator reaches it via `10.0.2.2:8000`.

```kotlin
OkHttpClient.Builder()
    .addInterceptor(logging)
    .addInterceptor(Interceptor { chain ->
        val request = chain.request().newBuilder()
            .addHeader("X-Authentication", "yes")
            .build()
        chain.proceed(request)
    })
    .build()
```
**Interceptors** run on every HTTP request. The logging interceptor prints full request/response bodies to Logcat. The custom interceptor adds `X-Authentication: yes` to every request automatically.

`GsonConverterFactory` automatically converts JSON ↔ Kotlin data classes. `retrofit.create(JobApiService::class.java)` generates the implementation at runtime — you never write HTTP code manually.

### `safeApiCall` — the error wrapper
```kotlin
private inline fun <T> safeApiCall(block: () -> NetworkResult<T>): NetworkResult<T> =
    try { block() } catch (e: Exception) { NetworkResult.Error(e.message ?: "Network error") }
```
Wraps every network call in try/catch. If the network is down, returns `NetworkResult.Error` instead of crashing. `inline` means no overhead from the lambda object.

### DTOs vs Domain models
`HabitDto` is the raw API shape. `Job` is your app's domain model. The mapper (`toDomain()`) converts between them — the ViewModel never sees raw API shapes.

---

## Q10b: Stateful vs Stateless Composables

### The simple answer

The difference is: **does this function own or connect to state?**

---

### Stateless — "dumb" display function

A stateless composable receives only plain data and lambdas. It has no idea where the data comes from or what happens when buttons are clicked. It just draws what it's given.

```kotlin
// LoginScreen.kt:72 — private, stateless
@Composable
private fun LoginScreen(
    email: String,            // plain String, not a StateFlow
    password: String,
    emailError: String?,
    isLoginEnabled: Boolean,
    onLoginClick: () -> Unit  // just a lambda, doesn't know what it does
) {
    // only UI here — no viewModel, no StateFlow, no business logic
    AuthTextField(value = email, ...)
    AuthPrimaryButton(enabled = isLoginEnabled, onClick = onLoginClick)
}
```

You can **preview this in Android Studio** because you can call it with hardcoded values — no ViewModel or Firebase needed.

---

### Stateful — "smart" connector function

A stateful composable owns or connects to state. It knows about the ViewModel, collects the flows, and passes real data down to the stateless version.

```kotlin
// LoginScreen.kt:39 — public, stateful
@Composable
fun LoginScreen(
    onLoginClick: () -> Unit,
    viewModel: LoginViewModel = hiltViewModel()  // owns the ViewModel
) {
    // collects real live data from StateFlows
    val state by viewModel.state.collectAsStateWithLifecycle()
    val emailError by viewModel.emailError.collectAsStateWithLifecycle()
    val isLoginEnabled by viewModel.isLoginEnabled.collectAsStateWithLifecycle()

    // passes everything down to the stateless version
    LoginScreen(
        email = state.email,
        emailError = emailError,
        isLoginEnabled = isLoginEnabled,
        onLoginClick = { viewModel.login() }
    )
}
```

---

### Why two functions instead of one?

| | Stateful | Stateless |
|---|---|---|
| Knows about ViewModel | Yes | No |
| Can be previewed in IDE | No | Yes |
| Easy to test in isolation | No | Yes |
| Used by NavGraph | Yes | No |
| Contains business logic | No (delegates to VM) | No |

The stateful one is the **entry point** (wired in NavGraph). The stateless one is the **renderer** (tested and previewed with any data). This pattern is called **state hoisting** — lifting state up out of the UI so the rendering layer stays pure.

Your `RegisterScreen.kt` does the same split — the public `RegistrationScreen` (has ViewModel + `rememberSaveable`) calls the private `RegistrationScreen` (stateless, just renders).

---

### Likely defense questions

| Question | Key answer |
|---|---|
| What is a stateless composable? | Takes only plain data and lambdas — no ViewModel, no flows, just renders |
| What is a stateful composable? | Owns state or connects to a ViewModel — collects flows, delegates rendering |
| Why split them? | Stateless can be previewed and tested without a ViewModel or running app |
| What is state hoisting? | Moving state up to the stateful composable so the renderer stays pure |
| Which one does NavGraph use? | The stateful (public) one — it's the screen entry point |

---

## Q10c: Are There Any State Variables Inside a Stateless Composable?

### In your code — no

Your private stateless composables contain zero state:

```kotlin
// LoginScreen.kt:72 — completely pure
@Composable
private fun LoginScreen(
    email: String,
    password: String,
    emailError: String?,
    isLoginEnabled: Boolean,
    ...
) {
    // no remember, no mutableStateOf, no ViewModel — nothing
}
```

Same for the private `RegistrationScreen` at line 104 — only parameters and UI.

---

### The one exception: `AddJobForm.kt`

```kotlin
// AddJobForm.kt:25-26
var jobTitle by remember { mutableStateOf("") }
var company  by remember { mutableStateOf("") }
```

`AddJobForm` owns its own local state inside a single composable — it is not split into stateful/stateless. This makes it technically a **stateful composable**, even though it doesn't use a ViewModel.

---

### The general rule

A composable *can* have local state (`remember`) and still be considered "stateless" in architecture terms — **if that state is purely cosmetic** (a dropdown open/closed, scroll position, animation). That kind of state has no business value and doesn't need to live in the ViewModel.

But if the state affects business logic — like form inputs that get submitted — it should be hoisted. Your Login and Register screens do this correctly: even `passwordVisible` is hoisted all the way to the stateful composable, not kept locally.

> **Rule of thumb:** Does the state need to survive the composable, be tested, or be shared? → hoist it. Is it purely visual and temporary? → keep it with `remember`.

---

### Likely defense questions

| Question | Key answer |
|---|---|
| Can a stateless composable have state? | Yes — local cosmetic state (`remember`) is fine; business logic state should be hoisted |
| Does your private LoginScreen have any state? | No — pure parameters and lambdas only |
| What about AddJobForm? | It owns local state with `remember` — it is stateful, not split |
| Why is `passwordVisible` hoisted instead of kept locally? | It's part of the form's interaction model; hoisting keeps the stateless composable fully previewable and testable |

---

## Q10d: What is a DTO and How Does It Work?

### What is a DTO?

A DTO (Data Transfer Object) is a data class whose fields **exactly match the shape of the JSON** the API sends back. It exists purely to receive raw API data — it is not the object your app actually uses.

---

### The problem it solves

The API returns this JSON:
```json
{
  "id": 4,
  "title": "Android Developer",
  "description": "Google",
  "frequency": "Once",
  "completed": false,
  "user_id": 1
}
```

Your app's `Job` domain model looks like:
```kotlin
data class Job(
    val id: Long,
    val title: String,
    val company: String,    // ← different name than "description"
    val status: String,     // ← doesn't exist in API at all
    val dateApplied: String,
    val userId: Int         // ← different name than "user_id"
)
```

They don't match. You can't parse the JSON directly into `Job`.

---

### Step 1 — The DTO matches the JSON exactly (`HabitDto.kt`)

```kotlin
data class HabitDto(
    @SerializedName("id")          val id: Int?,
    @SerializedName("title")       val title: String?,
    @SerializedName("description") val description: String?,
    @SerializedName("frequency")   val frequency: String?,
    @SerializedName("completed")   val completed: Boolean?,
    @SerializedName("user_id")     val userId: Int?
)
```

- `@SerializedName` maps the JSON key (`"user_id"`) to your Kotlin field (`userId`)
- Everything is **nullable** (`?`) because API responses can be unpredictable — a field might be missing
- `frequency` is included even though your app doesn't use it — the DTO is just a raw receiver

---

### Step 2 — The Mapper converts DTO → Domain (`RemoteJobMapper.kt`)

```kotlin
fun HabitDto.toDomain(): Job = Job(
    id          = (id ?: 0).toLong(),
    title       = title ?: "Untitled",
    company     = description ?: "",           // "description" becomes "company"
    status      = if (completed == true) "Offer Received" else "Applied",  // derived field
    dateApplied = "N/A",
    userId      = userId ?: 0
)
```

`?:` handles nullability — if a field is missing from the JSON it falls back to a safe default. `frequency` is simply dropped.

---

### Step 3 — The full flow

```
API returns JSON
    ↓
Gson parses it into HabitDto   (matches JSON shape exactly)
    ↓
.toDomain() mapper runs        (converts to your app's Job model)
    ↓
ViewModel receives List<Job>   (never sees HabitDto at all)
    ↓
UI displays it
```

---

### Why not just use `Job` directly?

| Reason | Explanation |
|---|---|
| Different field names | API uses `user_id`, your app uses `userId` |
| Extra fields | API has `frequency` your app doesn't need |
| Nullability | API fields can be missing; domain model should be non-nullable |
| API can change | If the API renames a field, you only update the DTO — ViewModel and UI are untouched |

---

### Likely defense questions

| Question | Key answer |
|---|---|
| What is a DTO? | A data class that matches the API's JSON shape exactly — used only for receiving raw data |
| Why not parse JSON directly into the domain model? | Field names differ, domain model is non-nullable, API has fields the app doesn't need |
| What does `@SerializedName` do? | Maps a JSON key name to a Kotlin field name |
| Why are DTO fields nullable? | API responses can be missing fields — nullable prevents a crash |
| What is a mapper? | A function that converts a DTO into a domain model the rest of the app uses |
| Where does the ViewModel get its data from? | The domain model (`Job`) — it never sees the DTO |

---

## Q10e: What Types of Navigation Are There?

Your app uses 4 distinct types of navigation, plus `popBackStack`.

---

### 1. Simple Navigation — go to a screen

```kotlin
// NavGraph.kt:37
navController.navigate(Screen.Register.route)
```
The most basic form. Navigates to a route string and adds the destination to the back stack — pressing Back returns to the previous screen.

---

### 2. Navigation with Back Stack Clearing — `popUpTo`

```kotlin
// NavGraph.kt:33-35 — after login
navController.navigate(Screen.Home.route) {
    popUpTo(Screen.Login.route) { inclusive = true }
}
```
After login, you don't want the user pressing Back and returning to Login. `popUpTo(Login) { inclusive = true }` removes Login from the back stack before navigating to Home. `inclusive = true` means Login itself is also removed (not just everything above it).

Used again on logout (`MainActivity.kt:60-62`) with `popUpTo(0)` — `0` clears the **entire** back stack so the user can't go Back into the app at all after signing out.

---

### 3. Navigation with Arguments — passing data between screens

```kotlin
// NavGraph.kt:59 — passing a jobId
navController.navigate(Screen.JobDetails.createRoute(jobId))
// → navigates to "job_details_screen/42"

// Screen.kt — the route template with {jobId} placeholder
data object JobDetails : Screen("job_details_screen/{jobId}") {
    fun createRoute(jobId: Long) = "job_details_screen/$jobId"
}

// NavGraph.kt:72-76 — declaring the argument type
composable(
    route = Screen.JobDetails.route,
    arguments = listOf(navArgument("jobId") { type = NavType.LongType })
)
```
`{jobId}` in the route acts like a URL parameter. `createRoute(42)` produces `"job_details_screen/42"`. The ViewModel reads it via `SavedStateHandle["jobId"]`.

---

### 4. Bottom Navigation — tab switching with state preservation

```kotlin
// MainActivity.kt:82-86
navController.navigate(screen.route) {
    popUpTo(Screen.Home.route) { saveState = true }
    launchSingleTop = true
    restoreState = true
}
```

Three flags working together:
- `launchSingleTop = true` — if you tap the tab you're already on, don't create a duplicate screen
- `saveState = true` — save scroll position and state when leaving a tab
- `restoreState = true` — restore that saved state when returning to the tab

The bottom bar is only shown on the 4 main screens, hidden on Login, Register, and JobDetails (`MainActivity.kt:72`).

---

### Also: `popBackStack` — going back one screen

```kotlin
// NavGraph.kt:48 — Register → Login
onLoginClick = { navController.popBackStack() }

// NavGraph.kt:79 — JobDetails → back
onBack = { navController.popBackStack() }
```
Removes the current screen from the back stack and returns to the previous one — the equivalent of pressing the system Back button.

---

### Summary

| Type | Where in your code | What it does |
|---|---|---|
| Simple navigate | `NavGraph.kt:37` | Go to a screen, add to back stack |
| Navigate + popUpTo | `NavGraph.kt:33-35` | Go to screen, remove previous screens from back stack |
| Navigate with argument | `NavGraph.kt:59` | Pass data (jobId) through the route URL |
| Bottom nav (tabs) | `MainActivity.kt:82-86` | Switch tabs, save/restore state, no duplicates |
| popBackStack | `NavGraph.kt:48, 79` | Go back one screen |

---

### Likely defense questions

| Question | Key answer |
|---|---|
| What types of navigation do you use? | Simple, popUpTo back-stack clearing, argument passing, bottom nav tabs, popBackStack |
| Why `popUpTo(Login) { inclusive = true }` after login? | Removes Login from back stack so pressing Back doesn't return to it |
| How do you pass data between screens? | Through the route URL as a path argument — `"job_details_screen/{jobId}"` |
| What does `launchSingleTop` do? | Prevents creating a duplicate of a screen already on top of the stack |
| What does `saveState`/`restoreState` do in bottom nav? | Saves and restores scroll position when switching tabs |
| What does `popUpTo(0)` on logout do? | Clears the entire back stack — user cannot press Back into the app |
| Why don't screens receive the NavController directly? | Decoupling — screen just calls `onBack()`, NavGraph decides what "back" means |

---

## Q10f: What Are Threads and Coroutines?

### Threads

A **thread** is an independent path of execution. Your phone can run multiple threads simultaneously — like having multiple workers doing different tasks in parallel.

Android has two important thread categories:

#### Main Thread (UI Thread)
The one thread allowed to draw the UI. Every button render, every text update, every animation happens here — 60 times per second. **If you block it**, the app freezes. If it stays frozen for 5 seconds, Android shows "App Not Responding" and kills the app.

```kotlin
// THIS WOULD FREEZE THE APP — database query on main thread
val jobs = database.jobDao().getAllJobs()  // blocks for 200ms → UI stutters
```

#### Background Threads
Used for slow work: network calls, database reads/writes, file I/O. The main thread stays free while background threads do the heavy lifting.

---

### Coroutines

A coroutine is **not** a thread. It's a block of code that can **pause itself** without blocking the thread it's on, then resume later when the result is ready.

Think of it like a waiter in a restaurant:
- **Thread blocking** = waiter stands at the kitchen window staring until the food is ready. Can't serve anyone else.
- **Coroutine** = waiter puts in the order, goes to serve other tables, comes back when the food is ready.

The thread is free to do other work while the coroutine waits.

---

### `suspend` — the key keyword

```kotlin
// AuthRepositoryImpl.kt
suspend fun signIn(email: String, password: String): NetworkResult<FirebaseUser> {
    val result = firebaseAuth
        .signInWithEmailAndPassword(email, password)
        .await()   // ← pauses here, frees the thread, resumes when Firebase responds
    return NetworkResult.Success(result.user!!)
}
```

`suspend` marks a function that **can be paused**. It can only be called from another `suspend` function or from inside a coroutine. At `.await()` the coroutine pauses — but the thread is not blocked, it can run other coroutines in the meantime.

---

### `viewModelScope.launch` — starting a coroutine

```kotlin
// LoginViewModel.kt:55-63
fun login() {
    viewModelScope.launch {               // starts a coroutine
        _uiState.update { LoginUiState.Loading }
        when (val result = authRepository.signIn(...)) {
            is NetworkResult.Success -> _uiState.update { LoginUiState.Success }
            is NetworkResult.Error   -> _uiState.update { LoginUiState.Error(result.message) }
            else -> Unit
        }
    }
}
```

`viewModelScope` is a coroutine scope built into every ViewModel. Coroutines launched here are **automatically cancelled** when the ViewModel is destroyed — no memory leaks, no background work continuing after the screen is gone.

`launch` starts the coroutine and **returns immediately** — `login()` itself is not `suspend`, it fires the coroutine and returns right away while the coroutine runs in the background.

---

### Threads vs Coroutines — the key difference

| | Threads | Coroutines |
|---|---|---|
| What blocks while waiting | The whole thread | Nothing — thread is free |
| Cost | Heavy — ~1MB each, OS managed | Lightweight — thousands can run at once |
| Cancellation | Hard to cancel safely | Built-in via scope |
| Code style | Callbacks / deeply nested | Looks like normal sequential code |

---

### `Dispatchers` — which thread does the coroutine run on?

```kotlin
viewModelScope.launch(Dispatchers.IO) {
    // runs on a background thread pool — for database/network
}
viewModelScope.launch(Dispatchers.Main) {
    // runs on the main thread — for UI updates
}
```

Room and Retrofit handle dispatcher switching automatically — you just call them and they internally move to `IO` for you.

---

### `.await()` — bridging Firebase callbacks to coroutines

Firebase was designed before coroutines existed and uses callbacks (`Task<T>`). `.await()` converts a Firebase Task into a suspending call:

```kotlin
// old callback style — deeply nested
firebaseAuth.signInWithEmailAndPassword(email, password)
    .addOnSuccessListener { result -> /* nested */ }
    .addOnFailureListener { error -> /* more nesting */ }

// coroutine style — reads top to bottom like normal code
val result = firebaseAuth.signInWithEmailAndPassword(email, password).await()
```

---

### Full picture in your app — login flow

```
User taps Login button
    ↓
login() called on Main Thread
    ↓
viewModelScope.launch { }    — coroutine starts
    ↓
_uiState = Loading           — updates UI spinner (Main Thread)
    ↓
authRepository.signIn().await() — coroutine PAUSES
    ↓                             Main Thread is FREE to keep drawing
Firebase responds
    ↓
coroutine RESUMES
    ↓
_uiState = Success           — updates UI (back on Main Thread)
    ↓
LaunchedEffect sees Success → navigates to Home
```

---

### Likely defense questions

| Question | Key answer |
|---|---|
| What is a thread? | An independent path of execution — Android has a Main Thread for UI and background threads for slow work |
| What happens if you block the Main Thread? | App freezes; after 5 seconds Android shows ANR (App Not Responding) |
| What is a coroutine? | A block of code that can pause without blocking its thread, then resume when ready |
| What does `suspend` mean? | Marks a function that can be paused; only callable from a coroutine or another suspend function |
| What is `viewModelScope`? | A coroutine scope tied to the ViewModel — all coroutines cancel automatically when the ViewModel is destroyed |
| What does `launch` do? | Starts a coroutine and returns immediately (fire and forget) |
| What is `.await()`? | Converts a Firebase Task (callback) into a suspending call — bridges the old API to coroutines |
| Coroutines vs threads? | Coroutines don't block the thread while waiting; they're lighter and thousands can run at once |
| What is a Dispatcher? | Controls which thread the coroutine runs on — `IO` for database/network, `Main` for UI |

---

## Q10g: State Management — All 6 Requirements Explained

### 1. Proper State Hoisting

State hoisting means **moving state up** to the right level so the composable that renders it doesn't own it.

**`LoginViewModel.kt:36`** — form state hoisted all the way to the ViewModel:
```kotlin
private val _state = MutableStateFlow(LoginState())  // ViewModel owns it
```
The private `LoginScreen` composable receives `email: String` and `onEmailChange: (String) -> Unit` — plain values and lambdas. It owns nothing.

**`RegisterScreen.kt:45-51`** — form fields hoisted to the stateful composable using `rememberSaveable`:
```kotlin
var fullName by rememberSaveable { mutableStateOf("") }
var email    by rememberSaveable { mutableStateOf("") }
```
`rememberSaveable` survives screen rotation (unlike plain `remember`). The private `RegistrationScreen` renders these values but doesn't own them.

> **Rule:** state is owned at the *lowest common ancestor* of everything that needs it. Email is needed by the text field AND the validator AND the button — so it lives in the ViewModel.

---

### 2. Single Source of Truth

Every piece of data has **exactly one owner**. Nothing is duplicated.

**`LoginViewModel.kt:36`** — email exists in one place, everything else is computed from it:
```kotlin
private val _state = MutableStateFlow(LoginState())
```
`emailError`, `passwordError`, and `isLoginEnabled` are all derived FROM `_state`. They don't store their own copy of the email.

**Room as source of truth for jobs (`JobListViewModel.kt:32-46`):**
```kotlin
val uiState = combine(jobRepository.getJobs(), _searchQuery) { jobs, query -> ... }
```
`jobRepository.getJobs()` is a `Flow` from Room. Room emits a new list whenever the database changes. The UI never holds its own copy of jobs. If you add a job in Dashboard → Room emits → `uiState` auto-updates → `JobListScreen` recomposes. No manual sync needed anywhere.

---

### 3. UI Reacting to State Changes

The UI never polls or refreshes manually — it reacts to state emissions automatically.

**`LoginScreen.kt:44-48`:**
```kotlin
val uiState        by viewModel.uiState.collectAsStateWithLifecycle()
val emailError     by viewModel.emailError.collectAsStateWithLifecycle()
val isLoginEnabled by viewModel.isLoginEnabled.collectAsStateWithLifecycle()
```
When a flow emits a new value, **only the composables that read that specific value recompose** — nothing else.

**`JobListingScreen.kt:85-117`** — `when` block reacts to all possible states:
```kotlin
when (uiState) {
    is JobListUiState.Loading -> CircularProgressIndicator()
    is JobListUiState.Success -> LazyColumn(...)
    is JobListUiState.Error   -> Text("Error: ...")
}
```
Screen automatically switches between spinner, list, and error based purely on what the ViewModel emits.

**`LoginScreen.kt:50-52`** — navigation reacts to state:
```kotlin
LaunchedEffect(uiState) {
    if (uiState is LoginUiState.Success) onLoginClick()
}
```
Navigation triggers once when `uiState` becomes `Success` — not on every recompose.

---

### 4. At Least Three Derived / Computed State

Derived state is **computed from other state** rather than stored independently. You have 7:

| Derived State | Computed From | File |
|---|---|---|
| `emailError` | `_state.email` via `.map {}` | `LoginViewModel.kt:39-41` |
| `passwordError` | `_state.password` via `.map {}` | `LoginViewModel.kt:43-45` |
| `isLoginEnabled` | `_state` + `emailError` + `passwordError` via `combine` | `LoginViewModel.kt:47-49` |
| `uiState` (JobList) | `getJobs()` + `_searchQuery` via `combine` | `JobListViewModel.kt:32-46` |
| `uiState` (Home) | `getJobStats()` + `getUser()` via `combine` | `HomeViewModel.kt:30-39` |
| `isTitleTooShort` | `jobTitle` length | `AddJobForm.kt:28` |
| `isFormValid` | `jobTitle` + `company` | `AddJobForm.kt:29` |

None of these store a separate copy of the data — they recompute automatically whenever their inputs change:
```kotlin
// LoginViewModel.kt:39-49
val emailError = _state.map {
    AuthValidators.validateEmail(it.email)  // recomputes every time email changes
}.stateIn(...)

val isLoginEnabled = combine(_state, emailError, passwordError) { s, eErr, pErr ->
    eErr == null && pErr == null && s.email.isNotBlank() && s.password.isNotBlank()
}.stateIn(...)  // recomputes when ANY of the 3 inputs change
```

---

### 5. No Business Logic Inside Composables

**Login button — `LoginScreen.kt:129-131`:**
```kotlin
AuthPrimaryButton(
    onClick = onLoginClick,   // lambda passed in — screen doesn't know what happens
    enabled = isLoginEnabled  // value passed in — screen doesn't compute this
)
```
The composable doesn't call Firebase, doesn't validate, doesn't know what `isLoginEnabled` means.

**Search filtering — `JobListViewModel.kt:36-40`:**
```kotlin
val filtered = if (query.isBlank()) jobs
else jobs.filter {
    it.title.contains(query, ignoreCase = true) ||
    it.company.contains(query, ignoreCase = true)
}
```
Filtering logic lives in the ViewModel. `JobListingScreen` just receives the already-filtered list and renders it.

> **One partial exception:** `RegisterScreen.kt:59-75` runs validators inline in the stateful composable. This is borderline acceptable — it's view-level validation (not calling Firebase or Room) — but ideally it would live in `RegisterViewModel` the same way `LoginViewModel` handles it.

---

### 6. Recomposition Handled Correctly

Recomposition = Compose re-running a composable when its state changes. The goal is to recompose **only what changed**.

**`collectAsStateWithLifecycle` stops when backgrounded (`LoginScreen.kt:44`):**
Automatically stops collecting when the screen is in the background. No wasted work while the app is hidden.

**`LaunchedEffect(key)` runs only when key changes (`LoginScreen.kt:50`):**
```kotlin
LaunchedEffect(uiState) {
    if (uiState is LoginUiState.Success) onLoginClick()
}
```
Without `LaunchedEffect`, `onLoginClick()` would fire on *every recomposition*. `LaunchedEffect` ensures it runs only once when `uiState` actually changes to `Success`.

**`remember { startDestination }` computed once (`MainActivity.kt:47`):**
```kotlin
val startDestination = remember {
    if (authViewModel.authState.value == AuthCheckState.LoggedIn)
        Screen.Home.route else Screen.Login.route
}
```
Without `remember`, this recomputes on every recomposition of `MainActivity`. `remember` caches the result so it's calculated exactly once.

**`key` in `LazyColumn` (`JobListingScreen.kt:102`):**
```kotlin
items(uiState.jobs, key = { it.id }) { job ->
    JobItemCard(...)
}
```
Without `key`, if one job changes, Compose might recompose *all* job cards. With `key = { it.id }`, Compose knows exactly which item changed and recomposes only that one card.

---

### Summary map

| Requirement | Where in your code |
|---|---|
| State hoisting | `LoginViewModel.kt:36`, `RegisterScreen.kt:45-51` |
| Single source of truth | `LoginViewModel.kt:36`, `JobListViewModel.kt:32` |
| UI reacts to state | `LoginScreen.kt:44-48`, `JobListingScreen.kt:85-117` |
| Derived state (7 examples) | `LoginViewModel.kt:39-49`, `JobListViewModel.kt:32-46`, `HomeViewModel.kt:30-39`, `AddJobForm.kt:28-29` |
| No business logic in composables | `LoginScreen.kt:129`, `JobListViewModel.kt:36-40` |
| Recomposition handled correctly | `LoginScreen.kt:50`, `MainActivity.kt:47`, `JobListingScreen.kt:102` |

---

### Likely defense questions

| Question | Key answer |
|---|---|
| What is state hoisting? | Moving state up to the lowest common ancestor that needs it — composables receive values, not own them |
| What is single source of truth? | One owner per piece of data — derived state is computed from it, not stored separately |
| How does the UI react to state changes? | `collectAsStateWithLifecycle()` converts a StateFlow to Compose State — any composable reading it recomposes automatically |
| What is derived state? Give examples | State computed from other state — `emailError` from `_state.email`, `isLoginEnabled` from 3 combined flows, filtered job list from jobs + search query |
| Why no business logic in composables? | Composables are destroyed and recreated on rotation — logic there is lost; ViewModel survives and owns logic |
| What is recomposition? | Compose re-running a composable when its state changes |
| What is `LaunchedEffect` for? | Running a side effect once when a key changes — prevents repeated navigation or API calls on every recompose |
| What does `key` in `LazyColumn` do? | Tells Compose which item is which so it only recomposes the item that actually changed |

---

## Q10h: What Are LazyColumns and Where Do You Have Them?

### What is a LazyColumn?

A `LazyColumn` is a **scrollable vertical list** that only renders items currently visible on screen. It is the Compose equivalent of `RecyclerView`.

#### Why "Lazy"?

A regular `Column` composes **all items upfront** — even ones 900 scrolls away. A `LazyColumn` only composes the ~8 items currently visible, creating and destroying composables as you scroll.

```
Regular Column:  renders ALL items upfront  → slow, uses lots of memory
LazyColumn:      renders only VISIBLE items → fast, memory efficient
```

#### Basic structure

```kotlin
LazyColumn {
    item { /* single item — header, spacer, form */ }
    items(list, key = { it.id }) { element ->
        /* one composable per element, only rendered when visible */
    }
}
```

- `item { }` — one individual composable
- `items(list) { }` — one composable per element in a list
- `key = { it.id }` — tells Compose which item is which so it only recomposes what changed

---

### Your 3 LazyColumns

#### 1. `DashboardScreen.kt:66` — LazyColumn as a page layout

```kotlin
LazyColumn(modifier = modifier.fillMaxSize().padding(16.dp)) {
    item { ScreenHeader(...) }
    item { Spacer(...) }
    item { DashboardMetricsCard(...) }
    item { Spacer(...) }
    item { AddJobForm(onSubmit = onAddJob) }
}
```

**No repeating list here.** This LazyColumn wraps the entire Dashboard page so that if the keyboard opens or the screen is short, the whole page can scroll. Each section (header, stats card, form) is a single `item {}`. This is a common pattern when a screen has a form at the bottom that might get pushed off screen.

---

#### 2. `JobListingScreen.kt:101` and `141` — LazyColumn for actual lists

```kotlin
// Local jobs — line 101
LazyColumn(modifier = Modifier.weight(1f, fill = false)) {
    items(uiState.jobs, key = { it.id }) { job ->
        JobItemCard(jobTitle = job.title, company = job.company, ...)
    }
}

// Remote (API) jobs — line 141
LazyColumn(modifier = Modifier.weight(1f, fill = false)) {
    items(remoteUiState.jobs, key = { "remote_${it.id}" }) { job ->
        JobItemCard(...)
    }
}
```

Classic use case — one list from Room, one list from the API. Each `JobItemCard` is only composed when it scrolls into view.

`key = { "remote_${it.id}" }` has a `"remote_"` prefix to prevent key collision — both lists could have a job with `id = 1`, so the keys must be distinct.

---

#### 3. `JobDetailsScreen.kt:147` — LazyColumn mixing `item` and `items`

```kotlin
LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
    item { JobInfoRow(label = "Company", ...) }
    item { JobInfoRow(label = "Date Applied", ...) }
    item { OutlinedTextField(...) }        // edit title
    item { ExposedDropdownMenuBox(...) }   // status dropdown
    item { Button("Save Changes") }
    item { OutlinedTextField(...) }        // add note input
    item { Button("Add Note") }

    items(notes, key = { it.id }) { note ->
        // one deletable row per interview note
    }

    item { Spacer(...) }
}
```

The whole detail screen is one `LazyColumn`. Fixed elements (company, edit fields, buttons) are `item {}`. Interview notes at the bottom are `items(notes)` — there could be zero or many, and they scroll as part of the same column. Deleting one note only recomposes that row thanks to `key = { it.id }`.

`verticalArrangement = Arrangement.spacedBy(8.dp)` automatically adds 8dp gap between every item — no manual `Spacer` needed between each element.

---

### Summary

| File | Line | LazyColumn use | Content type |
|---|---|---|---|
| `DashboardScreen.kt` | 66 | Page layout — makes whole screen scrollable | All `item {}` — no repeating list |
| `JobListingScreen.kt` | 101 | Local jobs from Room | `items(uiState.jobs)` with delete |
| `JobListingScreen.kt` | 141 | Remote jobs from API | `items(remoteUiState.jobs)` read-only |
| `JobDetailsScreen.kt` | 147 | Mixed fixed content + dynamic notes | `item {}` + `items(notes)` combined |

---

### Likely defense questions

| Question | Key answer |
|---|---|
| What is a LazyColumn? | Scrollable vertical list that only renders visible items — like RecyclerView |
| Why not just use a Column? | Column renders all items upfront — slow and memory-heavy for large lists |
| What is `item {}` vs `items(list)`? | `item` = one single composable; `items` = one composable per element in a list |
| What does `key` do in `items`? | Tells Compose which item is which — only the changed item recomposes, not the whole list |
| Why `"remote_"` prefix on the key in JobListingScreen? | Prevents key collision — both lists could have a job with id = 1 |
| Where do you use LazyColumn as a page layout? | DashboardScreen — wraps the whole screen so it scrolls when the keyboard appears |
| Where do you mix `item` and `items`? | JobDetailsScreen — fixed fields as `item {}`, interview notes as `items(notes)` |

---

## Q10i: ViewModel, Hilt, Room, Repository & Coroutines — Full Milestone Requirements

---

### ViewModel Implementation

#### You have 9 ViewModels (minimum was 5)

| ViewModel | Screen | UiState |
|---|---|---|
| `LoginViewModel` | LoginScreen | `LoginUiState` — Init, Loading, Success, Error |
| `RegisterViewModel` | RegisterScreen | `RegisterUiState` — Idle, Loading, Success, Error |
| `HomeViewModel` | HomeScreen | `HomeUiState` — Init, Loading, Success, Error |
| `JobListViewModel` | JobListingScreen | `JobListUiState` — Init, Loading, Success, Error |
| `JobDetailViewModel` | JobDetailsScreen | `JobDetailUiState` — Init, Loading, Success, Error, Deleted |
| `DashboardViewModel` | DashboardScreen | `DashboardUiState` + `AddJobState` — two UiStates |
| `ProfileViewModel` | ProfileScreen | `ProfileUiState` — Init, Loading, Success, Error |
| `AuthViewModel` | MainActivity | `AuthCheckState` — Loading, LoggedIn, LoggedOut |
| `RemoteJobViewModel` | JobListingScreen (API section) | `RemoteJobUiState` — Idle, Loading, Success, Error |

Every ViewModel uses `@HiltViewModel` + `@Inject constructor` — Hilt creates and injects them automatically.

**UiState sealed class — `LoginViewModel.kt:21-26`:**
```kotlin
sealed class LoginUiState {
    object Init    : LoginUiState()
    object Loading : LoginUiState()
    object Success : LoginUiState()
    data class Error(val message: String) : LoginUiState()
}
```
`sealed` means the compiler knows every possible state — `when` blocks on it are exhaustive.

**StateFlow pattern — `LoginViewModel.kt:33-34`:**
```kotlin
private val _uiState = MutableStateFlow<LoginUiState>(LoginUiState.Init)
val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()
```
Private mutable, public read-only. The screen can observe but never write directly.

**Stateful + Stateless — `LoginScreen.kt:39 / 72`:**
```kotlin
// Public stateful — connects to ViewModel
fun LoginScreen(viewModel: LoginViewModel = hiltViewModel()) { ... }

// Private stateless — receives plain data, renders only
private fun LoginScreen(email: String, isLoginEnabled: Boolean, ...) { ... }
```

---

### Dependency Injection — Hilt

Hilt is wired into every layer:

```
@HiltAndroidApp (MobileProgrammingApp.kt)   ← starts Hilt for the whole app
    ↓
@AndroidEntryPoint MainActivity             ← allows hiltViewModel() to work
    ↓
@HiltViewModel + @Inject (all 9 ViewModels) ← Hilt creates and injects them
    ↓
@Inject constructor (repositories)          ← Hilt provides these automatically
    ↓
@Module classes                             ← tell Hilt HOW to build each object
  DatabaseModule  → Room + DAOs
  NetworkModule   → Retrofit + OkHttp
  FirebaseModule  → FirebaseAuth + Firestore
  RepositoryModule → interfaces bound to implementations
```

**`RepositoryModule.kt` — binds interfaces to implementations:**
```kotlin
@Binds @Singleton
abstract fun bindJobRepository(impl: JobRepositoryImpl): JobRepository
// When a ViewModel asks for JobRepository, Hilt injects JobRepositoryImpl
```

**`DatabaseModule.kt` — provides Room and DAOs:**
```kotlin
@Provides @Singleton
fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
    Room.databaseBuilder(context, AppDatabase::class.java, "job_manager_db").build()

@Provides
fun provideJobDao(db: AppDatabase): JobApplicationDao = db.jobApplicationDao()
```
`@Singleton` = one instance for the whole app lifetime. The database must be a singleton — two instances would have inconsistent state.

---

### Room Database

#### 5 Entities (tables)

| Entity | Table | Purpose |
|---|---|---|
| `UserEntity` | `users` | Logged-in user — name, email, dark theme flag |
| `JobApplicationEntity` | `job_applications` | A tracked job — title, company, status, date |
| `SkillEntity` | `skills` | A skill name |
| `InterviewNoteEntity` | `interview_notes` | A note linked to a job via foreign key |
| `JobSkillCrossRef` | *(junction table)* | Links jobs ↔ skills for Many-to-Many |

**`JobApplicationEntity.kt:6-14`:**
```kotlin
@Entity(tableName = "job_applications")
data class JobApplicationEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val company: String,
    val status: String,
    val dateApplied: String,
    val userId: Int = 0
)
```

#### Full CRUD — `JobApplicationDao.kt`

```kotlin
@Insert(onConflict = OnConflictStrategy.REPLACE)
suspend fun insertJob(job: JobApplicationEntity): Long    // CREATE

@Query("SELECT * FROM job_applications ORDER BY id DESC")
fun getAllJobs(): Flow<List<JobApplicationEntity>>         // READ all (live stream)

@Query("SELECT * FROM job_applications WHERE id = :jobId")
fun getJobById(jobId: Long): Flow<JobApplicationEntity?>  // READ single

@Update
suspend fun updateJob(job: JobApplicationEntity)          // UPDATE

@Delete
suspend fun deleteJob(job: JobApplicationEntity)          // DELETE
```

`InterviewNoteDao.kt` has the same full CRUD: `insertNote`, `updateNote`, `deleteNote`, `getNotesForJob`.

**Data persists across restarts** because Room writes to an SQLite file on device storage. When the app restarts, `getAllJobs()` reads back everything previously saved automatically.

---

### Relationships

#### One-to-Many — `JobWithNotes.kt`

One job → many interview notes. The note has a `jobId` foreign key pointing to its parent:

```kotlin
data class JobWithNotes(
    @Embedded val job: JobApplicationEntity,
    @Relation(parentColumn = "id", entityColumn = "jobId")
    val notes: List<InterviewNoteEntity>
)
```
`@Embedded` includes all job columns inline. `@Relation` tells Room to fetch all notes where `notes.jobId == job.id`. `onDelete = CASCADE` on `InterviewNoteEntity` means deleting a job automatically deletes all its notes.

#### Many-to-Many — `JobWithSkills.kt` + `JobSkillCrossRef`

A job can require many skills. A skill can belong to many jobs. The junction table stores the pairs:

```kotlin
@Entity(primaryKeys = ["jobId", "skillId"])
data class JobSkillCrossRef(val jobId: Long, val skillId: Long)

data class JobWithSkills(
    @Embedded val job: JobApplicationEntity,
    @Relation(
        parentColumn = "id", entityColumn = "id",
        associateBy = Junction(
            value = JobSkillCrossRef::class,
            parentColumn = "jobId", entityColumn = "skillId"
        )
    )
    val skills: List<SkillEntity>
)
```
`Junction` tells Room to go through `JobSkillCrossRef` when resolving which skills belong to which job.

---

### Repository Pattern

The chain is always: **ViewModel → Repository interface → RepositoryImpl → DAO → Room**. No ViewModel imports a DAO directly.

**`JobRepositoryImpl.kt:40-50` — full CRUD with mapper:**
```kotlin
override suspend fun addJob(job: Job): Long = jobDao.insertJob(job.toEntity())
override suspend fun updateJob(job: Job)    = jobDao.updateJob(job.toEntity())
override suspend fun deleteJob(job: Job)    = jobDao.deleteJob(job.toEntity())
override fun getJobs(): Flow<List<Job>>     = jobDao.getAllJobs().map { list -> list.map { it.toDomain() } }
```
The repository runs the mapper (`toEntity()` / `toDomain()`) so the ViewModel only ever sees `Job` (domain model), never `JobApplicationEntity` (Room entity).

**`JobRepositoryImpl.kt:26-35` — 5 flows combined into one stats object:**
```kotlin
override fun getJobStats(): Flow<JobStats> {
    return combine(
        jobDao.getJobCount(),
        jobDao.getJobCountByStatus("Applied"),
        jobDao.getJobCountByStatus("Interviewing"),
        jobDao.getJobCountByStatus("Offer Received"),
        jobDao.getJobCountByStatus("Rejected")
    ) { t, a, i, o, r ->
        JobStats(total = t, applied = a, interviewing = i, offerReceived = o, rejected = r)
    }
}
```
Every time any count changes in Room, this emits an updated `JobStats` automatically.

---

### Coroutines

Coroutines are used at every layer. No blocking calls on the main thread anywhere.

| Usage | Example | Where |
|---|---|---|
| Background work | `viewModelScope.launch { }` | Every ViewModel action |
| Suspend DB write | `suspend fun insertJob(...)` | All DAO write operations |
| Suspend network | `.await()` on Firebase Task | `AuthRepositoryImpl` |
| Live stream | `Flow<List<Job>>` | All DAO read operations |
| Combine streams | `combine(flow1, flow2) { }` | `JobRepositoryImpl`, `HomeViewModel`, `LoginViewModel` |
| Convert to StateFlow | `.stateIn(viewModelScope, ...)` | Every ViewModel |

**`JobDetailViewModel.kt:42-61` — two parallel coroutines in `init`:**
```kotlin
init {
    viewModelScope.launch {
        jobRepository.getJobById(jobId).collect { job -> ... }    // watches the job
    }
    viewModelScope.launch {
        jobRepository.getNotesForJob(jobId).collect { notes -> ... }  // watches notes
    }
}
```
Both run simultaneously. When either the job or its notes change in Room, the corresponding coroutine fires and updates `_uiState`.

---

### Full requirement checklist

| Requirement | Status | Where |
|---|---|---|
| 5+ ViewModels | ✅ 9 | All in `ui/view_model/` |
| 1 screen = 1 ViewModel | ✅ | Every screen has its own VM |
| UiState on every ViewModel | ✅ | All 9 have a sealed class UiState |
| Stateful + stateless screens | ✅ | Login, Register, Dashboard, JobList, JobDetails, Profile |
| StateFlow / MutableStateFlow | ✅ | Every ViewModel |
| Hilt throughout | ✅ | All VMs, all repos, all modules |
| Room connected | ✅ | `AppDatabase`, 4 DAOs |
| Full CRUD | ✅ | `JobApplicationDao`, `InterviewNoteDao` |
| Data persists across restarts | ✅ | SQLite file on device storage |
| 5 Room entities | ✅ | User, Job, Skill, Note, CrossRef |
| DAO per entity | ✅ | UserDao, JobApplicationDao, SkillDao, InterviewNoteDao |
| One-to-Many relationship | ✅ | Job → Notes (`JobWithNotes`) |
| Many-to-Many relationship | ✅ | Job ↔ Skills (`JobWithSkills` + `JobSkillCrossRef`) |
| Repository pattern | ✅ | 5 repository interfaces + implementations |
| ViewModels use repositories | ✅ | No ViewModel imports a DAO |
| Coroutines everywhere | ✅ | `viewModelScope.launch`, `suspend`, `Flow`, `.await()` |
| No blocking on main thread | ✅ | All slow work inside coroutines |

---

### Likely defense questions

| Question | Key answer |
|---|---|
| How many ViewModels do you have? | 9 — one per screen plus AuthViewModel in MainActivity |
| What does a UiState sealed class give you? | Exhaustive state representation — compiler enforces handling Init, Loading, Success, Error |
| What is `@HiltViewModel`? | Tells Hilt to manage this ViewModel's creation and inject its dependencies |
| What does `@Binds` do? | Tells Hilt: when this interface is requested, inject this implementation |
| What is `@Singleton`? | One instance created for the app lifetime — required for Room database |
| How many Room entities do you have? | 5 — UserEntity, JobApplicationEntity, SkillEntity, InterviewNoteEntity, JobSkillCrossRef |
| What is a DAO? | Interface declaring database queries — Room generates the SQL implementation |
| What is `onDelete = CASCADE`? | When a parent job is deleted, all its child notes are deleted automatically |
| What is the junction table for? | Enables Many-to-Many — `JobSkillCrossRef` stores jobId + skillId pairs |
| Why does the ViewModel never import a DAO? | Repository pattern — ViewModel depends on the interface, not the data source |
| What does `toEntity()` / `toDomain()` do? | Mapper functions — convert between domain model (Job) and Room entity (JobApplicationEntity) |
| What is `viewModelScope`? | Coroutine scope tied to ViewModel — all coroutines cancel when ViewModel is destroyed |

---

## Q10j: Network Layer — Retrofit & Firebase Integration

---

### Retrofit — Two DTOs

**`HabitDto.kt`** — the response DTO, used when reading from the API:
```kotlin
data class HabitDto(
    @SerializedName("id")          val id: Int?,
    @SerializedName("title")       val title: String?,
    @SerializedName("description") val description: String?,
    @SerializedName("frequency")   val frequency: String?,
    @SerializedName("completed")   val completed: Boolean?,
    @SerializedName("user_id")     val userId: Int?
)
```
All nullable — API fields can be missing. `@SerializedName` maps the JSON key to the Kotlin field name.

**`CreateHabitDto.kt`** — the request DTO, used when sending data to the API (POST/PUT body):
```kotlin
data class CreateHabitDto(
    @SerializedName("title")       val title: String,
    @SerializedName("description") val description: String,
    @SerializedName("frequency")   val frequency: String,
    @SerializedName("user_id")     val userId: Int
)
```
Non-nullable — you control what you send. Two DTOs because the API's request and response shapes are different.

---

### Retrofit Service Interface — `JobApiService.kt` — all 4 HTTP methods

```kotlin
interface JobApiService {
    @GET("habits/")
    suspend fun getHabits(): Response<List<HabitDto>>                              // READ all

    @GET("habits/{id}")
    suspend fun getHabitById(@Path("id") id: Int): Response<HabitDto>             // READ one

    @POST("habits/")
    suspend fun createHabit(@Body habit: CreateHabitDto): Response<HabitDto>      // CREATE

    @PUT("habits/{id}")
    suspend fun updateHabit(@Path("id") id: Int, @Body habit: CreateHabitDto): Response<HabitDto>  // UPDATE

    @DELETE("habits/{id}")
    suspend fun deleteHabit(@Path("id") id: Int): Response<Unit>                  // DELETE
}
```

- `@GET / @POST / @PUT / @DELETE` — HTTP method + URL path
- `@Path("id")` — substitutes `{id}` in the URL with the actual value at runtime
- `@Body` — serializes the Kotlin object to JSON as the request body
- `suspend` — coroutine-native, no callbacks needed
- `Response<T>` — wraps result so you can check `isSuccessful`, `code()`, and `body()`

---

### NetworkModule — Hilt DI (`NetworkModule.kt`)

Built in 4 steps, each `@Provides @Singleton`:

```kotlin
// Step 1 — logging interceptor (prints full requests/responses to Logcat)
fun provideLoggingInterceptor(): HttpLoggingInterceptor =
    HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BODY }

// Step 2 — OkHttp client with two interceptors wired together
fun provideOkHttpClient(logging: HttpLoggingInterceptor): OkHttpClient =
    OkHttpClient.Builder()
        .addInterceptor(logging)
        .addInterceptor { chain ->
            val request = chain.request().newBuilder()
                .addHeader("X-Authentication", "yes")  // added to EVERY request automatically
                .build()
            chain.proceed(request)
        }.build()

// Step 3 — Retrofit pointed at the backend
fun provideRetrofit(client: OkHttpClient): Retrofit =
    Retrofit.Builder()
        .baseUrl("http://10.0.2.2:8000/")              // emulator alias for localhost
        .addConverterFactory(GsonConverterFactory.create())  // JSON ↔ Kotlin auto-conversion
        .client(client).build()

// Step 4 — the actual API service (Retrofit generates the implementation at runtime)
fun provideJobApiService(retrofit: Retrofit): JobApiService =
    retrofit.create(JobApiService::class.java)
```

`10.0.2.2` is the Android emulator's special alias for the host machine's localhost. The app manifest also has `android:usesCleartextTraffic="true"` because Android 9+ blocks plain HTTP by default.

---

### NetworkResult — the error wrapper (`NetworkResult.kt`)

```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T)                          : NetworkResult<T>()
    data class Error(val message: String, val code: Int? = null): NetworkResult<Nothing>()
    object Loading                                              : NetworkResult<Nothing>()
}
```
Every async operation returns one of these instead of throwing exceptions across layers. The ViewModel `when`-switches on it safely. `out T` makes it covariant.

---

### Repository Implementation — `RemoteJobRepositoryImpl.kt`

All 5 CRUD operations, all wrapped in `safeApiCall`:

```kotlin
override suspend fun fetchRemoteJobs(): NetworkResult<List<Job>> = safeApiCall {
    val response = apiService.getHabits()
    if (response.isSuccessful) {
        NetworkResult.Success(response.body()?.map { it.toDomain() } ?: emptyList())
    } else {
        NetworkResult.Error("HTTP ${response.code()}", response.code())
    }
}

// Safety net — catches network-level exceptions (no internet, timeout, crash)
private inline fun <T> safeApiCall(block: () -> NetworkResult<T>): NetworkResult<T> =
    try { block() } catch (e: Exception) { NetworkResult.Error(e.message ?: "Network error") }
```

Two levels of error handling: `response.isSuccessful` catches HTTP errors (4xx, 5xx). `safeApiCall` catches network-level exceptions (no connection, timeout).

The mapper converts between DTOs and domain models (`RemoteJobMapper.kt`):
```kotlin
fun HabitDto.toDomain(): Job = Job(
    id      = (id ?: 0).toLong(),
    title   = title ?: "Untitled",
    company = description ?: "",   // "description" maps to "company"
    status  = if (completed == true) "Offer Received" else "Applied",
    dateApplied = "N/A",
    userId  = userId ?: 0
)

fun Job.toCreateHabitDto(): CreateHabitDto = CreateHabitDto(
    title = title, description = company, frequency = "Once", userId = userId
)
```

---

### ViewModel connected to network — `RemoteJobViewModel.kt`

```kotlin
fun loadRemoteJobs() {
    viewModelScope.launch {
        _uiState.update { RemoteJobUiState.Loading }
        when (val result = remoteJobRepository.fetchRemoteJobs()) {
            is NetworkResult.Success -> _uiState.update { RemoteJobUiState.Success(result.data) }
            is NetworkResult.Error   -> _uiState.update { RemoteJobUiState.Error(result.message) }
            else -> Unit
        }
    }
}
```

`JobListingScreen.kt:134-160` reacts with a `when(remoteUiState)` block — spinner while loading, list on success, red error text on failure. The button is disabled during loading (`enabled = remoteUiState !is RemoteJobUiState.Loading`).

---

### Full Retrofit flow

```
User taps "Load Jobs from Server"
    ↓
RemoteJobViewModel.loadRemoteJobs()
    ↓
viewModelScope.launch { }              — coroutine starts
    ↓
_uiState = Loading                     — spinner appears
    ↓
remoteJobRepository.fetchRemoteJobs()
    ↓
safeApiCall { apiService.getHabits() } — Retrofit makes HTTP GET to 10.0.2.2:8000/habits/
    Header: X-Authentication: yes      — interceptor adds this automatically
    ↓
JSON response received
    ↓
Gson parses JSON → List<HabitDto>
    ↓
response.isSuccessful → .map { it.toDomain() } → List<Job>
    ↓
NetworkResult.Success(jobs)
    ↓
_uiState = RemoteJobUiState.Success(jobs)
    ↓
JobListingScreen recomposes → LazyColumn displays the list
```

---

### Firebase Auth — Sign Up (`AuthRepositoryImpl.kt:28-43`)

```kotlin
override suspend fun register(email: String, password: String, fullName: String): NetworkResult<FirebaseUser> =
    try {
        val result = firebaseAuth.createUserWithEmailAndPassword(email, password).await()
        val user = result.user!!
        user.updateProfile(
            UserProfileChangeRequest.Builder().setDisplayName(fullName).build()
        ).await()                  // saves the display name to the Firebase profile
        NetworkResult.Success(user)
    } catch (e: Exception) {
        NetworkResult.Error(e.message ?: "Registration failed")
    }
```
Two chained Firebase calls: create the account → update the profile name. Both `.await()` so they run sequentially in the coroutine without blocking.

---

### Firebase Auth — Sign In (`AuthRepositoryImpl.kt:19-26`)

```kotlin
override suspend fun signIn(email: String, password: String): NetworkResult<FirebaseUser> =
    try {
        val result = firebaseAuth.signInWithEmailAndPassword(email, password).await()
        NetworkResult.Success(result.user!!)
    } catch (e: Exception) {
        NetworkResult.Error(e.message ?: "Sign-in failed")
    }
```
Wrong credentials → Firebase throws an exception → catch converts it to `NetworkResult.Error` → `LoginUiState.Error(message)` → red error text shown above the Login button (`LoginScreen.kt:119-126`).

---

### Firebase Auth — Logout (`AuthRepositoryImpl.kt:45`)

```kotlin
override suspend fun signOut() = firebaseAuth.signOut()
```
`signOut()` clears Firebase's local session cache. `AuthViewModel.signOut()` sets `_authState = LoggedOut` → `MainActivity`'s `LaunchedEffect` navigates to Login and clears the entire back stack with `popUpTo(0) { inclusive = true }`.

---

### Persistent Login — `AuthRepositoryImpl.kt:17` + `AuthViewModel.kt:28-32`

```kotlin
override fun isLoggedIn(): Boolean = firebaseAuth.currentUser != null
```
`firebaseAuth.currentUser` is **synchronous** — Firebase caches the session token locally on the device. On app restart, no network call is made. If the user was logged in and never signed out, `currentUser` is non-null immediately.

```kotlin
init {  // AuthViewModel — runs before first frame is drawn
    _authState.value = if (authRepository.isLoggedIn())
        AuthCheckState.LoggedIn else AuthCheckState.LoggedOut
}
```
`MainActivity` reads this to choose `startDestination` — Home or Login — before NavHost renders. No loading screen, no flash.

---

### Firestore Collection — `FirestoreJobRepositoryImpl.kt`

**Data structure:**
```
Firestore
└── users/
    └── {firebaseUid}/
        └── jobs/
            └── {jobId}  →  { id, title, company, status, dateApplied, userId }
```

Each user has their own `jobs` sub-collection — data is isolated per user. The path is built by:
```kotlin
private fun userJobsCollection(userId: String) =
    firestore.collection("users").document(userId).collection("jobs")
```

Full CRUD is implemented: `addJob` (set), `updateJob` (set/overwrite), `deleteJob` (delete), `getJobsForUser` (realtime listener).

---

### Realtime Updates — `FirestoreJobRepositoryImpl.kt:19-40`

```kotlin
override fun getJobsForUser(userId: String): Flow<NetworkResult<List<Job>>> = callbackFlow {
    trySend(NetworkResult.Loading)

    val listener = userJobsCollection(userId).addSnapshotListener { snapshot, error ->
        if (error != null) {
            trySend(NetworkResult.Error(error.message ?: "Firestore error"))
            return@addSnapshotListener
        }
        val jobs = snapshot?.documents?.mapNotNull { doc ->
            val data = doc.data ?: return@mapNotNull null
            Job(id = (data["id"] as? Long) ?: 0L, title = (data["title"] as? String) ?: "", ...)
        } ?: emptyList()
        trySend(NetworkResult.Success(jobs))
    }

    awaitClose { listener.remove() }  // removes listener when nobody is collecting — prevents memory leak
}
```

`addSnapshotListener` fires **every time** the Firestore collection changes — from any device, in real time. `callbackFlow` bridges this callback into a Kotlin Flow. `awaitClose { listener.remove() }` is critical — removes the listener when the screen is gone.

---

### Error Handling — where errors surface in the UI

| Error type | Where caught | Where shown in UI |
|---|---|---|
| Wrong password / bad email | `AuthRepositoryImpl` catch | `LoginScreen.kt:119-126` — red text above button |
| Network down (Retrofit) | `safeApiCall` catch | `JobListingScreen.kt:153-158` — red error text |
| HTTP 4xx/5xx error | `response.isSuccessful` check | Same — `"HTTP 404"` message shown |
| Firestore error | `addSnapshotListener` error param | `callbackFlow` emits `NetworkResult.Error` |
| Add job failed | `DashboardViewModel` catch block | Snackbar at bottom of screen |

Every error path ends in a UI update — no silent failures.

---

### Likely defense questions

| Question | Key answer |
|---|---|
| What is Retrofit? | Type-safe HTTP client — you define an interface, it generates the implementation |
| What are the two DTOs and why two? | `HabitDto` (response, nullable) and `CreateHabitDto` (request, non-nullable) — request and response shapes differ |
| What does `@SerializedName` do? | Maps a JSON key (e.g. `"user_id"`) to a Kotlin field name (`userId`) |
| What does `@Body` do? | Serializes a Kotlin object to JSON and sends it as the HTTP request body |
| What does `@Path` do? | Substitutes a value into a URL template — `{id}` becomes `123` |
| What is `safeApiCall`? | Inline function wrapping every call in try/catch — returns `NetworkResult.Error` on any exception |
| What is `Response<T>`? | Wraps HTTP result — check `isSuccessful` for HTTP errors, `body()` for the data |
| Why `10.0.2.2`? | Emulator's alias for the host PC's localhost — `localhost` inside the emulator is the emulator itself |
| What is persistent login? | Firebase caches the session token locally — `currentUser` is non-null on restart without a network call |
| What does `signOut()` do? | Clears the local Firebase session cache — `currentUser` becomes null |
| What is `callbackFlow`? | Converts a callback-based API (Firestore listener) into a Kotlin Flow |
| What does `awaitClose` do? | Removes the Firestore listener when the Flow has no collectors — prevents memory leaks |
| What is realtime updates? | `addSnapshotListener` fires every time Firestore data changes — UI updates without any user action |
| How is error shown for wrong password? | Firebase throws exception → catch → `NetworkResult.Error` → `LoginUiState.Error` → red text in UI |

---

## Q11: Basic Logic & State — Where It Lives in Your Code

This section covers the milestone requirement: **use state to manage inputs and UI, at least one validation rule, and at least one edge case**. Your code meets and exceeds all three.

---

### 1. State Managing Inputs & UI Changes

**`LoginState.kt:3`** — the form's state model:
```kotlin
data class LoginState(
    val email: String = "",
    val password: String = "",
    val passwordVisible: Boolean = false
)
```
The ViewModel holds this in a `MutableStateFlow`. Every keystroke calls `_state.update { it.copy(email = email) }` (`LoginViewModel.kt:51`). The screen collects it with `collectAsStateWithLifecycle()` and recomposes automatically on every change — no manual refresh needed.

**`AddJobForm.kt:25-26`** — local state using `remember`:
```kotlin
var jobTitle by remember { mutableStateOf("") }
var company  by remember { mutableStateOf("") }
```
`remember { mutableStateOf(...) }` is the Compose way to keep state alive across recompositions *within a single composable*, without needing a ViewModel.

---

### 2. Validation Rules (Minimum: 1 — You have 5+)

All validators live in `AuthValidator.kt`:

| Function | Rules |
|---|---|
| `validateEmail` (line 7) | blank check → regex format check via `Patterns.EMAIL_ADDRESS` |
| `validatePassword` (line 15) | blank check → length must be ≥ 6 characters |
| `validateFullName` (line 23) | blank check |
| `validateConfirmPassword` (line 27) | blank check → must exactly match the password field |

**Disable submit button until valid — `LoginViewModel.kt:47-49`:**
```kotlin
val isLoginEnabled = combine(_state, emailError, passwordError) { s, eErr, pErr ->
    eErr == null && pErr == null && s.email.isNotBlank() && s.password.isNotBlank()
}
```
`combine` merges three flows. The button is only enabled when *all* conditions pass simultaneously.

**Show error message — `LoginScreen.kt:105-106`:**
```kotlin
isError = emailError != null,
errorMessage = emailError
```
The field turns red and shows the message string when the validator returns non-null.

**Second example — `AddJobForm.kt:28-48`:**
```kotlin
val isTitleTooShort = jobTitle.isNotEmpty() && jobTitle.length < 3
```
Shows "Job title must be at least 3 characters." below the field, and `enabled = isFormValid` disables the button.

---

### 3. Edge Cases (Minimum: 1 — You have 3)

**Empty list state — `JobListingScreen.kt:92-99`:**
```kotlin
if (uiState.jobs.isEmpty()) {
    Text(
        text = if (searchQuery.isBlank()) "No jobs yet. Add one from Dashboard!"
               else "No jobs found matching '$searchQuery'"
    )
}
```
Instead of a blank screen, the user gets a human-readable message.

**Prevent blank form submission — `DashboardViewModel.kt:60-63`:**
```kotlin
fun addJob(title: String, company: String, ...) {
    if (title.isBlank()) {
        _addJobState.update { AddJobState.Error("Title cannot be empty") }
        return
    }
```
Even though the UI button is already disabled, the ViewModel guards the same edge case — defensive programming at two layers.

**Prevent double-tap during loading — `LoginScreen.kt:61`:**
```kotlin
enabled = isLoginEnabled && uiState !is LoginUiState.Loading
```
Once Login is tapped and Firebase is processing, the button disables itself. Prevents a second request being fired before the first finishes.

---

### Quick reference map

| Concept | File | Line(s) |
|---|---|---|
| State model | `LoginState.kt` | 3–7 |
| Local state (remember) | `AddJobForm.kt` | 25–26 |
| ViewModel state + combine | `LoginViewModel.kt` | 36–49 |
| Validation rules (4 rules) | `AuthValidator.kt` | 7–33 |
| Disable button when invalid | `LoginViewModel.kt` / `LoginScreen.kt` | 47–49 / 130 |
| Show inline error message | `LoginScreen.kt` | 105–106 |
| Empty list edge case | `JobListingScreen.kt` | 92–99 |
| Blank form guard (ViewModel) | `DashboardViewModel.kt` | 60–63 |
| Button disabled during loading | `LoginScreen.kt` | 61 |

---

## Q12: BONUS — Extra Edge Cases & Extra Validation Explained

### BONUS Edge Case 1: Two sub-states inside "empty list" — `JobListingScreen.kt:92-99`

The minimum requirement is just showing *something* when the list is empty. Your code does more:

```kotlin
if (uiState.jobs.isEmpty()) {
    Text(
        text = if (searchQuery.isBlank()) "No jobs yet. Add one from Dashboard!"
               else "No jobs found matching '$searchQuery'"
    )
}
```

There are **two different reasons** the list can be empty:
1. The user genuinely has no jobs yet — `searchQuery.isBlank()` is true → "No jobs yet. Add one from Dashboard!"
2. The user searched for something that matched nothing — `searchQuery.isBlank()` is false → "No jobs found matching 'X'"

One `if` block, two meaningful messages based on the actual cause. This is a proper edge case because the same *state* (empty list) has two different *causes* that warrant different responses.

---

### BONUS Edge Case 2: Button locked during network request — `JobListingScreen.kt:128`

```kotlin
Button(
    onClick = onLoadRemoteJobs,
    enabled = remoteUiState !is RemoteJobUiState.Loading
)
```

While the API call is in flight (`RemoteJobUiState.Loading`), the "Load Jobs from Server" button is disabled. Without this guard, the user could:
- Tap the button 5 times
- Fire 5 simultaneous network requests
- Receive 5 overlapping responses, causing unpredictable UI

The button re-enables when loading finishes (Success, Error, or Idle). Same pattern is used on the Login button (`LoginScreen.kt:61`).

---

### BONUS Validation 1: Register form validates 4 fields at once — `RegisterScreen.kt:59-75`

```kotlin
val fullNameError      = AuthValidators.validateFullName(fullName)
val emailError         = AuthValidators.validateEmail(email)
val passwordError      = AuthValidators.validatePassword(password)
val confirmPasswordError = AuthValidators.validateConfirmPassword(password, confirmPassword)

val isRegisterEnabled = registerState !is RegisterUiState.Loading &&
        fullNameError == null &&
        emailError == null &&
        passwordError == null &&
        confirmPasswordError == null &&
        fullName.isNotBlank() && email.isNotBlank() &&
        password.isNotBlank() && confirmPassword.isNotBlank()
```

The register button is disabled unless **all 8 conditions** are true simultaneously. Every validator is evaluated on every keystroke in any field — the check is fully reactive.

---

### BONUS Validation 2: Cross-field validation — `AuthValidator.kt:27-33`

```kotlin
fun validateConfirmPassword(password: String, confirmPassword: String): String? {
    return when {
        confirmPassword.isBlank() -> "Please confirm your password"
        confirmPassword != password -> "Passwords do not match"
        else -> null
    }
}
```

This is a **cross-field validation** — it compares two separate inputs against each other. Most validators only check one field. This one requires both the `password` and `confirmPassword` fields as inputs and fails if they don't match. It's called with both values live: `AuthValidators.validateConfirmPassword(password = password, confirmPassword = confirmPassword)` (`RegisterScreen.kt:62-65`), so the error appears *in real time as the user types* the confirm password field.

---

### BONUS Validation 3: Email format regex — `AuthValidator.kt:10`

```kotlin
!Patterns.EMAIL_ADDRESS.matcher(email).matches() -> "Please enter a valid email"
```

`Patterns.EMAIL_ADDRESS` is Android's built-in compiled regex for RFC-valid email addresses. This catches inputs like `hello`, `hello@`, `hello@x` that are not blank but are not valid emails. It runs *after* the blank check so the error message is accurate — you get "Email is required" for blank, "Please enter a valid email" for a typed but malformed address.

---

### Summary: What you have vs. what was required

| Requirement | Minimum | Your Code |
|---|---|---|
| Use state for inputs | 1 state holder | `LoginState` (ViewModel) + `remember` in `AddJobForm` — 2 approaches |
| Validation rules | 1 rule | 5+ rules across 4 validators: blank, format, length, match |
| Disable button | ✅ | ✅ — disabled when invalid AND disabled during loading |
| Show error message | ✅ | ✅ — inline under each field, plus auth error above button |
| Edge cases | 1 | 3: empty list, blank form ViewModel guard, button lock during request |
| BONUS: extra edge cases | — | 2-sub-state empty message + network button lock |
| BONUS: extra validation | — | 4-field simultaneous validation + cross-field + regex format |

---

## Q11: The Architecture Pitch — 60-second answer

> "The app follows **MVVM** — Model, View, ViewModel — with a single Activity and Jetpack Compose for all UI.
>
> The **View** layer is pure Compose composables. They observe state from the ViewModel and emit events back — they have no business logic.
>
> The **ViewModel** holds UI state as `StateFlow`s, handles validation, and calls the repository. It survives configuration changes so state isn't lost on rotation.
>
> The **Model** layer has two data sources: Room for local SQLite storage, and remote sources — Firebase Auth and Firestore for user auth and cloud job storage, plus a Retrofit client for an external habits API. Repositories abstract these sources so the ViewModel doesn't know or care where data comes from.
>
> Dependency injection is handled by Hilt — it wires ViewModels, repositories, Firebase, Room, and Retrofit together automatically. All async work uses Kotlin Coroutines and Flow."

---

### Likely defense questions — all three topics

| Question | Key answer |
|---|---|
| Why one Activity? | Single Activity + Compose Navigation is the modern Android pattern |
| What does `@AndroidEntryPoint` do? | Tells Hilt to inject into this Activity — required for `hiltViewModel()` to work |
| How does the app know if the user is already logged in? | `AuthViewModel` checks `FirebaseAuth.currentUser` synchronously on launch |
| Why `remember { startDestination }`? | Computed once — not recalculated on every recomposition |
| What does `popUpTo(0)` on logout do? | Clears the entire back stack so the user can't go Back into the app |
| What is `launchSingleTop`? | Prevents creating duplicate copies of a screen when navigating to it |
| What is Retrofit? | Type-safe HTTP client — you define an interface, it generates the implementation |
| What does `@Body` do? | Serializes a Kotlin object to JSON and sends it as the request body |
| What does `@Path` do? | Substitutes a value into a URL template, e.g. `{id}` → `123` |
| What is an interceptor? | Middleware that runs on every HTTP request — used for logging and adding headers |
| Why `10.0.2.2` instead of `localhost`? | The emulator's alias for the host machine's localhost |
| What is `GsonConverterFactory`? | Automatically converts JSON responses into Kotlin data classes |
| What is `safeApiCall`? | Inline helper that wraps network calls in try/catch, returns `NetworkResult.Error` on failure |
| What is a DTO? | Data Transfer Object — raw API shape; mapped to a domain model before the ViewModel sees it |
| Explain your architecture in one minute | MVVM: Compose View → ViewModel with StateFlow → Repository → Room/Firebase/Retrofit, wired by Hilt |

---
