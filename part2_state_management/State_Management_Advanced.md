# State Management - Advanced
## iOS 17+ Observation Framework & Compose 2025

---

## ÜBERSICHT

Diese Datei behandelt **moderne, fortgeschrittene State Management Patterns** für 2024/2025:

**SwiftUI iOS 17+:**
- ✅ `@Observable` als neuer Standard (kein `@Published` mehr!)
- ✅ `@Environment(Type.self)` statt `@EnvironmentObject`
- ✅ Simplified Property Wrappers
- ✅ Bessere Performance durch granulares Tracking

**Jetpack Compose 2025:**
- ✅ `StateFlow` als Standard (LiveData deprecated)
- ✅ `rememberUpdatedState()` für Callbacks (Compose 1.7+)
- ✅ `SnapshotFlow` für UI Consistency
- ✅ Smart Recomposition Optimizations

---

## 1. SWIFTUI iOS 17+: OBSERVATION FRAMEWORK

### 1.1 Die Revolution: @Observable

#### Was hat sich geändert?

**Legacy (iOS 13-16):**
```swift
import Combine

class UserViewModel: ObservableObject {
    @Published var name: String = ""
    @Published var email: String = ""
    @Published var age: Int = 0
}

struct ContentView: View {
    @StateObject private var viewModel = UserViewModel()
    // oder
    @ObservedObject var viewModel: UserViewModel
}
```

**Modern (iOS 17+):**
```swift
import Observation  // ← Nicht mehr Combine!

@Observable
class UserViewModel {
    var name: String = ""      // ← Kein @Published!
    var email: String = ""
    var age: Int = 0
}

struct ContentView: View {
    @State private var viewModel = UserViewModel()  // ← Nur @State!
    // oder
    var viewModel: UserViewModel  // ← Kein Wrapper bei Übergabe!
}
```

**Vorteile:**
- ✅ **60% weniger Code**
- ✅ **Keine @Published Annotations**
- ✅ **Bessere Performance** (bis zu 50-70% weniger Redraws)
- ✅ **Computed Properties funktionieren automatisch**
- ✅ **Einfacheres Mental Model**

---

### 1.2 Komplexe ViewModels mit @Observable

#### Shopping Cart Beispiel

```swift
@Observable
class ShoppingCartViewModel {
    var items: [CartItem] = []
    var isLoading = false
    var errorMessage: String?
    
    // ✅ Computed Properties funktionieren automatisch!
    var itemCount: Int {
        items.reduce(0) { $0 + $1.quantity }
    }
    
    var subtotal: Double {
        items.reduce(0) { $0 + ($1.product.price * Double($1.quantity)) }
    }
    
    var tax: Double {
        subtotal * 0.19
    }
    
    var total: Double {
        subtotal + tax
    }
    
    var hasItems: Bool {
        !items.isEmpty
    }
    
    func addItem(_ product: Product) {
        if let index = items.firstIndex(where: { $0.product.id == product.id }) {
            items[index].quantity += 1
        } else {
            items.append(CartItem(product: product, quantity: 1))
        }
    }
    
    func updateQuantity(_ item: CartItem, quantity: Int) {
        guard let index = items.firstIndex(where: { $0.id == item.id }) else { return }
        if quantity <= 0 {
            items.remove(at: index)
        } else {
            items[index].quantity = quantity
        }
    }
    
    func removeItem(_ item: CartItem) {
        items.removeAll { $0.id == item.id }
    }
    
    func checkout() async {
        isLoading = true
        errorMessage = nil
        
        do {
            try await performCheckout()
            items.removeAll()
        } catch {
            errorMessage = error.localizedDescription
        }
        
        isLoading = false
    }
}
```

#### Verwendung in Views

```swift
struct CartView: View {
    @State private var cartViewModel = ShoppingCartViewModel()
    
    var body: some View {
        NavigationStack {
            Group {
                if cartViewModel.hasItems {
                    cartContent
                } else {
                    emptyCartView
                }
            }
            .navigationTitle("Shopping Cart (\(cartViewModel.itemCount))")
            .toolbar {
                if cartViewModel.hasItems {
                    Button("Checkout") {
                        Task { await cartViewModel.checkout() }
                    }
                    .disabled(cartViewModel.isLoading)
                }
            }
        }
    }
    
    private var cartContent: some View {
        VStack {
            List {
                ForEach(cartViewModel.items) { item in
                    CartItemRow(
                        item: item,
                        onQuantityChange: { quantity in
                            cartViewModel.updateQuantity(item, quantity: quantity)
                        },
                        onRemove: {
                            cartViewModel.removeItem(item)
                        }
                    )
                }
            }
            
            // Summary
            VStack(alignment: .trailing, spacing: 8) {
                HStack {
                    Text("Subtotal:")
                    Spacer()
                    Text("$\(cartViewModel.subtotal, specifier: "%.2f")")
                }
                HStack {
                    Text("Tax (19%):")
                    Spacer()
                    Text("$\(cartViewModel.tax, specifier: "%.2f")")
                }
                Divider()
                HStack {
                    Text("Total:")
                        .bold()
                    Spacer()
                    Text("$\(cartViewModel.total, specifier: "%.2f")")
                        .bold()
                }
            }
            .padding()
        }
    }
    
    private var emptyCartView: some View {
        ContentUnavailableView(
            "Cart is Empty",
            systemImage: "cart",
            description: Text("Add items to get started")
        )
    }
}
```

---

### 1.3 Modern Environment Pattern

#### Type-Safe Environment (iOS 17+)

**Legacy:**
```swift
@EnvironmentObject var settings: AppSettings
.environmentObject(settings)
```

**Modern:**
```swift
@Environment(AppSettings.self) private var settings
.environment(settings)
```

#### App-weite Settings

```swift
@Observable
class AppSettings {
    var isDarkMode = false
    var fontSize: Double = 16
    var language = "en"
    var notificationsEnabled = true
    
    // Computed
    var fontStyle: Font {
        .system(size: fontSize)
    }
    
    func save() {
        UserDefaults.standard.set(isDarkMode, forKey: "isDarkMode")
        UserDefaults.standard.set(fontSize, forKey: "fontSize")
        UserDefaults.standard.set(language, forKey: "language")
    }
    
    func load() {
        isDarkMode = UserDefaults.standard.bool(forKey: "isDarkMode")
        fontSize = UserDefaults.standard.double(forKey: "fontSize")
        language = UserDefaults.standard.string(forKey: "language") ?? "en"
    }
}

@main
struct MyApp: App {
    @State private var settings = AppSettings()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(settings)
                .preferredColorScheme(settings.isDarkMode ? .dark : .light)
                .onAppear {
                    settings.load()
                }
        }
    }
}

// Verwendung in jeder View
struct SettingsView: View {
    @Environment(AppSettings.self) private var settings
    
    var body: some View {
        Form {
            Toggle("Dark Mode", isOn: $settings.isDarkMode)
            
            VStack(alignment: .leading) {
                Text("Font Size: \(Int(settings.fontSize))")
                Slider(value: $settings.fontSize, in: 12...24)
            }
            
            Picker("Language", selection: $settings.language) {
                Text("English").tag("en")
                Text("Deutsch").tag("de")
            }
            
            Button("Save Settings") {
                settings.save()
            }
        }
    }
}
```

---

### 1.4 Nested Observable Objects

#### Das Problem (Legacy)

```swift
// ❌ Legacy: Nested ObservableObjects funktionieren nicht gut
class User: ObservableObject {
    @Published var name: String = ""
}

class AppState: ObservableObject {
    @Published var currentUser: User?
    // Problem: Änderungen in User triggern kein Update in AppState!
}
```

#### Die Lösung (Modern)

```swift
// ✅ Modern: @Observable funktioniert automatisch, egal wie tief!
@Observable
class User {
    var name: String = ""
    var email: String = ""
}

@Observable
class Profile {
    var bio: String = ""
    var avatarURL: String = ""
}

@Observable
class AppState {
    var currentUser: User?
    var profile: Profile?
    var isAuthenticated = false
    
    // Computed Properties über verschachtelte Objects
    var displayName: String {
        currentUser?.name ?? "Guest"
    }
}

// Verwendung
struct ProfileView: View {
    @State private var appState = AppState()
    
    var body: some View {
        VStack {
            // ✅ Updates automatisch bei Änderungen in User ODER Profile!
            Text("Welcome, \(appState.displayName)")
            
            if let user = appState.currentUser {
                TextField("Name", text: $user.name)
            }
            
            if let profile = appState.profile {
                TextField("Bio", text: $profile.bio)
            }
        }
    }
}
```

---

### 1.5 Performance: Granular Dependency Tracking

#### Der Unterschied zu Legacy

**Legacy (iOS 13-16):**
```swift
class ViewModel: ObservableObject {
    @Published var count = 0
    @Published var name = ""
}

struct MyView: View {
    @ObservedObject var viewModel: ViewModel
    
    var body: some View {
        // ⚠️ View wird bei JEDER Änderung neu gerendert
        // (count ODER name)
        Text("\(viewModel.count)")
    }
}
```

**Modern (iOS 17+):**
```swift
@Observable
class ViewModel {
    var count = 0
    var name = ""
}

struct MyView: View {
    var viewModel: ViewModel
    
    var body: some View {
        // ✅ View wird NUR bei count-Änderungen gerendert
        // name-Änderungen werden ignoriert!
        Text("\(viewModel.count)")
    }
}
```

**Performance-Gewinn:** Bis zu **50-70% weniger Redraws** in komplexen Apps!

---

## 2. JETPACK COMPOSE 2025: MODERNE PATTERNS

### 2.1 StateFlow als Standard (nicht mehr LiveData!)

#### Migration von LiveData zu StateFlow

**❌ Alt: LiveData (Legacy):**
```kotlin
class UserViewModel : ViewModel() {
    private val _username = MutableLiveData("")
    val username: LiveData<String> = _username
    
    fun updateUsername(name: String) {
        _username.value = name
    }
}

@Composable
fun ProfileScreen(viewModel: UserViewModel = viewModel()) {
    val username by viewModel.username.observeAsState("")
    TextField(value = username, onValueChange = { viewModel.updateUsername(it) })
}
```

**✅ Modern: StateFlow (2025 Standard):**
```kotlin
class UserViewModel : ViewModel() {
    private val _username = MutableStateFlow("")
    val username: StateFlow<String> = _username.asStateFlow()
    
    fun updateUsername(name: String) {
        _username.value = name
    }
}

@Composable
fun ProfileScreen(viewModel: UserViewModel = viewModel()) {
    val username by viewModel.username.collectAsState()
    TextField(value = username, onValueChange = { viewModel.updateUsername(it) })
}
```

**Warum StateFlow?**
- ✅ Bessere Kotlin Coroutines Integration
- ✅ Thread-safe by default
- ✅ Hot Flow (immer aktiv)
- ✅ Moderneres API
- ✅ Bessere Performance

---

### 2.2 rememberUpdatedState() - Compose 1.7+

#### Das Problem: Stale Callbacks

```kotlin
@Composable
fun CountdownTimer(
    seconds: Int,
    onFinish: () -> Unit  // ← Kann "stale" werden!
) {
    LaunchedEffect(seconds) {
        delay(seconds * 1000L)
        onFinish()  // ⚠️ Verwendet möglicherweise alte Version von onFinish!
    }
}
```

#### Die Lösung: rememberUpdatedState()

```kotlin
@Composable
fun CountdownTimer(
    seconds: Int,
    onFinish: () -> Unit
) {
    // ✅ Callback ist immer aktuell
    val currentOnFinish by rememberUpdatedState(onFinish)
    
    LaunchedEffect(seconds) {
        delay(seconds * 1000L)
        currentOnFinish()  // ✅ Verwendet immer die neueste Version!
    }
}
```

#### Practical Example: Event Tracking

```kotlin
@Composable
fun TrackedButton(
    text: String,
    onClick: () -> Unit,
    analyticsEvent: String
) {
    // ✅ Analytics event ist immer aktuell
    val currentAnalyticsEvent by rememberUpdatedState(analyticsEvent)
    
    Button(
        onClick = {
            Analytics.logEvent(currentAnalyticsEvent)
            onClick()
        }
    ) {
        Text(text)
    }
}
```

---

### 2.3 SnapshotFlow - UI Consistency

#### Was ist SnapshotFlow?

`SnapshotFlow` konvertiert Compose State in einen Flow, der bei Änderungen emittiert.

**Use Case:** State-Synchronisation, Side Effects, Observing

#### Beispiel: Scroll-basierte Logic

```kotlin
@Composable
fun ScrollableList() {
    val listState = rememberLazyListState()
    val coroutineScope = rememberCoroutineScope()
    
    // ✅ SnapshotFlow für reactive Scroll-Tracking
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                // Reagiere auf Scroll-Änderungen
                if (index > 10) {
                    // Show "Scroll to Top" button
                }
            }
    }
    
    LazyColumn(state = listState) {
        items(100) { index ->
            Text("Item $index")
        }
    }
}
```

#### Beispiel: Form Validation

```kotlin
@Composable
fun RegistrationForm() {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var isValid by remember { mutableStateOf(false) }
    
    // ✅ Automatic validation mit SnapshotFlow
    LaunchedEffect(Unit) {
        snapshotFlow { email to password }
            .map { (e, p) ->
                e.contains("@") && p.length >= 8
            }
            .collect { valid ->
                isValid = valid
            }
    }
    
    Column {
        TextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("Email") }
        )
        
        TextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation()
        )
        
        Button(
            onClick = { /* Register */ },
            enabled = isValid
        ) {
            Text("Register")
        }
    }
}
```

---

### 2.4 Advanced StateFlow Patterns

#### Combining Multiple StateFlows

```kotlin
class DashboardViewModel : ViewModel() {
    private val _userProfile = MutableStateFlow<UserProfile?>(null)
    private val _notifications = MutableStateFlow<List<Notification>>(emptyList())
    private val _settings = MutableStateFlow(AppSettings())
    
    // ✅ Combine multiple flows
    val dashboardState: StateFlow<DashboardState> = combine(
        _userProfile,
        _notifications,
        _settings
    ) { profile, notifications, settings ->
        DashboardState(
            profile = profile,
            unreadCount = notifications.count { !it.isRead },
            settings = settings
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = DashboardState()
    )
}

data class DashboardState(
    val profile: UserProfile? = null,
    val unreadCount: Int = 0,
    val settings: AppSettings = AppSettings()
)
```

#### StateFlow with debounce (Search)

```kotlin
class SearchViewModel : ViewModel() {
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    private val _results = MutableStateFlow<List<SearchResult>>(emptyList())
    val results: StateFlow<List<SearchResult>> = _results.asStateFlow()
    
    private val _isSearching = MutableStateFlow(false)
    val isSearching: StateFlow<Boolean> = _isSearching.asStateFlow()
    
    init {
        viewModelScope.launch {
            _searchQuery
                .debounce(300)  // ✅ Wait 300ms after user stops typing
                .filter { it.isNotEmpty() }
                .distinctUntilChanged()
                .collect { query ->
                    performSearch(query)
                }
        }
    }
    
    fun updateQuery(query: String) {
        _searchQuery.value = query
    }
    
    private suspend fun performSearch(query: String) {
        _isSearching.value = true
        try {
            val results = searchRepository.search(query)
            _results.value = results
        } catch (e: Exception) {
            // Handle error
        } finally {
            _isSearching.value = false
        }
    }
}
```

---

### 2.5 rememberSaveable mit Custom Saver

#### Einfache Typen

```kotlin
@Composable
fun SimpleCounter() {
    // ✅ Überlebt Configuration Changes
    var count by rememberSaveable { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

#### Custom Objects mit Saver

```kotlin
data class FormData(
    val name: String = "",
    val email: String = "",
    val age: Int = 0
)

val FormDataSaver = Saver<FormData, String>(
    save = { "${it.name}|${it.email}|${it.age}" },
    restore = {
        val parts = it.split("|")
        FormData(
            name = parts[0],
            email = parts[1],
            age = parts[2].toIntOrNull() ?: 0
        )
    }
)

@Composable
fun RegistrationForm() {
    var formData by rememberSaveable(stateSaver = FormDataSaver) {
        mutableStateOf(FormData())
    }
    
    Column {
        TextField(
            value = formData.name,
            onValueChange = { formData = formData.copy(name = it) },
            label = { Text("Name") }
        )
        
        TextField(
            value = formData.email,
            onValueChange = { formData = formData.copy(email = it) },
            label = { Text("Email") }
        )
        
        TextField(
            value = formData.age.toString(),
            onValueChange = { 
                formData = formData.copy(age = it.toIntOrNull() ?: 0)
            },
            label = { Text("Age") }
        )
    }
}
```

---

## 3. ARCHITECTURE PATTERNS (Modern 2025)

### 3.1 Alternative zu MVVM: Container Pattern (SwiftUI)

**MVVM ist weiterhin gültig**, aber für fortgeschrittene Apps gibt es moderne Alternativen:

#### Container Pattern

```swift
@Observable
class AppContainer {
    let userService: UserService
    let authService: AuthService
    let apiClient: APIClient
    
    @ObservationIgnored
    private let dependencies: Dependencies
    
    init(dependencies: Dependencies = .live) {
        self.dependencies = dependencies
        self.userService = dependencies.userService
        self.authService = dependencies.authService
        self.apiClient = dependencies.apiClient
    }
}

struct Dependencies {
    let userService: UserService
    let authService: AuthService
    let apiClient: APIClient
    
    static let live = Dependencies(
        userService: LiveUserService(),
        authService: LiveAuthService(),
        apiClient: LiveAPIClient()
    )
    
    static let mock = Dependencies(
        userService: MockUserService(),
        authService: MockAuthService(),
        apiClient: MockAPIClient()
    )
}

@main
struct MyApp: App {
    @State private var container = AppContainer()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(container)
        }
    }
}
```

---

### 3.2 The Composable Architecture (TCA) - Hinweis

Für sehr große SwiftUI Apps kann **The Composable Architecture** eine Alternative sein:

```swift
// Nur als Hinweis - nicht im Scope dieses Kompendiums
struct AppFeature: Reducer {
    struct State {
        var count = 0
    }
    
    enum Action {
        case increment
        case decrement
    }
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .increment:
            state.count += 1
            return .none
        case .decrement:
            state.count -= 1
            return .none
        }
    }
}
```

**Hinweis:** TCA ist optional und für Fortgeschrittene. MVVM bleibt der Standard!

---

## 4. PERFORMANCE BEST PRACTICES 2025

### 4.1 SwiftUI Optimizations

#### Task Modifier für Async Operations

```swift
struct DataView: View {
    @State private var viewModel = DataViewModel()
    
    var body: some View {
        List(viewModel.items) { item in
            ItemRow(item: item)
        }
        .task {
            // ✅ Automatisch cancelled bei View-Disappear
            await viewModel.loadData()
        }
    }
}
```

#### @Observable mit Conditional Access

```swift
@Observable
class ComplexViewModel {
    var data: [Item] = []
    var metadata: Metadata?
    var stats: Statistics?
}

struct OptimizedView: View {
    var viewModel: ComplexViewModel
    
    var body: some View {
        // ✅ Nur relevante Properties beobachten
        List(viewModel.data) { item in
            ItemRow(item: item)
        }
        // metadata und stats werden NICHT beobachtet
    }
}
```

---

### 4.2 Compose Optimizations

#### Smart Recomposition mit remember

```kotlin
@Composable
fun OptimizedList(items: List<Item>) {
    // ✅ Teure Berechnung nur einmal
    val sortedItems = remember(items) {
        items.sortedBy { it.priority }
    }
    
    LazyColumn {
        items(sortedItems, key = { it.id }) { item ->
            ItemRow(item)
        }
    }
}
```

#### Stable Types für bessere Performance

```kotlin
// ✅ Stable types werden nicht recomposed wenn sich Inhalt nicht ändert
@Stable
data class UserProfile(
    val id: String,
    val name: String,
    val email: String
)

@Composable
fun ProfileCard(profile: UserProfile) {
    // Nur Recompose wenn profile sich ändert
}
```

---

## 5. MIGRATION GUIDE: LEGACY → MODERN

### 5.1 SwiftUI: iOS 16 → iOS 17+

**Schritte:**

1. **Import ändern:**
```swift
// Alt:
import Combine

// Neu:
import Observation
```

2. **Klasse konvertieren:**
```swift
// Alt:
class MyViewModel: ObservableObject {
    @Published var value = 0
}

// Neu:
@Observable
class MyViewModel {
    var value = 0
}
```

3. **Views aktualisieren:**
```swift
// Alt:
@StateObject private var viewModel = MyViewModel()
@ObservedObject var viewModel: MyViewModel
@EnvironmentObject var settings: AppSettings

// Neu:
@State private var viewModel = MyViewModel()
var viewModel: MyViewModel
@Environment(AppSettings.self) private var settings
```

4. **Environment aktualisieren:**
```swift
// Alt:
.environmentObject(settings)

// Neu:
.environment(settings)
```

---

### 5.2 Compose: LiveData → StateFlow

**Schritte:**

1. **ViewModel ändern:**
```kotlin
// Alt:
private val _data = MutableLiveData<String>()
val data: LiveData<String> = _data

// Neu:
private val _data = MutableStateFlow("")
val data: StateFlow<String> = _data.asStateFlow()
```

2. **Composable aktualisieren:**
```kotlin
// Alt:
val data by viewModel.data.observeAsState("")

// Neu:
val data by viewModel.data.collectAsState()
```

3. **Flows verwenden:**
```kotlin
// Neu: Komplexe Transformationen
val filteredData = data
    .map { it.filter { item -> item.isActive } }
    .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())
```

---

## 6. ZUSAMMENFASSUNG

### Modern Best Practices 2025

| Framework | Legacy | Modern 2025 |
|-----------|--------|-------------|
| **SwiftUI Observable** | `ObservableObject + @Published` | `@Observable` |
| **SwiftUI View** | `@StateObject/@ObservedObject` | `@State / var` |
| **SwiftUI Environment** | `@EnvironmentObject` | `@Environment(Type.self)` |
| **Compose State** | `LiveData` | `StateFlow` |
| **Compose Callbacks** | Direct Lambda | `rememberUpdatedState()` |
| **Compose Observation** | Manual | `SnapshotFlow` |

### Key Takeaways

✅ **iOS 17+**: `@Observable` ist der neue Standard - nutze ihn!
✅ **Compose 2025**: `StateFlow` > `LiveData` - migriere bestehenden Code
✅ **Performance**: Moderne Frameworks sind 50-70% effizienter
✅ **Simplicity**: Weniger Boilerplate, mehr Produktivität
✅ **Future-Proof**: Gebaut für moderne, reactive Apps

---

