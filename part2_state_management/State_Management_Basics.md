# State Management - Basics
## SwiftUI & Jetpack Compose Grundlagen

---

## ÜBERSICHT

Diese Datei behandelt die **fundamentalen State Management Konzepte** für beide Frameworks:

| Konzept | SwiftUI | Jetpack Compose |
|---------|---------|-----------------|
| Lokaler State | `@State` | `remember { mutableStateOf() }` |
| Two-Way Binding | `@Binding` | State Parameter + Lambda |
| Observable Classes | `@Observable` (iOS 17+) | `StateFlow` in ViewModel |
| Shared State | `@Environment` | `CompositionLocal` |
| Persistence | `@AppStorage` | `DataStore` / `rememberSaveable` |

---

## 1. @STATE - LOKALER ZUSTAND

### 1.1 SwiftUI: @State

#### Was ist @State?

`@State` ist ein Property Wrapper für **lokalen, privaten Zustand** innerhalb einer View.

**Wichtige Eigenschaften:**
- ✅ Gehört der View
- ✅ Immer `private`
- ✅ Für Value Types (Int, String, Bool, Struct)
- ✅ Triggert automatisch View-Update

#### Grundlagen

```swift
struct CounterView: View {
    @State private var count = 0  // ← Lokaler State
    
    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") {
                count += 1  // ← Triggert View-Update
            }
        }
    }
}
```

#### Mehrere States verwalten

```swift
struct FormView: View {
    @State private var username = ""
    @State private var password = ""
    @State private var rememberMe = false
    @State private var isLoading = false
    
    var body: some View {
        Form {
            TextField("Username", text: $username)
            SecureField("Password", text: $password)
            Toggle("Remember Me", isOn: $rememberMe)
            
            Button("Login") {
                isLoading = true
                performLogin()
            }
            .disabled(username.isEmpty || isLoading)
            
            if isLoading {
                ProgressView()
            }
        }
    }
}
```

#### Computed Properties mit State

```swift
struct PriceCalculator: View {
    @State private var quantity = 1
    @State private var pricePerItem = 9.99
    
    // ✅ Computed Property - aktualisiert automatisch
    var totalPrice: Double {
        Double(quantity) * pricePerItem
    }
    
    var formattedTotal: String {
        String(format: "$%.2f", totalPrice)
    }
    
    var body: some View {
        VStack {
            Stepper("Quantity: \(quantity)", value: $quantity, in: 1...100)
            Text("Total: \(formattedTotal)")
                .font(.title)
                .bold()
        }
    }
}
```

#### Wann NICHT @State verwenden

```swift
// ❌ FALSCH: Nicht private
@State var count = 0

// ❌ FALSCH: Für komplexe Logik
@State private var userData = UserData()  // Besser: ViewModel

// ❌ FALSCH: Für geteilte Daten
@State private var appSettings = Settings()  // Besser: @Environment

// ✅ RICHTIG: Lokaler UI-State
@State private var isExpanded = false
@State private var selectedTab = 0
```

---

### 1.2 Jetpack Compose: remember & mutableStateOf

#### Was ist remember?

`remember` speichert einen Wert über Recompositions hinweg.

**Wichtige Eigenschaften:**
- ✅ Überlebt Recomposition
- ✅ Verloren bei Configuration Change
- ✅ Kombiniert mit `mutableStateOf` für reaktiven State
- ✅ Kann mehrere Values cachen

#### Grundlagen

```kotlin
@Composable
fun CounterView() {
    var count by remember { mutableStateOf(0) }
    
    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

#### Alternative Syntax

```kotlin
@Composable
fun CounterAlternative() {
    // Ohne Delegation
    val count = remember { mutableStateOf(0) }
    
    Column {
        Text("Count: ${count.value}")  // .value notwendig
        Button(onClick = { count.value++ }) {
            Text("Increment")
        }
    }
}
```

#### rememberSaveable für Persistenz

```kotlin
@Composable
fun PersistentCounter() {
    // ✅ Überlebt Configuration Changes (Rotation, etc.)
    var count by rememberSaveable { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

#### Custom Saver für komplexe Objekte

```kotlin
data class User(val name: String, val age: Int)

@Composable
fun UserForm() {
    var user by rememberSaveable(
        stateSaver = Saver(
            save = { "${it.name},${it.age}" },
            restore = { 
                val parts = it.split(",")
                User(parts[0], parts[1].toInt())
            }
        )
    ) {
        mutableStateOf(User("", 0))
    }
    
    Column {
        TextField(
            value = user.name,
            onValueChange = { user = user.copy(name = it) }
        )
        // ...
    }
}
```

#### Mehrere States

```kotlin
@Composable
fun FormScreen() {
    var username by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var rememberMe by remember { mutableStateOf(false) }
    var isLoading by remember { mutableStateOf(false) }
    
    Column {
        TextField(
            value = username,
            onValueChange = { username = it },
            label = { Text("Username") }
        )
        
        TextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation()
        )
        
        Row {
            Checkbox(
                checked = rememberMe,
                onCheckedChange = { rememberMe = it }
            )
            Text("Remember Me")
        }
        
        Button(
            onClick = { isLoading = true },
            enabled = username.isNotEmpty() && !isLoading
        ) {
            Text("Login")
        }
        
        if (isLoading) {
            CircularProgressIndicator()
        }
    }
}
```

---

## 2. @BINDING - TWO-WAY DATA FLOW

### 2.1 SwiftUI: @Binding

#### Was ist @Binding?

`@Binding` erstellt eine **bidirektionale Referenz** zu einem `@State` einer anderen View.

**Konzept:**
```
Parent (@State) ←→ Child (@Binding)
```

#### Grundlagen

```swift
// Parent View - besitzt die Daten
struct ParentView: View {
    @State private var isOn = false
    
    var body: some View {
        VStack {
            Text("Switch is: \(isOn ? "ON" : "OFF")")
            // $ erstellt Binding
            ChildToggle(isOn: $isOn)
        }
    }
}

// Child View - arbeitet mit den Daten
struct ChildToggle: View {
    @Binding var isOn: Bool
    
    var body: some View {
        Toggle("Toggle", isOn: $isOn)
    }
}
```

#### Custom Controls mit Binding

```swift
struct RatingControl: View {
    @Binding var rating: Int
    let maxRating: Int = 5
    
    var body: some View {
        HStack {
            ForEach(1...maxRating, id: \.self) { star in
                Image(systemName: star <= rating ? "star.fill" : "star")
                    .foregroundColor(.yellow)
                    .onTapGesture {
                        rating = star
                    }
            }
        }
    }
}

// Verwendung
struct ReviewView: View {
    @State private var userRating = 0
    
    var body: some View {
        VStack {
            RatingControl(rating: $userRating)
            Text("Rating: \(userRating) stars")
        }
    }
}
```

#### Constant Bindings für Previews

```swift
struct TextFieldView: View {
    @Binding var text: String
    
    var body: some View {
        TextField("Enter text", text: $text)
    }
}

#Preview {
    TextFieldView(text: .constant("Preview Text"))
}
```

#### Binding Transformations

```swift
struct TemperatureView: View {
    @State private var celsius: Double = 0
    
    // Computed Binding
    var fahrenheitBinding: Binding<Double> {
        Binding(
            get: { celsius * 9/5 + 32 },
            set: { celsius = ($0 - 32) * 5/9 }
        )
    }
    
    var body: some View {
        VStack {
            TextField("Celsius", value: $celsius, format: .number)
            TextField("Fahrenheit", value: fahrenheitBinding, format: .number)
        }
    }
}
```

---

### 2.2 Jetpack Compose: State Parameter + Callback

#### Compose hat kein direktes @Binding

Compose verwendet **State Parameter + Lambda Callbacks** für bidirektionale Kommunikation.

**Pattern:**
```
Parent (State) → Child (Value + Callback) → Parent (Update State)
```

#### Grundlagen

```kotlin
// Parent Composable - besitzt State
@Composable
fun ParentScreen() {
    var isOn by remember { mutableStateOf(false) }
    
    Column {
        Text("Switch is: ${if (isOn) "ON" else "OFF"}")
        
        // Übergibt Value + Callback
        ChildToggle(
            isOn = isOn,
            onToggleChange = { isOn = it }
        )
    }
}

// Child Composable - empfängt Value + Callback
@Composable
fun ChildToggle(
    isOn: Boolean,
    onToggleChange: (Boolean) -> Unit
) {
    Switch(
        checked = isOn,
        onCheckedChange = onToggleChange
    )
}
```

#### Custom Control mit State Parameter

```kotlin
@Composable
fun RatingControl(
    rating: Int,
    onRatingChange: (Int) -> Unit,
    maxRating: Int = 5
) {
    Row {
        repeat(maxRating) { index ->
            Icon(
                imageVector = if (index < rating) {
                    Icons.Filled.Star
                } else {
                    Icons.Outlined.Star
                },
                contentDescription = null,
                tint = Color.Yellow,
                modifier = Modifier.clickable {
                    onRatingChange(index + 1)
                }
            )
        }
    }
}

// Verwendung
@Composable
fun ReviewScreen() {
    var userRating by remember { mutableStateOf(0) }
    
    Column {
        RatingControl(
            rating = userRating,
            onRatingChange = { userRating = it }
        )
        Text("Rating: $userRating stars")
    }
}
```

#### Hoisted State Pattern

```kotlin
// ✅ Best Practice: State Hoisting
@Composable
fun StatefulCounter() {
    // State lebt hier
    var count by remember { mutableStateOf(0) }
    
    // Wird an Stateless Component übergeben
    StatelessCounter(
        count = count,
        onIncrement = { count++ },
        onDecrement = { count-- }
    )
}

@Composable
fun StatelessCounter(
    count: Int,
    onIncrement: () -> Unit,
    onDecrement: () -> Unit
) {
    Row {
        Button(onClick = onDecrement) { Text("-") }
        Text("$count", modifier = Modifier.padding(horizontal = 16.dp))
        Button(onClick = onIncrement) { Text("+") }
    }
}
```

---

## 3. OBSERVABLE / STATEFLOW

### 3.1 SwiftUI: @Observable (iOS 17+)

#### Das moderne Observation Framework

**Ab iOS 17 ist `@Observable` der Standard!** Kein `@Published` mehr notwendig.

```swift
import Observation  // ← Wichtig!

@Observable
class UserViewModel {
    var username: String = ""
    var email: String = ""
    var isLoggedIn: Bool = false
    
    // ✅ Alle Properties sind automatisch observable!
    // ✅ Keine @Published Annotation notwendig!
    
    func login() {
        // Login logic
        isLoggedIn = true
    }
}
```

#### Verwendung in Views

```swift
struct ProfileView: View {
    @State private var viewModel = UserViewModel()
    // ← Einfach @State, kein @StateObject mehr!
    
    var body: some View {
        Form {
            TextField("Username", text: $viewModel.username)
            TextField("Email", text: $viewModel.email)
            
            Button("Login") {
                viewModel.login()
            }
            
            if viewModel.isLoggedIn {
                Text("Welcome, \(viewModel.username)!")
            }
        }
    }
}
```

#### Child Views - Kein Property Wrapper!

```swift
// Parent
struct ParentView: View {
    @State private var viewModel = UserViewModel()
    
    var body: some View {
        ChildView(viewModel: viewModel)
    }
}

// Child
struct ChildView: View {
    var viewModel: UserViewModel  // ← Kein @ObservedObject!
    
    var body: some View {
        Text(viewModel.username)
    }
}
```

#### Computed Properties funktionieren automatisch!

```swift
@Observable
class ShoppingCart {
    var items: [CartItem] = []
    
    // ✅ Computed Properties funktionieren out-of-the-box!
    var itemCount: Int {
        items.reduce(0) { $0 + $1.quantity }
    }
    
    var total: Double {
        items.reduce(0) { $0 + ($1.price * Double($1.quantity)) }
    }
}
```

---

### 3.2 Jetpack Compose: StateFlow in ViewModel

#### Was ist StateFlow?

`StateFlow` ist der **moderne Standard** für reactive State in ViewModels (ersetzt LiveData).

**Eigenschaften:**
- ✅ Hot Flow (immer aktiv)
- ✅ Emittiert immer den aktuellen Value
- ✅ Thread-safe
- ✅ Lifecycle-aware in Compose

#### Grundlagen

```kotlin
class UserViewModel : ViewModel() {
    // Private mutable state
    private val _username = MutableStateFlow("")
    val username: StateFlow<String> = _username.asStateFlow()
    
    private val _isLoggedIn = MutableStateFlow(false)
    val isLoggedIn: StateFlow<Boolean> = _isLoggedIn.asStateFlow()
    
    fun updateUsername(name: String) {
        _username.value = name
    }
    
    fun login() {
        viewModelScope.launch {
            // Login logic
            _isLoggedIn.value = true
        }
    }
}
```

#### Verwendung in Composables

```kotlin
@Composable
fun ProfileScreen(viewModel: UserViewModel = viewModel()) {
    val username by viewModel.username.collectAsState()
    val isLoggedIn by viewModel.isLoggedIn.collectAsState()
    
    Column {
        TextField(
            value = username,
            onValueChange = { viewModel.updateUsername(it) },
            label = { Text("Username") }
        )
        
        Button(onClick = { viewModel.login() }) {
            Text("Login")
        }
        
        if (isLoggedIn) {
            Text("Welcome, $username!")
        }
    }
}
```

#### Derived State mit combine

```kotlin
class ShoppingCartViewModel : ViewModel() {
    private val _items = MutableStateFlow<List<CartItem>>(emptyList())
    val items: StateFlow<List<CartItem>> = _items.asStateFlow()
    
    // ✅ Computed State mit combine
    val itemCount: StateFlow<Int> = items.map { items ->
        items.sumOf { it.quantity }
    }.stateIn(viewModelScope, SharingStarted.Lazily, 0)
    
    val total: StateFlow<Double> = items.map { items ->
        items.sumOf { it.price * it.quantity }
    }.stateIn(viewModelScope, SharingStarted.Lazily, 0.0)
}
```

#### LiveData vs. StateFlow (2025 Standard)

**LiveData (Legacy - nicht mehr empfohlen):**
```kotlin
// ❌ Alt: LiveData
class OldViewModel : ViewModel() {
    private val _count = MutableLiveData(0)
    val count: LiveData<Int> = _count
    
    fun increment() {
        _count.value = (_count.value ?: 0) + 1
    }
}

@Composable
fun OldScreen(viewModel: OldViewModel = viewModel()) {
    val count by viewModel.count.observeAsState(0)
    Text("Count: $count")
}
```

**StateFlow (Modern - empfohlen):**
```kotlin
// ✅ Modern: StateFlow
class ModernViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()
    
    fun increment() {
        _count.value++
    }
}

@Composable
fun ModernScreen(viewModel: ModernViewModel = viewModel()) {
    val count by viewModel.count.collectAsState()
    Text("Count: $count")
}
```

---

## 4. ENVIRONMENT / SHARED STATE

### 4.1 SwiftUI: @Environment (iOS 17+)

#### Moderne Environment mit Type-Safe Access

**Ab iOS 17 verwende:**
- `.environment()` statt `.environmentObject()`
- `@Environment(Type.self)` statt `@EnvironmentObject`

#### Beispiel: App Settings

```swift
@Observable
class AppSettings {
    var isDarkMode = false
    var fontSize: Double = 16
    var language = "en"
}

@main
struct MyApp: App {
    @State private var settings = AppSettings()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(settings)  // ← Modern!
        }
    }
}

// Verwendung in beliebiger View
struct SettingsView: View {
    @Environment(AppSettings.self) private var settings
    
    var body: some View {
        Form {
            Toggle("Dark Mode", isOn: $settings.isDarkMode)
            
            Slider(value: $settings.fontSize, in: 12...24) {
                Text("Font Size")
            }
            
            Picker("Language", selection: $settings.language) {
                Text("English").tag("en")
                Text("Deutsch").tag("de")
            }
        }
    }
}

// In einer anderen View
struct ContentView: View {
    @Environment(AppSettings.self) private var settings
    
    var body: some View {
        Text("Hello World")
            .font(.system(size: settings.fontSize))
            .foregroundColor(settings.isDarkMode ? .white : .black)
    }
}
```

---

### 4.2 Jetpack Compose: CompositionLocal

#### Was ist CompositionLocal?

`CompositionLocal` ermöglicht das Teilen von Daten im Composition Tree ohne explizite Parameter-Übergabe.

#### Grundlagen

```kotlin
// 1. Definiere CompositionLocal
val LocalAppSettings = compositionLocalOf { AppSettings() }

data class AppSettings(
    val isDarkMode: Boolean = false,
    val fontSize: Float = 16f,
    val language: String = "en"
)

// 2. Provide im Root
@Composable
fun App() {
    val settings = remember { mutableStateOf(AppSettings()) }
    
    CompositionLocalProvider(LocalAppSettings provides settings.value) {
        // Alle Child Composables haben Zugriff
        MainScreen()
    }
}

// 3. Consume in Child Composables
@Composable
fun SettingsScreen() {
    val settings = LocalAppSettings.current
    
    Column {
        Text("Dark Mode: ${settings.isDarkMode}")
        Text("Font Size: ${settings.fontSize}")
        Text("Language: ${settings.language}")
    }
}
```

#### Mit ViewModel für Shared State

```kotlin
class AppSettingsManager {
    private val _settings = MutableStateFlow(AppSettings())
    val settings: StateFlow<AppSettings> = _settings.asStateFlow()
    
    fun updateDarkMode(enabled: Boolean) {
        _settings.value = _settings.value.copy(isDarkMode = enabled)
    }
    
    fun updateFontSize(size: Float) {
        _settings.value = _settings.value.copy(fontSize = size)
    }
}

val LocalSettingsManager = compositionLocalOf { AppSettingsManager() }

@Composable
fun AppRoot() {
    val settingsManager = remember { AppSettingsManager() }
    
    CompositionLocalProvider(LocalSettingsManager provides settingsManager) {
        val settings by settingsManager.settings.collectAsState()
        
        MaterialTheme(
            colorScheme = if (settings.isDarkMode) darkColorScheme() else lightColorScheme()
        ) {
            MainScreen()
        }
    }
}
```

---

## 5. VERGLEICHSTABELLE

| Feature | SwiftUI | Jetpack Compose |
|---------|---------|-----------------|
| **Lokaler State** | `@State private var x = 0` | `var x by remember { mutableStateOf(0) }` |
| **Two-Way Binding** | `@Binding var x: Int` | `x: Int, onXChange: (Int) -> Unit` |
| **Observable Class** | `@Observable class VM` | `StateFlow` in ViewModel |
| **View Ownership** | `@State private var vm = VM()` | `val vm: VM = viewModel()` |
| **Shared State** | `@Environment(Settings.self)` | `CompositionLocal` |
| **Persistence** | `@AppStorage` | `rememberSaveable` / DataStore |
| **Computed Properties** | Direkt im ViewModel | `derivedStateOf` oder Flow.map |

---

## 6. BEST PRACTICES

### State Management Entscheidungsbaum

```
Brauche ich State?
├─ Ja
│  ├─ Nur für diese View? → @State / remember
│  ├─ Für Parent/Child? → @Binding / State Parameter
│  ├─ Komplexe Logik? → ViewModel mit @Observable/StateFlow
│  ├─ App-weit? → @Environment / CompositionLocal
│  └─ Persistent? → @AppStorage / rememberSaveable
└─ Nein → Normaler Parameter
```

### Häufige Fehler vermeiden

**SwiftUI:**
```swift
// ❌ State nicht private
@State var count = 0

// ✅ Immer private
@State private var count = 0

// ❌ State als Parameter
struct MyView: View {
    @State var value: Int  // FALSCH!
}

// ✅ Als normaler Parameter
struct MyView: View {
    let value: Int
}
```

**Jetpack Compose:**
```kotlin
// ❌ Kein remember
var count = 0  // Verloren bei Recompose

// ✅ Mit remember
var count by remember { mutableStateOf(0) }

// ❌ ViewModel bei jedem Recompose neu
@Composable
fun Screen(vm: VM = VM())  // FALSCH!

// ✅ ViewModel cachen
@Composable
fun Screen(vm: VM = viewModel())  // RICHTIG!
```

---

