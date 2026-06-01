# Project Defense Preparation Guide
## Job Tracker App — Android Mobile Programming 2025/2026

---

# Table of Contents

1. [How to Introduce Your App](#1-how-to-introduce-your-app)
2. [Project Architecture — MVVM](#2-project-architecture--mvvm)
3. [Folder Structure Explained](#3-folder-structure-explained)
4. [Dependency Injection — Hilt](#4-dependency-injection--hilt)
5. [Room Database — Local Storage](#5-room-database--local-storage)
6. [Coroutines and Asynchronous Programming](#6-coroutines-and-asynchronous-programming)
7. [StateFlow and State Management](#7-stateflow-and-state-management)
8. [Navigation Compose](#8-navigation-compose)
9. [Retrofit — Network Layer](#9-retrofit--network-layer)
10. [Firebase Authentication](#10-firebase-authentication)
11. [Firebase Firestore — Cloud Database](#11-firebase-firestore--cloud-database)
12. [How All Three Databases Work Together](#12-how-all-three-databases-work-together)
13. [Screen-by-Screen Walkthrough](#13-screen-by-screen-walkthrough)
14. [Expected Defense Questions and Answers](#14-expected-defense-questions-and-answers)
15. [Key Files Cheat Sheet](#15-key-files-cheat-sheet)
16. [Concepts Glossary](#16-concepts-glossary)

---

# 1. How to Introduce Your App

## What to Say in the First 30 Seconds

> "I built a job application tracker called LifeRPG. Users can register and log in using Firebase Authentication. Once logged in, they can track their job applications — add, edit, delete jobs and write interview notes. All data is stored locally using Room database and also synced to the cloud using Firebase Firestore. The app also connects to a REST API using Retrofit to browse available job listings fetched from an external server."

## What Problems Does It Solve?

- Keeps track of where you applied for jobs, the status, and interview notes
- Data is available offline (Room) and also backed up in the cloud (Firestore)
- You can browse new job listings from the server without leaving the app

## App Screens

| Screen | Purpose |
|---|---|
| Login | Email + password authentication via Firebase |
| Register | Create a new account via Firebase |
| Home | Overview — stats about your applications |
| Job Listing | Browse your tracked jobs + load jobs from API |
| Job Details | View/edit a specific job, add interview notes |
| Dashboard | Add new jobs, view statistics |
| Profile | View account info, toggle dark mode, logout |

---

# 2. Project Architecture — MVVM

## What is MVVM?

MVVM stands for **Model — View — ViewModel**. It is a design pattern that separates your app into three distinct layers so that each part has one job.

```
┌─────────────────────────────────────────────────────┐
│                      VIEW (UI)                       │
│          Composable screens — display only           │
│         Collects state, sends user events            │
└──────────────────────┬──────────────────────────────┘
                       │  calls functions / observes state
┌──────────────────────▼──────────────────────────────┐
│                    VIEWMODEL                         │
│         Holds business logic and UI state            │
│    Calls repositories, updates StateFlow             │
└──────────────────────┬──────────────────────────────┘
                       │  calls repository functions
┌──────────────────────▼──────────────────────────────┐
│                      MODEL                           │
│  Repository → DAO → Room / Firebase / Retrofit       │
│        Data classes, entities, DTOs                  │
└─────────────────────────────────────────────────────┘
```

## The Golden Rule

> **The UI never talks directly to the database. The ViewModel never imports Room or Firebase. Screens only display state.**

## Why MVVM?

- **Separation of concerns** — each layer does one thing
- **Testable** — you can test the ViewModel without the UI
- **Survives rotation** — ViewModel outlives the screen, state is not lost
- **Reactive** — UI automatically updates when data changes

## Data Flow Example — Adding a Job

```
User types title and taps "Add Job"
        ↓
DashboardScreen calls viewModel.addJob("Google", "SWE", "Applied")
        ↓
DashboardViewModel validates input, creates a Job object
        ↓
Calls jobRepository.addJob(job)         → saves to Room (SQLite)
Calls firestoreJobRepository.addJob()   → syncs to cloud (Firestore)
        ↓
_addJobState.update { AddJobState.Success }
        ↓
DashboardScreen sees new state via StateFlow → shows success message
```

---

# 3. Folder Structure Explained

```
com.example.mobileprog/
│
├── model/                          ← DATA LAYER
│   ├── data/
│   │   ├── Job.kt                  ← Domain model (what the app uses internally)
│   │   ├── User.kt
│   │   ├── local/
│   │   │   ├── entity/             ← Room database tables
│   │   │   │   ├── UserEntity.kt
│   │   │   │   ├── JobApplicationEntity.kt
│   │   │   │   ├── SkillEntity.kt
│   │   │   │   ├── InterviewNoteEntity.kt
│   │   │   │   └── JobSkillCrossRef.kt  ← junction table
│   │   │   ├── dao/                ← Database query interfaces
│   │   │   │   ├── UserDao.kt
│   │   │   │   ├── JobApplicationDao.kt
│   │   │   │   ├── SkillDao.kt
│   │   │   │   └── InterviewNoteDao.kt
│   │   │   ├── db/
│   │   │   │   └── AppDatabase.kt  ← Room database definition
│   │   │   └── util/
│   │   │       ├── JobWithNotes.kt  ← One-to-Many relationship
│   │   │       └── JobWithSkills.kt ← Many-to-Many relationship
│   │   └── remote/                 ← NETWORK LAYER (NEW in Assignment 4)
│   │       ├── NetworkResult.kt    ← Loading/Success/Error wrapper
│   │       ├── dto/
│   │       │   ├── HabitDto.kt     ← API response shape
│   │       │   └── CreateHabitDto.kt ← API request shape
│   │       └── api/
│   │           └── JobApiService.kt ← Retrofit interface
│   ├── di/                         ← DEPENDENCY INJECTION
│   │   ├── DatabaseModule.kt       ← Provides Room + DAOs
│   │   ├── NetworkModule.kt        ← Provides Retrofit + OkHttp
│   │   ├── FirebaseModule.kt       ← Provides FirebaseAuth + Firestore
│   │   └── RepositoryModule.kt     ← Binds interfaces to implementations
│   └── repository/                 ← REPOSITORY LAYER
│       ├── JobRepository.kt        ← Interface (Room-based)
│       ├── JobRepositoryImpl.kt
│       ├── UserRepository.kt
│       ├── UserRepositoryImpl.kt
│       ├── RemoteJobRepository.kt  ← Interface (Retrofit-based)
│       ├── RemoteJobRepositoryImpl.kt
│       ├── AuthRepository.kt       ← Interface (Firebase Auth)
│       ├── AuthRepositoryImpl.kt
│       ├── FirestoreJobRepository.kt ← Interface (Firestore)
│       ├── FirestoreJobRepositoryImpl.kt
│       └── mappers/
│           ├── JobMapper.kt        ← Entity ↔ Domain conversion
│           ├── UserMapper.kt
│           └── RemoteJobMapper.kt  ← DTO ↔ Domain conversion
│
├── ui/                             ← PRESENTATION LAYER
│   ├── navigation/
│   │   ├── NavGraph.kt             ← All routes defined here
│   │   └── Screen.kt              ← Route name constants
│   ├── theme/                      ← Colors, typography, dark mode
│   ├── view/
│   │   └── component/
│   │       └── screen/
│   │           ├── LoginScreen/    ← Login + Register screens
│   │           ├── home/           ← Home screen
│   │           ├── JobListing/     ← Job list screen
│   │           ├── job_details/    ← Job detail screen
│   │           ├── dashboard/      ← Dashboard screen
│   │           └── profile/        ← Profile screen
│   └── view_model/                 ← ALL VIEWMODELS
│       ├── LoginViewModel.kt
│       ├── RegisterViewModel.kt
│       ├── HomeViewModel.kt
│       ├── JobListViewModel.kt
│       ├── JobDetailViewModel.kt
│       ├── DashboardViewModel.kt
│       ├── ProfileViewModel.kt
│       ├── AuthViewModel.kt        ← NEW: manages login state
│       └── RemoteJobViewModel.kt   ← NEW: fetches from Retrofit
│
└── MainActivity.kt                 ← Single entry point, bottom nav
```

---

# 4. Dependency Injection — Hilt

## What is Dependency Injection?

Imagine a ViewModel needs a Repository, and the Repository needs a DAO, and the DAO needs a Database. Without DI, you would manually create each one and pass them through constructors everywhere. With Hilt, you just declare what you need and Hilt provides it automatically.

## Without Hilt (painful)

```kotlin
val database = Room.databaseBuilder(...).build()
val dao = database.jobDao()
val repository = JobRepositoryImpl(dao)
val viewModel = DashboardViewModel(repository)
```

## With Hilt (clean)

```kotlin
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val jobRepository: JobRepository   // Hilt provides this automatically
) : ViewModel()
```

## Your Three Hilt Modules

### DatabaseModule — provides Room
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "job_manager_db").build()

    @Provides
    fun provideJobDao(db: AppDatabase): JobApplicationDao = db.jobApplicationDao()
    // ... other DAOs
}
```

### NetworkModule — provides Retrofit
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor { chain ->
                // Adds "X-Authentication: yes" to every request
                chain.proceed(chain.request().newBuilder()
                    .addHeader("X-Authentication", "yes").build())
            }.build()

    @Provides @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("http://10.0.2.2:8000/")  // emulator → localhost
            .addConverterFactory(GsonConverterFactory.create())
            .client(client).build()

    @Provides @Singleton
    fun provideJobApiService(retrofit: Retrofit): JobApiService =
        retrofit.create(JobApiService::class.java)
}
```

### FirebaseModule — provides Firebase
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object FirebaseModule {
    @Provides @Singleton
    fun provideFirebaseAuth(): FirebaseAuth = FirebaseAuth.getInstance()

    @Provides @Singleton
    fun provideFirestore(): FirebaseFirestore = FirebaseFirestore.getInstance()
}
```

### RepositoryModule — connects interfaces to implementations
```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds @Singleton
    abstract fun bindJobRepository(impl: JobRepositoryImpl): JobRepository

    @Binds @Singleton
    abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
    // ... all 5 repositories
}
```

## Key Annotations to Know

| Annotation | Meaning |
|---|---|
| `@HiltAndroidApp` | On the Application class — starts Hilt |
| `@AndroidEntryPoint` | On MainActivity — allows Hilt injection |
| `@HiltViewModel` | On ViewModel — Hilt manages its creation |
| `@Inject constructor` | "Hilt, please provide these parameters" |
| `@Module` | This class provides dependencies |
| `@InstallIn(SingletonComponent::class)` | These live for the whole app lifetime |
| `@Provides` | This function creates a dependency |
| `@Binds` | This interface is implemented by this class |
| `@Singleton` | Create only one instance, reuse everywhere |

---

# 5. Room Database — Local Storage

## What is Room?

Room is Android's official library for SQLite. It lets you define database tables as Kotlin data classes (entities), define queries as interfaces (DAOs), and use them without writing raw SQL.

## Your Four Entities

### UserEntity — the logged-in user's local profile
```kotlin
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val fullName: String,
    val email: String,
    val isDarkTheme: Boolean = false
)
```

### JobApplicationEntity — a job the user is tracking
```kotlin
@Entity(tableName = "job_applications")
data class JobApplicationEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val company: String,
    val status: String,        // "Applied", "Interviewing", "Offer Received", "Rejected"
    val dateApplied: String,
    val userId: Int
)
```

### SkillEntity — a skill required for a job
```kotlin
@Entity(tableName = "skills")
data class SkillEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String
)
```

### InterviewNoteEntity — notes written after an interview
```kotlin
@Entity(
    tableName = "interview_notes",
    foreignKeys = [ForeignKey(
        entity = JobApplicationEntity::class,
        parentColumns = ["id"],
        childColumns = ["jobId"],
        onDelete = CASCADE    // if job deleted, its notes are deleted too
    )]
)
data class InterviewNoteEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val jobId: Long,          // links to the job
    val note: String,
    val date: String
)
```

## Relationships

### One-to-Many: One Job has Many Notes
```kotlin
data class JobWithNotes(
    @Embedded val job: JobApplicationEntity,
    @Relation(parentColumn = "id", entityColumn = "jobId")
    val notes: List<InterviewNoteEntity>
)
```
One job can have zero or more interview notes. When you delete a job, all its notes are automatically deleted (CASCADE).

### Many-to-Many: Jobs ↔ Skills (via junction table)
```kotlin
@Entity(primaryKeys = ["jobId", "skillId"])
data class JobSkillCrossRef(
    val jobId: Long,
    val skillId: Long
)
```
A job can require many skills. A skill can belong to many jobs. The junction table stores the combinations.

## What is a DAO?

DAO = Data Access Object. It's an interface where you declare database operations. Room generates the SQL automatically.

```kotlin
@Dao
interface JobApplicationDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertJob(job: JobApplicationEntity): Long   // returns the new ID

    @Update
    suspend fun updateJob(job: JobApplicationEntity)

    @Delete
    suspend fun deleteJob(job: JobApplicationEntity)

    @Query("SELECT * FROM job_applications ORDER BY id DESC")
    fun getAllJobs(): Flow<List<JobApplicationEntity>>       // reactive — auto-updates UI

    @Query("SELECT COUNT(*) FROM job_applications WHERE status = :status")
    fun getJobCountByStatus(status: String): Flow<Int>
}
```

## Why Return Flow from DAOs?

When a DAO function returns `Flow<T>`, Room automatically emits a new value every time the table changes. This means:
- You query once
- The UI automatically updates whenever data changes
- No manual refresh needed

---

# 6. Coroutines and Asynchronous Programming

## What is a Coroutine?

A coroutine is a way to run code asynchronously (in the background) without blocking the main thread. If you ran a database query on the main thread, the app would freeze until it finished. Coroutines let you write async code that looks like normal sequential code.

## The Problem Without Coroutines

```kotlin
// THIS WOULD CRASH — you cannot do database work on the main thread
val jobs = database.jobDao().getAllJobs()  // blocks the UI thread
```

## The Solution With Coroutines

```kotlin
// Inside a ViewModel — runs on background thread, doesn't block UI
viewModelScope.launch {
    val userId = userRepository.getUser().first()?.id ?: 0
    jobRepository.addJob(job)   // runs in background
    _addJobState.update { AddJobState.Success }  // back on main thread
}
```

## Key Coroutine Concepts

| Concept | Meaning |
|---|---|
| `suspend fun` | A function that can be paused and resumed without blocking a thread |
| `viewModelScope.launch` | Start a coroutine that lives as long as the ViewModel |
| `Flow<T>` | A stream that emits values over time (like a reactive list) |
| `.await()` | Wait for a Firebase Task to complete (from `kotlinx-coroutines-play-services`) |
| `Dispatchers.IO` | Background thread pool for I/O operations |

## How Firebase Uses Coroutines

Firebase operations return `Task<T>`. The `kotlinx-coroutines-play-services` library adds `.await()` so you can use them in coroutines:

```kotlin
suspend fun signIn(email: String, password: String): NetworkResult<FirebaseUser> =
    try {
        val result = firebaseAuth
            .signInWithEmailAndPassword(email, password)
            .await()   // waits for Firebase to respond without blocking the thread
        NetworkResult.Success(result.user!!)
    } catch (e: Exception) {
        NetworkResult.Error(e.message ?: "Sign-in failed")
    }
```

---

# 7. StateFlow and State Management

## What is StateFlow?

StateFlow is a data holder that:
- Always has a current value
- Notifies all collectors when the value changes
- Is the recommended way to expose state from a ViewModel to the UI

## The UiState Pattern

Every ViewModel has a sealed class that represents all possible states of the screen:

```kotlin
sealed class LoginUiState {
    object Init : LoginUiState()        // just opened the screen
    object Loading : LoginUiState()     // request in progress
    object Success : LoginUiState()     // login worked
    data class Error(val message: String) : LoginUiState()  // login failed
}
```

## How the ViewModel Exposes State

```kotlin
// Private — only the ViewModel can change this
private val _uiState = MutableStateFlow<LoginUiState>(LoginUiState.Init)

// Public — the screen can only read this, not modify it
val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()
```

## How the Screen Reacts to State

```kotlin
@Composable
fun LoginScreen(viewModel: LoginViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (uiState) {
        is LoginUiState.Loading -> CircularProgressIndicator()
        is LoginUiState.Success -> { /* navigate to home */ }
        is LoginUiState.Error   -> Text((uiState as LoginUiState.Error).message)
        else -> { /* show the form */ }
    }
}
```

## Stateful vs Stateless Composables

Every screen has two versions — this is required by the assignment:

```kotlin
// STATEFUL — knows about the ViewModel, connects to real data
@Composable
fun LoginScreen(viewModel: LoginViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    LoginScreen(   // calls the stateless version
        uiState = uiState,
        onLoginClick = { viewModel.login() }
    )
}

// STATELESS — only receives data and callbacks, easy to preview and test
@Composable
private fun LoginScreen(
    uiState: LoginUiState,
    onLoginClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    // just UI, no business logic
}
```

---

# 8. Navigation Compose

## How Navigation Works

```kotlin
// Screen.kt — all route names in one place
sealed class Screen(val route: String) {
    data object Login   : Screen("login_screen")
    data object Home    : Screen("home_screen")
    data object JobDetails : Screen("job_details_screen/{jobId}") {
        fun createRoute(jobId: Long) = "job_details_screen/$jobId"
    }
    // ...
}
```

```kotlin
// NavGraph.kt — all destinations wired up
NavHost(navController, startDestination = startDestination) {
    composable(Screen.Login.route) {
        LoginScreen(onLoginClick = {
            navController.navigate(Screen.Home.route) {
                popUpTo(Screen.Login.route) { inclusive = true }  // clears login from back stack
            }
        })
    }
    composable(
        route = Screen.JobDetails.route,
        arguments = listOf(navArgument("jobId") { type = NavType.LongType })
    ) {
        JobDetailScreen(onBack = { navController.popBackStack() })
    }
}
```

## Passing Arguments Between Screens

```kotlin
// Navigate and pass a job ID
navController.navigate(Screen.JobDetails.createRoute(jobId = 42L))
// → goes to "job_details_screen/42"

// Receive it in the ViewModel via SavedStateHandle
@HiltViewModel
class JobDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val jobId: Long = checkNotNull(savedStateHandle["jobId"])  // reads "42"
}
```

## Back Stack Management

```kotlin
navController.navigate(Screen.Home.route) {
    popUpTo(Screen.Login.route) { inclusive = true }
    // removes Login from the back stack so pressing Back doesn't go back to Login
}
```

## Persistent Login and Start Destination

```kotlin
// MainActivity.kt
val startDestination = remember {
    if (authViewModel.authState.value == AuthCheckState.LoggedIn)
        Screen.Home.route    // already logged in → go straight to Home
    else
        Screen.Login.route   // not logged in → show Login
}
NavHost(startDestination = startDestination) { ... }
```

`FirebaseAuth.currentUser` is a synchronous check (no network call). If a user was previously logged in, Firebase caches their session locally. So on startup, we immediately know if they're logged in and skip the Login screen entirely.

---

# 9. Retrofit — Network Layer

## What is Retrofit?

Retrofit is a library that turns your API description (an interface with annotations) into real HTTP calls. You describe your API, Retrofit handles the HTTP, JSON parsing, errors, and threading.

## The Flow

```
Button tap in UI
    ↓
RemoteJobViewModel.loadRemoteJobs()
    ↓
RemoteJobRepositoryImpl.fetchRemoteJobs()
    ↓
JobApiService.getHabits()      ← Retrofit makes the HTTP request
    ↓
GET http://10.0.2.2:8000/habits/
Header: X-Authentication: yes
    ↓
JSON response: [{"title":"...", "description":"...", ...}]
    ↓
Gson parses JSON → List<HabitDto>
    ↓
HabitDto.toDomain() → List<Job>
    ↓
RemoteJobUiState.Success(jobs) → UI displays the list
```

## DTO — Data Transfer Object

A DTO is a simple data class whose fields exactly match the JSON the API returns. It is NOT your domain model — it's just the shape of the API response.

**API returns this JSON:**
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

**Your DTO matches it exactly:**
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

**The mapper converts DTO → Domain model:**
```kotlin
fun HabitDto.toDomain(): Job = Job(
    id = (id ?: 0).toLong(),
    title = title ?: "Untitled",
    company = description ?: "",
    status = if (completed == true) "Offer Received" else "Applied",
    dateApplied = "N/A",
    userId = userId ?: 0
)
```

## Why Not Use the Domain Model Directly?

- API field names are different (`user_id` vs `userId`)
- API has fields you don't need (`frequency`)
- If the API changes, you only update the DTO, not your whole app
- DTOs can be nullable — domain models should not be

## The NetworkResult Wrapper

Instead of throwing exceptions directly, every network operation returns a sealed class:

```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val message: String) : NetworkResult<Nothing>()
    object Loading : NetworkResult<Nothing>()
}
```

This forces the ViewModel to handle every case:
```kotlin
when (val result = remoteJobRepository.fetchRemoteJobs()) {
    is NetworkResult.Success -> _uiState.update { RemoteJobUiState.Success(result.data) }
    is NetworkResult.Error   -> _uiState.update { RemoteJobUiState.Error(result.message) }
    else -> Unit
}
```

## Why `10.0.2.2` Instead of `localhost`?

The Android emulator runs in a virtual machine. Its `localhost` is the emulator itself. To reach your PC's localhost (where the API server runs), you use `10.0.2.2` — the emulator's special alias for the PC's localhost.

## Why `usesCleartextTraffic="true"`?

Android 9+ blocks plain HTTP by default for security. Since the lab API uses `http://` (not `https://`), you must explicitly allow it in the manifest for development:
```xml
<application android:usesCleartextTraffic="true" ...>
```

---

# 10. Firebase Authentication

## What is Firebase Auth?

Firebase Auth is a ready-made authentication service. Instead of building your own login system (storing passwords securely, handling sessions, etc.), Firebase handles all of it. You just call `signIn()` and `createUser()`.

## Registration Flow

```
User fills: fullName, email, password, confirmPassword
    ↓
RegisterScreen validates all fields (client-side)
    ↓
Calls RegisterViewModel.register(fullName, email, password)
    ↓
AuthRepositoryImpl.register():
    1. firebaseAuth.createUserWithEmailAndPassword(email, password).await()
       → Firebase creates the account and returns a FirebaseUser
    2. user.updateProfile(displayName = fullName).await()
       → Saves the display name to Firebase
    3. userRepository.saveUser(User(fullName, email))
       → Also saves to local Room database
    ↓
RegisterUiState.Success → navigate to Home
```

## Login Flow

```
User enters email + password
    ↓
LoginViewModel.login()
    ↓
AuthRepositoryImpl.signIn(email, password):
    firebaseAuth.signInWithEmailAndPassword(email, password).await()
    → Firebase checks credentials, returns FirebaseUser
    ↓
LoginUiState.Success → navigate to Home
```

If login fails (wrong password, email not found), Firebase throws an exception. The catch block converts it to `NetworkResult.Error(e.message)` and the UI shows the error message.

## Persistent Login — Staying Logged In

Firebase stores the user's session token on the device. When the app restarts:

```kotlin
// AuthViewModel.kt — runs when the ViewModel is created
init {
    _authState.value = if (authRepository.isLoggedIn()) {
        AuthCheckState.LoggedIn    // Firebase found a cached session
    } else {
        AuthCheckState.LoggedOut   // no session found
    }
}

// AuthRepositoryImpl.kt
override fun isLoggedIn(): Boolean = firebaseAuth.currentUser != null
// FirebaseAuth.currentUser is synchronous — it checks local cache, no network call
```

## Logout Flow

```
User taps Logout button on ProfileScreen
    ↓
ProfileScreen calls onLogout() → goes to AuthViewModel.signOut()
    ↓
AuthRepositoryImpl.signOut():
    firebaseAuth.signOut()   // clears the local session cache
    ↓
_authState.value = AuthCheckState.LoggedOut
    ↓
LaunchedEffect in MainActivity sees LoggedOut state change
    ↓
navController.navigate(Screen.Login.route) { popUpTo(0) { inclusive = true } }
    → navigates to Login, clears entire back stack
```

---

# 11. Firebase Firestore — Cloud Database

## What is Firestore?

Firestore is Firebase's cloud NoSQL database. Data is organized as **collections** of **documents** (similar to folders and files, or tables and rows).

## Your Data Structure

```
Firestore
└── users/                          ← collection
    └── {firebaseUid}/              ← document (one per user)
        └── jobs/                   ← sub-collection
            └── {jobId}/            ← document (one per job)
                ├── id: 1
                ├── title: "Android Developer"
                ├── company: "Google"
                ├── status: "Applied"
                ├── dateApplied: "31.05.2026."
                └── userId: 0
```

## Writing to Firestore

Every time a job is added via Dashboard:

```kotlin
// DashboardViewModel.kt
val localId = jobRepository.addJob(job)          // save to Room first
val firebaseUid = authRepository.currentUser?.uid
if (firebaseUid != null) {
    firestoreJobRepository.addJob(firebaseUid, job.copy(id = localId))  // then sync to cloud
}
```

```kotlin
// FirestoreJobRepositoryImpl.kt
override suspend fun addJob(userId: String, job: Job): NetworkResult<Unit> =
    try {
        userJobsCollection(userId)
            .document(job.id.toString())
            .set(job.toMap())    // converts Job to a Map<String, Any>
            .await()
        NetworkResult.Success(Unit)
    } catch (e: Exception) {
        NetworkResult.Error(e.message ?: "Failed to add job to cloud")
    }
```

## Realtime Updates — callbackFlow

Firestore can push changes to the app instantly. When any document in your jobs collection changes (from any device), your app receives the update automatically:

```kotlin
override fun getJobsForUser(userId: String): Flow<NetworkResult<List<Job>>> = callbackFlow {
    trySend(NetworkResult.Loading)

    // Register a listener — Firestore calls this every time data changes
    val listener = userJobsCollection(userId).addSnapshotListener { snapshot, error ->
        if (error != null) {
            trySend(NetworkResult.Error(error.message ?: "Firestore error"))
            return@addSnapshotListener
        }
        val jobs = snapshot?.documents?.mapNotNull { doc ->
            // manually extract each field from the document
            val data = doc.data ?: return@mapNotNull null
            Job(
                id = (data["id"] as? Long) ?: 0L,
                title = (data["title"] as? String) ?: "",
                // ...
            )
        } ?: emptyList()
        trySend(NetworkResult.Success(jobs))   // emit the new list
    }

    awaitClose { listener.remove() }   // clean up when the flow is cancelled
}
```

The `awaitClose { listener.remove() }` is critical — it removes the listener when the screen is gone, preventing memory leaks.

---

# 12. How All Three Databases Work Together

## Visual Overview

```
┌─────────────────────────────────────────────────────┐
│                    YOUR APP                          │
│                                                      │
│  ┌──────────┐    ┌────────────┐    ┌──────────────┐ │
│  │   ROOM   │    │ FIRESTORE  │    │   RETROFIT   │ │
│  │  SQLite  │    │   Cloud    │    │   REST API   │ │
│  │  Local   │    │   NoSQL    │    │  External    │ │
│  └────┬─────┘    └─────┬──────┘    └──────┬───────┘ │
│       │                │                  │          │
│  Always works     Sync & realtime    Browse only     │
│  offline          updates            (read only)     │
└───────┼────────────────┼──────────────────┼──────────┘
        │                │                  │
   JobRepository  FirestoreRepo      RemoteJobRepo
        │                │                  │
   DashboardVM     DashboardVM       RemoteJobVM
   JobListVM       (on addJob)       (on button tap)
```

## When Each Is Used

| Action | Room | Firestore | Retrofit |
|---|---|---|---|
| Add a job | ✅ Saves locally | ✅ Syncs to cloud | ❌ |
| View my jobs | ✅ Reads from local | ❌ | ❌ |
| Delete a job | ✅ Deletes locally | ❌ Not yet wired | ❌ |
| App starts (offline) | ✅ Works fine | ❌ No connection | ❌ |
| Tap "Load from Server" | ❌ | ❌ | ✅ Fetches habits |
| Login / Register | ❌ Saves profile | ❌ | ❌ Firebase Auth |

## Why This Design Makes Sense

- **Room** = always available, fast, works offline. Primary source of truth.
- **Firestore** = cloud backup, realtime sync, would allow multi-device in future.
- **Retrofit** = demonstrating the network layer required by the assignment. In a real app, this would be a real job listings API.

---

# 13. Screen-by-Screen Walkthrough

## LoginScreen

- **ViewModel:** `LoginViewModel`
- **What it does:** Validates email/password format in real-time, calls Firebase Auth on submit
- **State:** `LoginUiState` (Init → Loading → Success/Error)
- **Key:** Real-time validation using `combine()` on two StateFlows — the login button is only enabled when both email and password pass validation

## RegisterScreen

- **ViewModel:** `RegisterViewModel`
- **What it does:** Validates all 4 fields (name, email, password, confirm password), creates Firebase account + local Room user
- **State:** `RegisterUiState` (Idle → Loading → Success/Error)
- **Key:** `isRegisterEnabled` computed from all 4 validator functions — button disabled until everything is valid

## HomeScreen

- **ViewModel:** `HomeViewModel`
- **What it does:** Combines two Flows (job stats + current user) into one UI state
- **State:** `HomeUiState.Success(stats, user)`
- **Key:** Uses `combine()` — whenever either jobs or user changes, the screen automatically updates

## JobListingScreen

- **ViewModels:** `JobListViewModel` + `RemoteJobViewModel`
- **What it does:** Shows local jobs with search, plus a button to load remote jobs from the API
- **State:** `JobListUiState` + `RemoteJobUiState`
- **Key:** Two completely separate data sources displayed in the same screen

## JobDetailsScreen

- **ViewModel:** `JobDetailViewModel`
- **What it does:** View/edit job title and status, add/delete interview notes
- **State:** `JobDetailUiState.Success(job, notes)`
- **Key:** Receives `jobId` as a navigation argument, loads the specific job from Room

## DashboardScreen

- **ViewModel:** `DashboardViewModel`
- **What it does:** Shows job statistics, form to add a new job
- **State:** `DashboardUiState` (stats) + `AddJobState` (add form)
- **Key:** Two separate StateFlows for two separate concerns on the same screen

## ProfileScreen

- **ViewModel:** `ProfileViewModel`
- **What it does:** Shows user info, theme toggle, logout button
- **State:** `ProfileUiState.Success(user)`
- **Key:** Theme preference saved to Room (persists across restarts), logout clears Firebase session

---

# 14. Expected Defense Questions and Answers

## Architecture Questions

**Q: What is MVVM and why did you use it?**
> MVVM separates the app into three layers. The View (Composables) only displays state and passes user events to the ViewModel. The ViewModel holds the logic, calls repositories, and exposes state via StateFlow. The Model contains the data — entities, DAOs, repositories. I used it because it keeps screens thin, state survives screen rotation since ViewModels outlive Composables, and each part is independently testable.

**Q: What is the Repository pattern?**
> A repository is an abstraction between the ViewModel and the data source. The ViewModel only calls repository functions — it doesn't know if data comes from Room, Firestore, or Retrofit. This means I can change where data comes from without touching the ViewModel or screen. For example, `JobRepository` hides Room, `FirestoreJobRepository` hides Firestore, `RemoteJobRepository` hides Retrofit — the ViewModel treats them all the same way.

**Q: Why is there an interface and an implementation for each repository?**
> The interface defines what operations are available (`addJob`, `getJobs`, etc.). The implementation contains the actual code. The ViewModel depends on the interface, not the implementation. This is the Dependency Inversion Principle — high-level modules don't depend on low-level details. Hilt's `@Binds` tells it which implementation to use for each interface.

---

## Assignment 4 Specific Questions

**Q: Explain your Retrofit setup from start to finish.**
> I have a `NetworkModule` Hilt module that creates: (1) an `OkHttpClient` with a logging interceptor and a custom interceptor that adds `X-Authentication: yes` to every request header. (2) A `Retrofit` instance pointing to `http://10.0.2.2:8000/` using Gson for JSON parsing and that OkHttpClient. (3) A `JobApiService` created from Retrofit. These are all singletons — created once and reused. The `RemoteJobRepositoryImpl` receives the `JobApiService` via Hilt injection.

**Q: What is a DTO and why not use the domain model directly?**
> DTO stands for Data Transfer Object. It's a data class whose fields exactly match the API's JSON structure. The API uses `user_id` while my domain model uses `userId`. The API has a `frequency` field I don't need. DTOs are also fully nullable since API responses can be unpredictable. My domain `Job` class is clean and non-nullable. A mapper function converts between them.

**Q: How does Firebase Auth persistent login work?**
> Firebase Auth stores the user's session token locally on the device. When the app starts, `FirebaseAuth.currentUser` synchronously returns the cached user without any network call. In `AuthViewModel.init`, I check `isLoggedIn()` which is just `firebaseAuth.currentUser != null`. Based on this, `MainActivity` sets the correct `startDestination` for the NavHost — either Home or Login — before the first frame is drawn. No flash, no redirect.

**Q: What are Firestore realtime updates and how did you implement them?**
> Firestore can push changes to the app without polling. I used `callbackFlow` — a Kotlin coroutine primitive that bridges callback-based APIs to Flow. Inside it, I register a Firestore `addSnapshotListener` which calls back every time the collection changes. I `trySend()` the new data into the flow each time. When the collecting scope is cancelled (e.g. the screen is gone), `awaitClose { listener.remove() }` removes the listener to prevent memory leaks.

**Q: Why is the base URL `10.0.2.2` and not `localhost`?**
> The Android emulator runs in a virtual machine. Inside the emulator, `localhost` refers to the emulator itself, not the PC. Android emulators have a special alias `10.0.2.2` that routes to the host machine's localhost. So `http://10.0.2.2:8000/` from the emulator reaches `http://localhost:8000/` on my PC where the API server is running.

---

## Room Questions

**Q: How many entities do you have and what are they?**
> Four entities: `UserEntity` (stores the logged-in user's name, email, theme preference), `JobApplicationEntity` (a job being tracked with title, company, status, date), `SkillEntity` (a skill name), `InterviewNoteEntity` (a note linked to a job). Plus `JobSkillCrossRef` as a junction table for the many-to-many relationship between jobs and skills.

**Q: Explain the relationships in your database.**
> I have two. First, One-to-Many: one `JobApplicationEntity` has many `InterviewNoteEntity` entries. The note has a `jobId` foreign key pointing to the job, with `CASCADE` delete so when a job is deleted, all its notes are deleted automatically. Second, Many-to-Many: jobs and skills are linked through `JobSkillCrossRef`. A job can require many skills; a skill can belong to many jobs. The cross-reference table stores `jobId + skillId` pairs.

**Q: How does Room data automatically update the UI?**
> DAO methods return `Flow<T>`. Room observes the database table and emits a new value every time the table changes. The repository passes this Flow to the ViewModel. The ViewModel converts it to a `StateFlow` using `stateIn()`. The Composable collects it with `collectAsStateWithLifecycle()` and recomposes whenever new data arrives. The whole chain is reactive — one database write triggers an automatic UI update.

---

## Hilt Questions

**Q: Why use dependency injection?**
> Without DI, I would manually create every object and pass it through constructors. A ViewModel needs a Repository, which needs a DAO, which needs a Database. With Hilt, I just declare what I need in the constructor and Hilt provides the right instance. It also manages lifetimes — `@Singleton` instances are created once and reused. It makes the code easier to maintain and test.

**Q: What is `@Singleton` and why does it matter?**
> `@Singleton` means Hilt creates the dependency once for the whole app lifetime and reuses the same instance everywhere. The database must be a singleton — if you had two database instances, they might have inconsistent state. Retrofit is also a singleton because creating it is expensive. Repository implementations are singletons so they can hold cached state if needed.

---

## Coroutines/StateFlow Questions

**Q: Why use coroutines instead of callbacks?**
> Callbacks lead to "callback hell" — deeply nested code that's hard to read and maintain. Coroutines let you write async code that looks sequential. `suspend fun signIn()` looks like a regular function but runs asynchronously. Error handling uses normal try/catch. Kotlin coroutines also automatically cancel when the ViewModel is cleared, preventing memory leaks.

**Q: What is `viewModelScope`?**
> It's a coroutine scope tied to the ViewModel's lifetime. When the ViewModel is cleared (screen is permanently gone), all coroutines in `viewModelScope` are automatically cancelled. This prevents situations where a background operation tries to update UI that no longer exists.

**Q: What is the difference between `StateFlow` and `MutableStateFlow`?**
> `MutableStateFlow` has both read and write access — only the ViewModel should hold this. `StateFlow` is read-only — the screen can observe values but cannot change them directly. The ViewModel exposes `_uiState.asStateFlow()` so screens get a read-only view. This ensures state can only be changed through the ViewModel's functions, keeping a single source of truth.

---

# 15. Key Files Cheat Sheet

| What to Explain | File to Open |
|---|---|
| Retrofit interface (GET/POST/PUT/DELETE) | `model/data/remote/api/JobApiService.kt` |
| DTO definition | `model/data/remote/dto/HabitDto.kt` |
| DTO → Domain mapping | `model/repository/mappers/RemoteJobMapper.kt` |
| NetworkModule (Hilt) | `model/di/NetworkModule.kt` |
| Firebase Auth implementation | `model/repository/AuthRepositoryImpl.kt` |
| Firestore realtime listener | `model/repository/FirestoreJobRepositoryImpl.kt` |
| Persistent login logic | `MainActivity.kt` (lines ~35-65) |
| Auth state management | `ui/view_model/AuthViewModel.kt` |
| UiState pattern example | `ui/view_model/LoginViewModel.kt` |
| Hilt modules overview | `model/di/RepositoryModule.kt` |
| Room entities | `model/data/local/entity/` folder |
| One-to-Many relationship | `model/data/local/util/JobWithNotes.kt` |
| Many-to-Many relationship | `model/data/local/util/JobWithSkills.kt` |
| Navigation + routes | `ui/navigation/NavGraph.kt` |
| Argument passing | `ui/view_model/JobDetailViewModel.kt` (SavedStateHandle) |
| Remote jobs in UI | `ui/view/component/screen/JobListing/JobListingScreen.kt` |

---

# 16. Concepts Glossary

| Term | Simple Explanation |
|---|---|
| **MVVM** | Design pattern: View displays, ViewModel thinks, Model stores |
| **Composable** | A Kotlin function annotated with `@Composable` that draws UI |
| **StateFlow** | A value holder that notifies observers when it changes |
| **Coroutine** | Lightweight thread for async work — looks synchronous |
| **suspend fun** | A function that can pause without blocking a thread |
| **Flow** | A stream of values emitted over time |
| **Room** | Android's SQLite wrapper library |
| **Entity** | A Kotlin data class annotated with `@Entity` — becomes a DB table |
| **DAO** | Data Access Object — interface with DB query functions |
| **Repository** | Abstraction layer between ViewModel and data sources |
| **Hilt** | Dependency injection framework for Android |
| **@Singleton** | Only one instance created for the whole app |
| **Retrofit** | HTTP client — turns interface into real network calls |
| **DTO** | Data Transfer Object — matches API JSON shape exactly |
| **Gson** | JSON ↔ Kotlin object conversion library |
| **OkHttp** | The HTTP engine Retrofit uses underneath |
| **Interceptor** | Code that runs on every HTTP request/response |
| **Firebase Auth** | Ready-made authentication service by Google |
| **Firestore** | Firebase's cloud NoSQL database |
| **callbackFlow** | Coroutine bridge for callback-based APIs like Firestore listeners |
| **NavHost** | Container that displays the current screen |
| **NavController** | Controls which screen is shown |
| **NavGraph** | Declares all screens and how to navigate between them |
| **popUpTo** | Removes screens from the back stack during navigation |
| **SavedStateHandle** | Survives process death — used to retrieve navigation arguments in ViewModel |
| **@SerializedName** | Maps a Kotlin field to a different JSON key name |
| **10.0.2.2** | Emulator's alias for the host PC's localhost |
| **usesCleartextTraffic** | Allows plain HTTP (Android blocks it by default since API 28) |
| **CASCADE** | When a parent record is deleted, child records are deleted automatically |
| **Junction table** | A table linking two tables in a Many-to-Many relationship |

---

*Good luck with the defense! Focus on being able to trace the data flow for each feature from the button tap all the way to the database and back.*
