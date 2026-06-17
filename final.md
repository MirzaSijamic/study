# Mobile Programming — Concept Study Guide

A concept-first walkthrough of the whole course: **what each thing is, how it's used, and a worked code example with explanation.** Read it once end to end, then use the section headers to revise. Code is idiomatic Kotlin/Jetpack Compose; you don't need any specific project to follow it.

## Contents
1. Android Studio, Project Structure & Debugging
2. Kotlin Fundamentals
3. Jetpack Compose — Declarative UI
4. Components (Buttons, Text Fields, Images, Menus, Dialogs)
5. State in Compose
6. Scrollable Lists (Lazy layouts)
7. Navigation & Intents
8. ViewModel & State Management (MVVM, Hilt, StateFlow)
9. Coroutines & Asynchronous Programming
10. Data Management Layer (Room)
11. APIs & Networking (Retrofit)
12. Cloud Integration (Firebase)
13. Themes, File Handling & Animations
14. Testing, Stability & Advanced Debugging

---

## 1. Android Studio, Project Structure & Debugging

### What mobile development is
A mobile app runs on a constrained, event-driven device: limited memory/CPU/battery, a screen that rotates, and an OS that can pause or kill your app at any time. **Android** is Google's mobile OS; you write apps in **Kotlin** (or Java) using the **Android SDK** (the libraries and tools that let your code talk to the OS).

### Activities and the lifecycle
An **Activity** is a single screen/entry point of an app. The OS drives it through a **lifecycle** via callbacks you can override:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { App() }   // sets the Compose UI for this Activity
    }
}
```

Key callbacks and their meaning: `onCreate` (screen is being created — set up UI), `onStart` (becoming visible), `onResume` (in the foreground, interactive), `onPause` (losing focus), `onStop` (no longer visible), `onDestroy` (being torn down). **Why it matters:** rotating the device destroys and recreates the Activity (`onDestroy` → `onCreate`), so any state held directly in the Activity is lost unless saved. This is the core reason `ViewModel` exists (Section 8).

### Native vs cross-platform
*Native* (Kotlin + Android SDK) gives full access to platform features and best performance, at the cost of writing separate code per platform. *Cross-platform* (Flutter, React Native) shares one codebase across Android and iOS but adds an abstraction layer and can lag platform features. This course is native Android.

### IDE, SDK, and project structure
**Android Studio** is the IDE (editor + build + emulator + debugger). The **SDK** is the platform code your app depends on. A project is organised roughly as:

```
app/
 ├─ src/main/java/...      ← your Kotlin code
 ├─ src/main/res/          ← resources (drawables, strings, themes)
 ├─ src/main/AndroidManifest.xml
 ├─ src/test/              ← local unit tests (JVM)
 └─ src/androidTest/       ← instrumented tests (device/emulator)
build.gradle.kts (project) + app/build.gradle.kts (module)
```

The **AndroidManifest.xml** declares the app's components (activities), permissions (e.g. `INTERNET`), and metadata. The OS reads it to know what your app contains.

### Gradle and dependencies
**Gradle** is the build system. It compiles code, merges resources, and pulls in **dependencies** (external libraries) you declare:

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("androidx.compose.material3:material3:1.2.0")
    implementation("androidx.room:room-runtime:2.6.1")
}
```

"Building" = turning source + resources + dependencies into an installable APK/AAB.

### Emulator vs real device
The **emulator (AVD)** is a virtual phone on your computer — convenient for quick iteration. A **real device** is still essential because it reveals true performance, real sensors, actual network behaviour, and OEM quirks the emulator can't reproduce. (Note: from the emulator, your dev machine's `localhost` is reached at `10.0.2.2` — relevant in Section 11.)

### Debugging methodology
Debugging is a disciplined loop, not guessing: **Reproduce → Isolate → Explain → Fix → Verify.** Distinguish the **symptom** (what you see) from the **cause** (why it happens), and use evidence over hunches. Common tools:

- **Logging (Logcat):** print runtime state with severity levels.
  ```kotlin
  private const val TAG = "LoginScreen"
  Log.d(TAG, "login tapped, email=$email")   // d = debug; also v/i/w/e
  ```
  Use consistent tags and messages that explain intent and state. Remove noisy logs from release builds.
- **Stack traces:** when an exception is thrown, the trace names the **exception type**, message, and the **file:line** of each call. Read top-down to find the **root cause** (often the deepest frame in *your* code), and beware **cascading errors** (one failure triggering others).
- **Breakpoints:** pause execution at a line, **step** through code (over/into/out), and **inspect runtime state** (variable values, the call stack) live in the debugger.
- **Profiling:** measure CPU, memory, and rendering to find performance problems (jank, leaks). Mobile devices have a strict **memory hierarchy** and limited RAM, so allocations and leaks matter.

**Exam-worthy facts:** rotation recreates the Activity (state loss); the Manifest declares components/permissions; Gradle manages the build and dependencies; debugging is Reproduce→Isolate→Explain→Fix→Verify; a stack trace's file:line points to where it broke.

---

## 2. Kotlin Fundamentals

Kotlin became Google's official Android language in 2017. It's concise, null-safe, fully interoperable with Java, and its features (lambdas, trailing-lambda syntax) are what make Jetpack Compose readable.

### Variables and immutability
`val` is read-only (assign once); `var` is mutable. **Prefer `val`** — immutability prevents a whole class of bugs.

```kotlin
val name = "Mira"     // cannot be reassigned
var count = 0         // can change
count += 1
```

### Types, inference, interpolation
Kotlin infers types but you can be explicit. String interpolation uses `$`:

```kotlin
val age: Int = 22            // explicit
val city = "Graz"            // inferred as String
val msg = "$name is $age, lives in ${city.uppercase()}"
```

Common types: `Int`, `Long`, `Double`, `Boolean`, `Char`, `String`. Use explicit types on public APIs for clarity.

### Null safety (a core feature)
A type is **non-nullable** by default; add `?` to allow null. The compiler then forces you to handle null:

```kotlin
val a: String = "hi"      // can never be null
val b: String? = null     // nullable

val len1 = b?.length              // safe call → Int? (null if b is null)
val len2 = b?.length ?: 0         // Elvis: fallback when null
val len3 = b!!.length             // !! asserts non-null → throws NPE if null (avoid)
```

`?.` short-circuits to null; `?:` supplies a default; `!!` forces and can crash. Idiomatic Kotlin avoids `!!`.

### Functions
```kotlin
fun add(a: Int, b: Int): Int { return a + b }
fun square(x: Int): Int = x * x                 // single-expression
fun greet(name: String, greeting: String = "Hi") = "$greeting, $name"  // default param
greet("Mira")                  // "Hi, Mira"
greet("Mira", greeting = "Yo") // named argument
```

**Default parameters** remove the need for many overloads; **named arguments** make calls readable. Functions can take other functions as parameters — the foundation for Compose's `onClick` callbacks.

### Object-oriented Kotlin
```kotlin
class User(val name: String, var age: Int) {     // primary constructor + properties
    val isAdult: Boolean get() = age >= 18        // custom getter (computed)
}

data class Point(val x: Int, val y: Int)          // auto equals/hashCode/toString/copy
val p2 = Point(1, 2).copy(y = 9)                  // copy → Point(1, 9); original unchanged

object Config { const val BASE_URL = "https://api.example.com/" }  // singleton

class Repo private constructor() {
    companion object { fun create() = Repo() }    // factory via companion object
}
```

`data class` gives you value semantics and `.copy()` (central to immutable state updates). Visibility modifiers: `public` (default), `private`, `internal`, `protected`.

### Functional programming
```kotlin
val double: (Int) -> Int = { x -> x * 2 }         // lambda
fun transform(n: Int, op: (Int) -> Int) = op(n)   // higher-order function
transform(5) { it * it }                          // trailing lambda + implicit `it` → 25
```

**Scope functions** run a block in the context of an object: `let` (transform/null-guard), `apply` (configure, returns receiver), `also` (side-effect, returns receiver), `run`, `with`.

```kotlin
val email = user.email?.let { it.trim().lowercase() } ?: "unknown"
val intent = Intent().apply { putExtra("id", 5) }   // configure then return the Intent
```

### Collections
Immutable (`listOf`, `setOf`, `mapOf`) vs mutable (`mutableListOf`, etc.). Rich operations return **new** collections:

```kotlin
val nums = listOf(1, 2, 3, 4, 5)
val evensSquared = nums.filter { it % 2 == 0 }.map { it * it }   // [4, 16]
val total = nums.sum()                                          // aggregation → 15
val byParity = nums.groupBy { it % 2 }                          // {1=[1,3,5], 0=[2,4]}
```

**Key idea:** `map`/`filter` don't mutate the source — they produce new lists. This mirrors how Compose state works.

**Exam-worthy facts:** `val` vs `var`; `?.`/`?:`/`!!` behaviour; `.copy()` returns a new object; `map`/`filter` return new collections; trailing-lambda syntax is why `Button(onClick = {...}) { ... }` reads cleanly.

---

## 3. Jetpack Compose — Declarative UI

### Imperative vs declarative
The old Android way was **imperative**: define XML layouts, then mutate views by hand (`findViewById`, `setText`). You had to keep each widget in sync with state. **Compose is declarative**: you write functions that *describe* the UI for a given state, and the framework updates the screen when state changes. The mental model is **UI = f(state)**.

### Composable functions
A composable is a `@Composable`-annotated function that emits UI (it returns `Unit`, it doesn't *return* views):

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello $name!")
}
```

It takes data as parameters and calls other composables. Composables compose hierarchically, which makes UI modular and reusable. The entry point is `setContent { ... }` inside an Activity.

### Recomposition
When state a composable reads changes, Compose **re-executes** that composable to produce updated UI — this is **recomposition**. Compose is smart: it skips composables whose inputs didn't change, redrawing only what's necessary.

### Layout composables: Column, Row, Box
```kotlin
@Composable
fun Profile() {
    Column(
        verticalArrangement = Arrangement.spacedBy(8.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Mira")
        Row(horizontalArrangement = Arrangement.SpaceBetween) {
            Text("Posts"); Text("Followers")
        }
        Box(contentAlignment = Alignment.Center) {
            Text("overlay")
        }
    }
}
```

- **Column** stacks children vertically; **Row** horizontally; **Box** layers children on top of each other (z-order).
- **Arrangement** positions children along the main axis; **Alignment** along the cross axis.

### The Modifier system
A **Modifier** decorates a composable: size, padding, background, click handling, etc. Modifiers **chain**, and **order matters**:

```kotlin
Text(
    "Tap me",
    modifier = Modifier
        .padding(16.dp)              // (A) outer space first
        .background(Color.Yellow)    // (B) background covers padding-inset area
        .padding(8.dp)               // (C) inner space inside the background
        .clickable { /* ... */ }
)
```

If you swap the two paddings, the yellow background's size changes — because each modifier wraps the result of the previous one. Common modifiers: `padding`, `size`/`width`/`height`/`fillMaxWidth`, `background`, `border`, `shadow`, `clip`, `clickable`.

### Core content composables
`Text`, `Image`, `Icon`, and `Spacer` (empty space). `Surface` and `Card` are container composables that apply Material elevation/shape/color:

```kotlin
Card(modifier = Modifier.padding(16.dp)) {
    Column(Modifier.padding(16.dp)) {
        Text("Title", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(4.dp))
        Text("Body text")
    }
}
```

### Reusable composables and @Preview
Break screens into small, parameterised composables and pass data in. Always expose a `modifier: Modifier = Modifier` parameter so callers can position your component. `@Preview` renders a composable in the IDE without running the app:

```kotlin
@Composable
fun UserCard(name: String, modifier: Modifier = Modifier) {
    Card(modifier) { Text(name, Modifier.padding(16.dp)) }
}

@Preview(showBackground = true)
@Composable
fun UserCardPreview() { UserCard(name = "Mira") }
```

You can have multiple previews (light/dark, different states) to inspect UI variations quickly.

**Exam-worthy facts:** declarative = UI is a function of state; recomposition re-runs composables when their state changes and skips the rest; Column/Row/Box differ by axis/stacking; modifier order changes the result; `@Preview` needs a no-arg composable.

---

## 4. Components (Buttons, Text Fields, Images, Menus, Dialogs)

### Buttons and event-driven programming
UI is **event-driven**: the user acts, an event fires, you update state. A button takes an `onClick` callback:

```kotlin
Button(onClick = { viewModel.save() }) { Text("Save") }
```

Material 3 expresses **button importance** through type: `Button` (filled, highest emphasis), `FilledTonalButton`, `OutlinedButton`, `ElevatedButton`, and `TextButton` (lowest). A `FloatingActionButton` (FAB) is the screen's primary action.

### TextField — input as controlled state
A text field doesn't store its own text; **you** hold the value in state and feed it back (controlled component):

```kotlin
var query by remember { mutableStateOf("") }
val isValid = query.length >= 3

OutlinedTextField(
    value = query,
    onValueChange = { query = it },          // event up: update state
    label = { Text("Search") },
    isError = !isValid,
    singleLine = true
)
```

The flow is: keystroke → `onValueChange` → state updates → recomposition → field shows new `value`. Basic validation is just a derived boolean (`isValid`).

### Selection components
- **Checkbox** — independent boolean(s), multiple can be on.
- **RadioButton** — mutually exclusive choice within a group.
- **Switch** — a boolean toggle.

```kotlin
var checked by remember { mutableStateOf(false) }
Checkbox(checked = checked, onCheckedChange = { checked = it })
```

Choose by semantics: multiple independent options → checkboxes; one-of-many → radio buttons.

### Enabled/disabled state
Components take an `enabled` flag and show visual feedback automatically:

```kotlin
Button(onClick = { submit() }, enabled = isValid) { Text("Submit") }
```

### Unidirectional Data Flow (UDF)
State flows **down** (as parameters) and events flow **up** (as callbacks). Components are told what to show and report what happened; they don't own app logic. This is the backbone pattern of Compose.

### Images, scaling, and remote loading
```kotlin
Image(
    painter = painterResource(R.drawable.avatar),
    contentDescription = "User avatar",          // accessibility; null if decorative
    contentScale = ContentScale.Crop,            // how the image fills its bounds
    modifier = Modifier.size(64.dp).clip(CircleShape)
)
```

`ContentScale` options: **Fit** (whole image visible, may letterbox), **Crop** (fills bounds, may clip edges), **FillBounds** (stretches, distorts), **Inside** (shrinks to fit, never enlarges). `contentDescription` feeds screen readers.

Remote images use **Coil**'s `AsyncImage`, with placeholder/error support:

```kotlin
AsyncImage(
    model = "https://example.com/pic.jpg",
    contentDescription = null,
    placeholder = painterResource(R.drawable.loading)
)
```

### Icons
`Icon(imageVector = Icons.Default.Settings, contentDescription = "Settings", tint = MaterialTheme.colorScheme.primary)`. Icons are semantic signals; tint them with theme colors.

### Menus, dialogs, snackbars
- **DropdownMenu** — a popup list of actions, often anchored in a `TopAppBar`'s overflow icon.
- **AlertDialog** — interrupts the user to confirm/decide; use sparingly (it has an interruption cost).
- **Snackbar** — a brief, non-blocking message at the bottom for success/error feedback.

```kotlin
var menuOpen by remember { mutableStateOf(false) }
IconButton(onClick = { menuOpen = true }) { Icon(Icons.Default.MoreVert, "Menu") }
DropdownMenu(expanded = menuOpen, onDismissRequest = { menuOpen = false }) {
    DropdownMenuItem(text = { Text("Refresh") }, onClick = { menuOpen = false; refresh() })
}

if (showDialog) {
    AlertDialog(
        onDismissRequest = { showDialog = false },
        title = { Text("Delete?") },
        text = { Text("This can't be undone.") },
        confirmButton = { TextButton(onClick = { delete() }) { Text("Delete") } },
        dismissButton = { TextButton(onClick = { showDialog = false }) { Text("Cancel") } }
    )
}
```

`TopAppBar`/`BottomAppBar` provide the standard top title/actions bar and bottom action bar.

**Exam-worthy facts:** a TextField is controlled (you own `value`, react to `onValueChange`); button type signals emphasis; ContentScale.Crop fills+clips while Fit letterboxes; UDF = state down, events up; dialogs interrupt, snackbars don't.

---

## 5. State in Compose

### State = the data the UI reacts to
**UI = function of state.** When state changes, the UI updates automatically through recomposition. State is any value that, when changed, should change what's on screen.

### remember and mutableStateOf
A plain variable inside a composable won't work — it's not observable and resets every recomposition. You need both pieces:

```kotlin
var count by remember { mutableStateOf(0) }
Button(onClick = { count++ }) { Text("Count: $count") }
```

- `mutableStateOf(0)` creates **observable** state — writing to it schedules recomposition.
- `remember { }` keeps that state **across recompositions** (so it isn't recreated each time).

Without `mutableStateOf`, taps don't recompose. Without `remember`, the value resets to 0 on every recomposition. You need both.

### remember vs rememberSaveable
`remember` survives recomposition but **not** configuration changes (rotation) or process death. `rememberSaveable` also survives those by saving to a Bundle:

```kotlin
var name by rememberSaveable { mutableStateOf("") }   // survives rotation
```

Use `rememberSaveable` for user input you don't want to lose on rotation; use the ViewModel (Section 8) for anything that must survive longer or belongs to app logic.

### Derived state
Don't store what you can compute. **Derive** values from existing state so there's a single source of truth:

```kotlin
val items = remember { mutableStateListOf<String>() }
val isEmpty = items.isEmpty()                 // simple derivation, recomputed on read

// derivedStateOf: only recompute (and recompose) when the RESULT changes
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}
```

`derivedStateOf` is an optimisation: it avoids recomposing on every tiny change when only a coarser computed value matters.

### Stateful vs stateless composables
- **Stateful**: owns state internally (e.g. holds its own `remember { mutableStateOf }`). Convenient but harder to reuse/test.
- **Stateless**: receives state and callbacks as parameters, owns nothing. Reusable, previewable, testable.

### State hoisting
**Hoisting** = moving state up to the caller, turning a composable stateless. The pattern is **"state down, events up"** with a single source of truth:

```kotlin
// Stateless: knows nothing about where the value lives
@Composable
fun NameField(value: String, onValueChange: (String) -> Unit, modifier: Modifier = Modifier) {
    OutlinedTextField(value = value, onValueChange = onValueChange, modifier = modifier)
}

// Stateful caller owns the state and passes it down
@Composable
fun SignupForm() {
    var name by rememberSaveable { mutableStateOf("") }
    NameField(value = name, onValueChange = { name = it })
}
```

Benefits: the child is reusable and testable; the parent has the single source of truth; data flow is unidirectional. Not everything must be hoisted — purely local UI state (e.g. whether a dropdown is open) can stay inside the component.

### Classifying state by placement
Ask "where should this live?" — scalar UI state (a toggle) can be local; form/collection/business state usually belongs higher up or in a ViewModel. Right placement = the lowest common ancestor that needs it.

**Exam-worthy facts:** need `remember` **and** `mutableStateOf`; `rememberSaveable` survives rotation; derive instead of duplicating state; hoisting makes composables stateless and reusable; "data down, events up".

---

## 6. Scrollable Lists (Lazy layouts)

### Eager vs lazy composition
A `Column` composes **all** its children immediately — fine for a few items, disastrous for thousands. **Lazy layouts** only compose the items currently visible (plus a small buffer), recycling as you scroll. **Key insight:** use lazy layouts for long or unknown-length lists.

### LazyColumn / LazyRow
```kotlin
LazyColumn(
    contentPadding = PaddingValues(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    item { Text("Header") }                       // single item
    items(users, key = { it.id }) { user ->       // list of items, with stable keys
        UserRow(user)
    }
}
```

You describe content in a DSL: `item { }` for one, `items(list) { }` for many.

### Item keys
By default Compose tracks items by position. Providing a stable **key** (e.g. `key = { it.id }`) lets Compose correctly track items across insertions/removals/reorders — preserving state and animations, and avoiding wasted recomposition. **Rule:** keys must be unique and stable.

### LazyListState and programmatic scrolling
`rememberLazyListState()` exposes scroll info and lets you scroll in code:

```kotlin
val listState = rememberLazyListState()
val scope = rememberCoroutineScope()
// observable: listState.firstVisibleItemIndex, firstVisibleItemScrollOffset
Button(onClick = { scope.launch { listState.animateScrollToItem(0) } }) {
    Text("Top")
}
LazyColumn(state = listState) { /* ... */ }
```

### Grids and staggered grids
`LazyVerticalGrid(columns = GridCells.Fixed(2))` lays items in a grid; `LazyVerticalStaggeredGrid` lets items have different heights (Pinterest-style) instead of forcing a uniform row height.

### Spacing
Three ways: `contentPadding` (space around the whole list, including the scrollable edges), `Arrangement.spacedBy(n)` (space *between* items), and per-item `Modifier.padding` (space inside an item).

### Advanced features
- **Sticky headers**: `stickyHeader { }` keeps a section header pinned while its items scroll.
- **Item animations**: `Modifier.animateItemPlacement()` animates reorders/insertions (works with stable keys).
- **Empty / loading states**: branch on your data — show a spinner while loading, a message when empty, the list when populated.

### Performance principles
Only compose visible items (lazy handles this), minimise recompositions, don't do heavy work inside item composables, and never nest two scrollables that scroll the same direction.

**Exam-worthy facts:** `Column` is eager (composes all), `LazyColumn` is lazy (composes visible only); stable keys preserve item identity; `contentPadding` vs `spacedBy` vs item padding; programmatic scroll needs a coroutine scope.

---

## 7. Navigation & Intents

### The Navigation component
Before, moving between screens meant manual fragment transactions and back-stack juggling. The **Navigation Compose** library centralises this: a single **NavController** owns the back stack, and a **NavHost** maps **routes** (strings) to composable destinations.

```kotlin
@Composable
fun AppNav() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = "home") {
        composable("home") {
            HomeScreen(onOpenDetails = { id -> navController.navigate("details/$id") })
        }
        composable(
            route = "details/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.IntType })
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getInt("itemId") ?: 0
            DetailsScreen(itemId = id)
        }
    }
}
```

**Rule:** create the `NavController` once at the top (with `rememberNavController()`) and pass navigation events down as callbacks (keeping screens reusable and testable).

### Routes as sealed classes
Hard-coded strings are error-prone. A **sealed class** centralises routes and argument building:

```kotlin
sealed class Screen(val route: String) {
    data object Home : Screen("home")
    data object Details : Screen("details/{itemId}") {
        fun create(itemId: Int) = "details/$itemId"
    }
}
navController.navigate(Screen.Details.create(42))
```

Sealed classes give compile-time safety and a single place to manage routes.

### Arguments
Arguments are encoded in the route. **Required** args are part of the path (`details/{itemId}`) with a declared `NavType`; **optional** args use query syntax (`?sort={sort}`) with defaults. With Hilt + a ViewModel, the argument is read from the injected `SavedStateHandle` rather than passed manually (see Section 8).

### Bottom navigation and per-tab state
A `NavigationBar` with `NavigationBarItem`s switches top-level destinations. To keep each tab's scroll/state when switching, navigate with:

```kotlin
navController.navigate(route) {
    popUpTo(navController.graph.startDestinationId) { saveState = true }
    launchSingleTop = true
    restoreState = true
}
```

`launchSingleTop` avoids stacking duplicates; `saveState`/`restoreState` preserve each tab. **Nested graphs** group related destinations under a parent route.

### Intents
An **Intent** is Android's mechanism to start a component or ask the system to perform an action. *Explicit* intents target a specific component; *implicit* intents describe an action and let the OS pick a handler (e.g. share text, open a URL):

```kotlin
val share = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "Check this out!")
}
context.startActivity(Intent.createChooser(share, "Share via"))
```

**Exam-worthy facts:** NavController owns the back stack; routes are strings, best organised in a sealed class; required args live in the path with a NavType; `launchSingleTop`/`saveState`/`restoreState` manage tab behaviour; implicit intents delegate an action to the OS.

---

## 8. ViewModel & State Management (MVVM, Hilt, StateFlow)

### Why a ViewModel?
On rotation the OS destroys and recreates the Activity/composition — any state held there is lost. A **ViewModel** is **retained across configuration changes**, so state stored in it survives rotation. It also separates UI logic from the UI itself.

| State location | Survives recomposition | Survives rotation | Survives process death |
|---|---|---|---|
| plain `var` in composable | no | no | no |
| `remember` | yes | no | no |
| `rememberSaveable` | yes | yes | limited (Bundle) |
| `ViewModel` | yes | **yes** | no (use SavedStateHandle) |

### MVVM
**Model** (data + business logic: repositories, database, network) — **ViewModel** (holds UI state, handles events, talks to the Model) — **View** (composables that render state and send events up). The View never touches the Model directly.

### Hilt — dependency injection
A **dependency** is an object a class needs to do its job (e.g. a ViewModel needs a repository). **Hilt** creates and supplies these automatically. The four core annotations:

```kotlin
@HiltAndroidApp                 // (1) on the Application class — bootstraps Hilt
class MyApp : Application()

@AndroidEntryPoint              // (2) on the Activity — enables injection into its UI
class MainActivity : ComponentActivity() { /* ... */ }

@HiltViewModel                  // (3) on the ViewModel
class HomeViewModel @Inject constructor(   // (4) @Inject tells Hilt how to build it
    private val repo: UserRepository
) : ViewModel()
```

In a composable, obtain it with `val vm: HomeViewModel = hiltViewModel()`. **Modules** tell Hilt how to provide types it can't construct itself: `@Provides` (function body builds it) and `@Binds` (maps an interface to its implementation).

### Exposing state with StateFlow
A plain variable can't notify the UI. **StateFlow** is an observable state holder that always has a current value and emits updates. Use the **private mutable / public read-only** split:

```kotlin
@HiltViewModel
class CounterViewModel @Inject constructor() : ViewModel() {
    private val _uiState = MutableStateFlow(CounterUiState(count = 0))
    val uiState: StateFlow<CounterUiState> = _uiState.asStateFlow()   // read-only outside

    fun increment() = _uiState.update { it.copy(count = it.count + 1) }
}

data class CounterUiState(val count: Int)
```

The composable collects it lifecycle-aware (pauses when the screen isn't visible):

```kotlin
@Composable
fun CounterScreen(vm: CounterViewModel = hiltViewModel()) {
    val state by vm.uiState.collectAsStateWithLifecycle()
    Button(onClick = vm::increment) { Text("Count: ${state.count}") }
}
```

### Modelling UI state with a sealed interface
Instead of juggling booleans (`isLoading`, `hasError`…), represent mutually exclusive states with a **sealed interface**:

```kotlin
sealed interface HomeUiState {
    data object Loading : HomeUiState                 // no data → data object (singleton)
    data class Success(val users: List<User>) : HomeUiState
    data class Error(val message: String) : HomeUiState
}
```

`data object` for states with no payload, `data class` for those carrying data. The UI handles them with an exhaustive `when`:

```kotlin
when (val s = state) {
    HomeUiState.Loading -> CircularProgressIndicator()
    is HomeUiState.Success -> UserList(s.users)
    is HomeUiState.Error -> Text(s.message)
}
```

### Coroutines in the ViewModel
`viewModelScope` is a coroutine scope tied to the ViewModel's lifetime — work launched in it is **cancelled automatically** when the ViewModel is cleared (no leaks):

```kotlin
fun load() = viewModelScope.launch {
    _uiState.value = HomeUiState.Loading
    _uiState.value = HomeUiState.Success(repo.getUsers())
}
```

### One-time events
StateFlow is wrong for one-off events (navigate once, show a snackbar once) because new collectors re-receive the current value. Use a **Channel** (or `SharedFlow`) for events that should fire exactly once:

```kotlin
private val _events = Channel<UiEvent>()
val events = _events.receiveAsFlow()
// later: _events.send(UiEvent.NavigateToDetails(id))
```

**Exam-worthy facts:** ViewModel survives rotation (Activity/composition do not); MVVM keeps the View off the Model; the four Hilt annotations; private `MutableStateFlow` + public `asStateFlow()`; `collectAsStateWithLifecycle`; sealed interface (`data object` vs `data class`) for UI state; `viewModelScope` auto-cancels; Channel/SharedFlow for one-time events.

---

## 9. Coroutines & Asynchronous Programming

### The main-thread problem
The **UI (main) thread** draws the screen ~60 times/second. If you block it with slow work (network, database, big computation), the UI freezes and Android may show "App Not Responding". **Solution:** move slow work off the main thread.

### What a coroutine is
A **coroutine** is a lightweight, suspendable unit of work. Thousands can run on a few OS threads because a suspended coroutine **releases its thread** instead of blocking it. A `suspend` function can pause and resume without blocking:

```kotlin
suspend fun fetchUser(): User { /* may suspend at I/O */ }
```

### Dispatchers — choosing the thread
A **Dispatcher** decides which thread(s) a coroutine runs on:
- `Dispatchers.Main` — UI work.
- `Dispatchers.IO` — network/disk (blocking I/O).
- `Dispatchers.Default` — CPU-heavy work (sorting, parsing).

```kotlin
val data = withContext(Dispatchers.IO) { api.download() }   // switch thread, then return
```

### Structured concurrency and scopes
Coroutines launched in a **scope** are children of it; cancelling the scope cancels them all. This prevents leaks. In Android you rarely create raw scopes — you use `viewModelScope` (ViewModel) or `rememberCoroutineScope()` (UI). The safe pattern is to launch inside a lifecycle-bound scope:

```kotlin
viewModelScope.launch {
    val result = withContext(Dispatchers.IO) { repo.load() }
    _uiState.value = result
}
```

A **SupervisorJob** isolates child failures: one failing child doesn't cancel its siblings.

### Flow — reactive streams
A **Flow** emits a sequence of values over time. It's **cold**: nothing runs until it's collected. Build with `flow { emit(...) }` or `flowOf(...)`, transform with intermediate operators (`map`, `filter`), and collect to consume:

```kotlin
val doubled = flowOf(1, 2, 3).map { it * 10 }.filter { it > 10 }   // cold; nothing yet
viewModelScope.launch { doubled.collect { println(it) } }          // now it runs → 20, 30
```

### StateFlow and SharedFlow
- **StateFlow** is a **hot** flow that always holds one current value — ideal for UI state.
- **SharedFlow** is a hot flow without a "current value" — ideal for one-time events.

To turn a cold flow (e.g. from Room) into UI state, use `stateIn`:

```kotlin
val users: StateFlow<List<User>> = repo.usersFlow()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
```

`WhileSubscribed(5000)` keeps the upstream alive for 5000 ms after the last collector leaves, so quick rotations don't re-trigger the query.

### Collecting Flow in Compose
Use `collectAsStateWithLifecycle()` so collection pauses when the screen is backgrounded.

### Parallel work with async
`launch` fires and forgets; `async` returns a `Deferred<T>` you `await`. Run independent calls **in parallel** instead of sequentially:

```kotlin
val a = async { repo.loadA() }      // both start now
val b = async { repo.loadB() }
val combined = a.await() + b.await() // wait for both
```

### Cancellation
Cancellation is **cooperative**: a coroutine must reach a suspension point (or check `isActive`) to actually stop. In long computation loops, check `isActive` (or call `ensureActive()`) so cancellation works.

**Always:** do slow work off the main thread, use lifecycle scopes, expose state as StateFlow. **Never:** block the main thread, use `GlobalScope`, or swallow `CancellationException`.

**Exam-worthy facts:** coroutines suspend without blocking threads; IO for network/disk, Default for CPU; Flow is cold, StateFlow/SharedFlow are hot; `stateIn` + `WhileSubscribed(5000)`; `async`/`await` for parallelism; cancellation is cooperative.

---

## 10. Data Management Layer (Room)

### Why a local database
In-memory data dies when the app is killed. For structured, queryable, persistent data you need a database. Android ships **SQLite**; **Room** is the recommended object-mapping layer over it — type-safe, compile-time-checked queries, and `Flow` support.

### Room's three components
1. **Entity** — a table (a `data class`).
2. **DAO** — the queries (an interface).
3. **Database** — ties entities + DAOs together.

### Entities
```kotlin
@Entity(tableName = "notes")
data class NoteEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "title") val title: String,
    val body: String,
    @Ignore val transient: String = ""    // not stored
)
```

Each `@Entity` maps to a table; `@PrimaryKey` is the unique row id (often `autoGenerate`). Prefer storing primitive types.

### DAO
```kotlin
@Dao
interface NoteDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(note: NoteEntity): Long

    @Update suspend fun update(note: NoteEntity)
    @Delete suspend fun delete(note: NoteEntity)

    @Query("SELECT * FROM notes ORDER BY id DESC")
    fun getAll(): Flow<List<NoteEntity>>        // observed: re-emits on any change
}
```

- Writes are `suspend` (one-shot, off the main thread). Reads returning `Flow` are **observed** — the query re-emits automatically whenever the table changes (this is what drives live UI).
- **OnConflictStrategy**: `REPLACE` overwrites a row with the same primary key; `IGNORE` keeps the existing row.

### Database class
```kotlin
@Database(entities = [NoteEntity::class], version = 1, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun noteDao(): NoteDao
}
```

Build it with the **application context** (not an Activity context, to avoid leaks). The `version` must be incremented whenever the schema changes, paired with a **migration** (or `fallbackToDestructiveMigration()`, which wipes data).

### Relationships
- **1:1 / 1:N**: a child entity holds a foreign key to the parent; read combined data with an `@Relation` POJO.
- **M:N**: needs a **junction (cross-ref) table** with a composite primary key.
- **Index** foreign-key columns (`@Index`) for query performance, and use `onDelete = CASCADE` so deleting a parent removes its children automatically.

### Migrations
Changing the schema requires bumping `version` and supplying a `Migration` that runs the needed SQL; otherwise the app crashes on upgrade. Export the schema (`exportSchema = true`) to track changes.

### Repository pattern
Don't let the UI/ViewModel call the DAO directly. A **repository** sits between them, mapping database **entities** to clean **domain models** and hiding the data source. This makes it swappable (Room today, network later) and testable.

```kotlin
class NoteRepository @Inject constructor(private val dao: NoteDao) {
    fun notes(): Flow<List<Note>> = dao.getAll().map { list -> list.map { it.toDomain() } }
    suspend fun add(note: Note) = dao.insert(note.toEntity())
}
```

### Wiring with Hilt + the complete flow
```kotlin
@Module @InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides @Singleton
    fun provideDb(@ApplicationContext ctx: Context): AppDatabase =
        Room.databaseBuilder(ctx, AppDatabase::class.java, "app.db").build()

    @Provides fun provideNoteDao(db: AppDatabase): NoteDao = db.noteDao()
}
```

**End-to-end:** UI event → ViewModel (`viewModelScope.launch`) → Repository → DAO → SQLite write. Because the list screen observes a `Flow` from `getAll()`, the write makes the query re-emit → Repository maps entity→domain → ViewModel's StateFlow updates → UI recomposes. The database is the **single source of truth**.

**Exam-worthy facts:** Entity/DAO/Database are Room's three parts; `Flow` return = observed/auto-refresh, `suspend` = one-shot; REPLACE vs IGNORE; bump `version` + migrate on schema change; M:N needs a junction table; the repository maps entity↔domain and is the single source of truth boundary.

---

## 11. APIs & Networking (Retrofit)

### Why network access
A local-only app can't sync, share, or fetch fresh data. Most real apps talk to a **server** over **HTTP** using a **REST** API: resources addressed by URLs, acted on with methods — `GET` (read), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (remove).

### Retrofit
**Retrofit** turns an annotated Kotlin interface into a working HTTP client. You declare *what* the endpoints are; Retrofit generates the networking code.

### DTOs
A **DTO** (Data Transfer Object) models the JSON the server returns. `@SerializedName` maps a JSON field whose name differs from your property:

```kotlin
data class HabitDto(
    val id: Int,
    val name: String,
    @SerializedName("created_at") val createdAt: String
)
```

### The API service interface
```kotlin
interface HabitApiService {
    @GET("habits")
    suspend fun getHabits(): List<HabitDto>

    @GET("habits/{id}")
    suspend fun getHabit(@Path("id") id: Int): HabitDto

    @POST("habits")
    suspend fun create(@Body dto: HabitDto): HabitDto

    @GET("habits")
    suspend fun search(@Query("q") query: String): List<HabitDto>
}
```

`@Path` fills a URL segment, `@Query` adds `?q=...`, `@Body` sends a JSON request body. Endpoints are `suspend` so they run off the main thread.

### OkHttp and interceptors
Retrofit uses **OkHttp** underneath. An **interceptor** can inspect/modify every request — commonly logging:

```kotlin
val logging = HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BODY }
val client = OkHttpClient.Builder().addInterceptor(logging).build()
```

Remove `BODY` logging in production. Two gotchas: the **base URL must end with a slash**, and the emulator reaches your machine's localhost at **`10.0.2.2`** (a physical device needs your LAN IP). Plain `http` (cleartext) is blocked by default — use `https` or explicitly allow it.

### Mapping DTOs to domain models
Keep network shapes out of the rest of the app. Map DTO → domain with an extension function (same idea as Room's entity↔domain):

```kotlin
fun HabitDto.toDomain() = Habit(id = id, name = name, createdAt = createdAt)
```

### Repository + Hilt wiring
The **repository** hides Retrofit behind an interface and returns domain models:

```kotlin
class HabitRepositoryImpl @Inject constructor(
    private val api: HabitApiService
) : HabitRepository {
    override suspend fun getHabits(): List<Habit> = api.getHabits().map { it.toDomain() }
}

@Module @InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")          // trailing slash!
        .client(client)
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    @Provides fun provideApi(retrofit: Retrofit): HabitApiService =
        retrofit.create(HabitApiService::class.java)
}
```

### Error handling
Network calls fail (no connection, timeouts, non-2xx, bad JSON). Wrap them and surface a usable state — but **re-throw `CancellationException`** so coroutine cancellation still works:

```kotlin
fun load() = viewModelScope.launch {
    _uiState.value = UiState.Loading
    try {
        _uiState.value = UiState.Success(repo.getHabits())
    } catch (e: CancellationException) {
        throw e                                  // never swallow cancellation
    } catch (e: Exception) {
        _uiState.value = UiState.Error(e.message ?: "Network error")
    }
}
```

The **Network Inspector** (App Inspection) shows each request/response, headers, and timing for debugging.

**Exam-worthy facts:** Retrofit generates the client from an annotated interface; `@Path`/`@Query`/`@Body`; base URL ends with `/`; emulator localhost = `10.0.2.2`; cleartext blocked by default; map DTO→domain; re-throw `CancellationException`.

---

## 12. Cloud Integration (Firebase)

### What Firebase is
**Firebase** is Google's backend-as-a-service: ready-made authentication, a cloud database, file storage, and more — so you don't run your own server. It sits at the **data/backend layer**, behind your repository, just like Room or Retrofit.

### Authentication
**Firebase Authentication** manages users and sign-in (email/password, Google, etc.). The `FirebaseAuth` object is the entry point:

```kotlin
val auth = FirebaseAuth.getInstance()
auth.signInWithEmailAndPassword(email, password)
    .addOnSuccessListener { /* signed in: auth.currentUser */ }
    .addOnFailureListener { e -> /* show error */ }

val isLoggedIn = auth.currentUser != null
```

**Google Sign-In** is two steps: obtain a Google credential via the system, then exchange it with Firebase. Check the current user with `auth.currentUser`.

### Cloud Firestore
A **NoSQL** document database organised as **collections → documents → fields** (documents can hold sub-collections). You read/write documents by path:

```kotlin
val db = FirebaseFirestore.getInstance()
db.collection("quests").document(questId).get()
    .addOnSuccessListener { doc ->
        val title = doc.getString("title")
        val id = doc.id                          // the document's ID
    }
db.collection("quests").add(mapOf("title" to "Run 5k"))   // auto-ID document
```

**Security Rules** (server-side) decide who can read/write what — never trust the client alone.

### Firebase Storage
For large binary files (images, videos) — Firestore is for structured data, Storage is for files:

```kotlin
val storage = FirebaseStorage.getInstance()
val ref = storage.reference.child("avatars/$uid.jpg")
ref.putFile(localUri)
    .addOnSuccessListener { ref.downloadUrl.addOnSuccessListener { url -> /* show */ } }
```

**Storage vs Firestore:** files/blobs → Storage; structured queryable data → Firestore.

### Hilt module and error handling
Provide the Firebase singletons through a Hilt module so ViewModels can inject them:

```kotlin
@Module @InstallIn(SingletonComponent::class)
object FirebaseModule {
    @Provides @Singleton fun auth() = FirebaseAuth.getInstance()
    @Provides @Singleton fun firestore() = FirebaseFirestore.getInstance()
}
```

Firebase operations are async and report failures via listeners (or, wrapped in coroutines, via `.await()` and try/catch). Map Firebase exceptions to friendly messages in the ViewModel, just like network errors.

**Exam-worthy facts:** Firebase = BaaS at the backend layer; `FirebaseAuth` for sign-in, `auth.currentUser` for login state; Firestore is collections/documents/fields; Storage is for files (vs Firestore for data); Security Rules enforce access server-side; inject Firebase via a Hilt module.

---

## 13. Themes, File Handling & Animations

### Material 3 and the semantic color system
**Material 3** styles apps with **semantic color roles** instead of raw colors. Each role has a paired "on" color for content drawn on top, guaranteeing contrast:

- `surface` / `onSurface` — screen backgrounds and body text.
- `primary` / `onPrimary` — primary buttons, FABs.
- `error` / `onError` — destructive actions, validation errors.
- `primaryContainer` / `onPrimaryContainer` — chips, badges, tags.

A **provider/consumer** architecture: `MaterialTheme` provides the scheme; composables consume it:

```kotlin
Surface(color = MaterialTheme.colorScheme.surface) {
    Text("Hello", color = MaterialTheme.colorScheme.onSurface)
}
Button(/* uses primary/onPrimary by default */) { Text("Save") }
```

Because colors are referenced by role, switching to a dark scheme (or recoloring the app) requires no changes in individual composables — they automatically pick up the active scheme.

### File handling
File I/O on Android needs care because of **scoped storage** (apps can't freely roam the filesystem) and permissions. The three pillars of safe file I/O: use the right **scoped** location (app-specific dirs need no permission), do I/O **off the main thread**, and share files via a **FileProvider** (which hands other apps a safe content URI instead of a raw file path):

```kotlin
// app-specific file (no permission needed), written off the main thread
withContext(Dispatchers.IO) {
    context.filesDir.resolve("notes.txt").writeText(content)
}
```

A **FileProvider** is declared in the Manifest and used to generate a `content://` URI when, e.g., sharing a photo with another app.

### Animations
Motion communicates change and guides attention. Common APIs:

- **`animate*AsState`** — animate a single value when state changes:
  ```kotlin
  val alpha by animateFloatAsState(if (selected) 1f else 0.5f, label = "alpha")
  Box(Modifier.graphicsLayer { this.alpha = alpha })
  ```
- **`AnimatedVisibility`** — animate a composable appearing/disappearing.
- **`AnimatedContent`** — animate transitions between different content (e.g. screen or step changes). It needs a **stable key/target state** so it knows what is changing:
  ```kotlin
  AnimatedContent(targetState = step, label = "step") { current -> StepUi(current) }
  ```

The **layering rule**: prefer animating cheap graphics-layer properties (alpha, scale, offset) over relayout, to keep animations smooth.

**Exam-worthy facts:** Material 3 uses semantic role pairs (e.g. `surface`/`onSurface`); `MaterialTheme.colorScheme` is the provider; scoped storage + off-main-thread + FileProvider for safe file I/O; `animate*AsState` / `AnimatedVisibility` / `AnimatedContent` (stable key required).

---

## 14. Testing, Stability & Advanced Debugging

### Why test, and the testing pyramid
Tests catch regressions cheaply and document behaviour. The **testing pyramid**: many fast **unit tests** at the base, fewer **integration** tests, and a small number of slow **UI/end-to-end** tests on top.

### Where tests live
- `src/test/` — **local unit tests** run on the JVM (fast, no device). For pure logic and ViewModels.
- `src/androidTest/` — **instrumented tests** run on a device/emulator. For Room DAOs, Context-dependent code, and Compose UI.

Typical dependencies: JUnit, a coroutine test dispatcher, a mocking library (MockK), and **Turbine** for testing Flows.

### Testing a ViewModel
Replace real dependencies with fakes/mocks, drive an action, assert the resulting state:

```kotlin
@Test
fun `loading then success`() = runTest {
    val repo = FakeRepo(users = listOf(User(1, "Mira")))
    val vm = HomeViewModel(repo)
    vm.load()
    assertEquals(HomeUiState.Success(listOf(User(1, "Mira"))), vm.uiState.value)
}
```

### Testing Flows with Turbine
Turbine lets you assert each emission of a Flow in order:

```kotlin
vm.uiState.test {
    assertEquals(HomeUiState.Loading, awaitItem())
    assertTrue(awaitItem() is HomeUiState.Success)
    cancelAndIgnoreRemainingEvents()
}
```

### Compose UI tests
`createComposeRule()` sets the content, then you use **finders** (`onNodeWithText`, `onNodeWithTag`), **actions** (`performClick`, `performTextInput`), and **assertions** (`assertIsDisplayed`, `assertTextEquals`):

```kotlin
@get:Rule val rule = createComposeRule()

@Test fun increments() {
    rule.setContent { CounterScreen() }
    rule.onNodeWithText("Count: 0").assertIsDisplayed()
    rule.onNodeWithText("Count: 0").performClick()
    rule.onNodeWithText("Count: 1").assertIsDisplayed()
}
```

This is why stateless composables matter — you can drive them directly with test state.

### Advanced debugging & stability tools
- **Layout Inspector** — inspect the live composable/view tree and properties.
- **Database Inspector** — browse and query your Room database while the app runs.
- **Network Inspector** — view live HTTP traffic.
- **Memory / CPU Profiler** — find leaks, allocations, and hot spots.
- **LeakCanary** — a library that automatically detects memory leaks in debug builds and tells you the retaining reference chain.

### Release builds
A release build is **minified/shrunk** with R8/ProGuard (removes unused code, obfuscates names), reducing size and surface area. Keep ProGuard rules for anything accessed reflectively (e.g. some serialization).

**Exam-worthy facts:** unit tests in `src/test` (JVM), instrumented in `src/androidTest` (device); Turbine asserts Flow emissions; Compose tests = finders + actions + assertions; LeakCanary finds leaks in debug; R8/ProGuard shrinks/obfuscates release builds.

---

## Quick revision map

| Topic | One-line essence |
|---|---|
| Activity lifecycle | Rotation recreates the Activity → state loss → use ViewModel |
| Kotlin | `val`/`var`, null safety (`?./?:/!!`), `.copy()`, lambdas, collections return new lists |
| Compose | UI = f(state); recomposition re-runs changed composables; modifier order matters |
| Components | TextField is controlled; UDF = state down, events up; ContentScale variants |
| State | `remember` + `mutableStateOf`; `rememberSaveable` survives rotation; hoisting |
| Lists | `LazyColumn` composes only visible items; stable keys preserve identity |
| Navigation | NavController + routes (sealed class); args via path + NavType / SavedStateHandle |
| ViewModel | survives rotation; MVVM; Hilt's 4 annotations; private/public StateFlow |
| Coroutines | suspend without blocking; IO/Default; Flow cold, StateFlow hot; `stateIn` |
| Room | Entity/DAO/Database; Flow=observed, suspend=one-shot; repository = single source of truth |
| API | Retrofit interface; `@Path/@Query/@Body`; map DTO→domain; re-throw cancellation |
| Firebase | BaaS; Auth/Firestore/Storage; inject via Hilt module |
| Themes/Files/Anim | Material 3 role pairs; scoped storage + FileProvider; `animate*AsState` |
| Testing | pyramid; `test` vs `androidTest`; Turbine for flows; finders/actions/assertions |
