# ⚙️ Part 2 – State Management in Deklarativer UI

## 🎯 Ziel
Verstehen, wie **Zustandsverwaltung (State Management)** in SwiftUI und Jetpack Compose funktioniert – von einfachen lokalen Zuständen bis zu komplexen, gemeinsam genutzten Datenstrukturen.

---

## 🧩 1. Grundlagen des State Managements

> **Definition:**  
> State beschreibt den *aktuellen Zustand* einer App oder View.  
> Deklarative Frameworks leiten die gesamte UI direkt aus diesem Zustand ab.

### 🔄 Der State-Zyklus
```
Zustand (State) → UI wird berechnet → User-Interaktion → Zustand ändert sich → UI aktualisiert sich
```

---

## 🍏 SwiftUI State Management

### 1) `@State` – Lokaler View-Zustand
```swift
struct CounterView: View {
    @State private var count = 0
    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") { count += 1 }
        }
    }
}
```
**Regeln:** lokal/privat, Value Types, triggert Rebuild.

### 2) `@Binding` – Bidirektionale Datenverbindung
```swift
struct ParentView: View {
    @State private var isOn = false
    var body: some View { ToggleView(isOn: $isOn) }
}
struct ToggleView: View {
    @Binding var isOn: Bool
    var body: some View { Toggle("Switch", isOn: $isOn) }
}
```
**Merke:** Binding ist eine Referenz auf `@State` einer Parent-View.

### 3) `@Observable` – Modern (iOS 17+)
```swift
import Observation
@Observable class UserViewModel {
    var username = ""
    var isLoggedIn = false
    func login() { isLoggedIn = true }
}
struct LoginView: View {
    @State private var viewModel = UserViewModel()
    var body: some View {
        VStack {
            TextField("Username", text: $viewModel.username)
            Button("Login") { viewModel.login() }
        }
    }
}
```
**Vorteile:** Kein `@Published`/`ObservableObject`, weniger Boilerplate, computed properties funktionieren nativ.

### 4) `@Environment` – App-weiter Zustand
```swift
@Observable class AppSettings { var isDarkMode = false }
@main struct MyApp: App {
    @State private var settings = AppSettings()
    var body: some Scene {
        WindowGroup { ContentView().environment(settings) }
    }
}
struct ContentView: View {
    @Environment(AppSettings.self) private var settings
    var body: some View { Toggle("Dark Mode", isOn: $settings.isDarkMode) }
}
```

---

## 🤖 Jetpack Compose State Management

### 1) `remember` & `mutableStateOf`
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) { Text("Add") }
    }
}
```

### 2) `rememberSaveable` – Konfigurationswechsel überleben
```kotlin
@Composable
fun PersistentCounter() {
    var count by rememberSaveable { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("Count: $count") }
}
```

### 3) ViewModel + StateFlow – Shared State
```kotlin
class UserViewModel : ViewModel() {
    private val _username = MutableStateFlow("")
    val username: StateFlow<String> = _username.asStateFlow()
    fun updateUsername(newName: String) { _username.value = newName }
}
@Composable
fun UserScreen(viewModel: UserViewModel = viewModel()) {
    val username by viewModel.username.collectAsState()
    TextField(value = username, onValueChange = { viewModel.updateUsername(it) })
}
```
**Best Practice (2025):** `StateFlow` > `LiveData`; `collectAsState()` verbindet automatisch mit der UI.

### 4) Performance & UDF in Compose
- `derivedStateOf` für berechnete Werte nutzen
- Logik ins ViewModel verlagern, nicht in die Composable
- Zustände klein & fokussiert halten

---

## ⚠️ Fehler & Best Practices

| Fehler | Ursache | Lösung |
|--------|----------|--------|
| ViewModel in Subview neu erstellt | Rebuild zerstört State | ViewModel im Parent halten (`@State`/`viewModel()`) |
| Mehrere States für denselben Wert | Inkonsistente Daten | Single Source of Truth |
| Zustand außerhalb `remember`/`@State` | Kein Recomposition-Trigger | State korrekt deklarieren |

✅ **Merksatz:** *State klar, eindeutig, vorhersehbar halten.*

---

## 🧠 Zusammenfassung
- SwiftUI & Compose basieren beide auf **reaktivem State Management**.  
- Moderne Systeme (`@Observable`, `StateFlow`) reduzieren Boilerplate.  
- Klarer **UDF-Datenfluss** sorgt für Stabilität und Performance.
