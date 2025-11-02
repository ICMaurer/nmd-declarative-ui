# Deklarative UI - Theorie & Grundlagen
## Native Mobile Development mit SwiftUI & Jetpack Compose

---

## 1. GRUNDKONZEPTE

### 1.1 Deklarativ vs. Imperativ: Was vs. Wie

#### Das fundamentale Paradigma

**Deklarative UI-Gestaltung (What):**
- Beschreibt **WAS** die UI darstellen soll
- UI ist eine **Funktion des Zustands**: `UI = f(State)`
- UI-Updates erfolgen **automatisch** bei Zustandsänderungen
- Fokus auf das **Endergebnis**, nicht den Weg dorthin

**Imperative UI-Gestaltung (How):**
- Definiert **WIE** UI-Elemente manipuliert werden
- Entwickler müssen UI-Updates **manuell** durchführen
- Explizite Steuerung jeder Zustandsänderung
- Fokus auf die **Schritte** zur Erreichung des Ziels

#### Visueller Vergleich

```
IMPERATIV (How):                    DEKLARATIV (What):
┌──────────────────┐               ┌──────────────────┐
│ 1. Find TextView │               │ State: count = 5 │
│ 2. Update Text   │               │                  │
│ 3. Find Button   │               │ UI = f(State)    │
│ 4. Add Listener  │               │                  │
│ 5. Update Color  │               │ Text("Count: 5") │
│ ...              │               └──────────────────┘
└──────────────────┘               
```

#### Code-Beispiel: Counter

**Imperativ (Traditionelles Android):**
```kotlin
// Manuelles UI-Management
val textView = findViewById<TextView>(R.id.textView)
val button = findViewById<Button>(R.id.button)
var count = 0

button.setOnClickListener {
    count++
    textView.text = "Count: $count"        // Manuelles Update
    if (count >= 10) {
        textView.setTextColor(Color.RED)   // Manuelle Bedingung
    }
}
```

**Deklarativ (Jetpack Compose):**
```kotlin
// Automatisches UI-Management
var count by remember { mutableStateOf(0) }

Column {
    Text(
        text = "Count: $count",
        color = if (count >= 10) Color.Red else Color.Black
    )
    Button(onClick = { count++ }) {
        Text("Increment")
    }
}
```

**SwiftUI:**
```swift
@State private var count = 0

VStack {
    Text("Count: \(count)")
        .foregroundColor(count >= 10 ? .red : .black)
    Button("Increment") {
        count += 1
    }
}
```

---

### 1.2 Der State → UI → Action Loop

#### Das Herzstück deklarativer UIs

```
           ┌─────────────────────────────────────┐
           │                                     │
           │        UNIDIRECTIONAL FLOW          │
           │                                     │
           ▼                                     │
      ┌─────────┐           ┌─────────┐    ┌─────────┐
      │  STATE  │ ────────▶ │   UI    │───▶│ ACTION  │
      │         │  render   │         │ evt│         │
      └─────────┘           └─────────┘    └─────────┘
           ▲                                     │
           │                                     │
           └─────────────────────────────────────┘
                        update state
```

**Schritte im Detail:**

1. **State** (Zustand): 
   - Enthält alle UI-relevanten Daten
   - Single Source of Truth
   - Beispiel: `count = 0`

2. **UI Rendering**: 
   - UI wird basierend auf State berechnet
   - Automatische Aktualisierung bei State-Änderung
   - Beispiel: `Text("Count: 0")`

3. **User Action** (Event):
   - Benutzerinteraktion (Click, Input, etc.)
   - Triggert State-Änderung
   - Beispiel: Button-Klick

4. **State Update**:
   - Neuer State wird berechnet
   - Loop beginnt von vorne
   - Beispiel: `count = 1`

#### Praktisches Beispiel

```swift
// SwiftUI: State → UI → Action Loop
struct LoginView: View {
    // STATE
    @State private var username = ""
    @State private var password = ""
    @State private var isLoading = false
    
    // UI (Funktion des States)
    var body: some View {
        VStack {
            TextField("Username", text: $username)
            SecureField("Password", text: $password)
            
            Button("Login") {
                // ACTION
                performLogin()
            }
            .disabled(username.isEmpty || isLoading)
            
            if isLoading {
                ProgressView()
            }
        }
    }
    
    // STATE UPDATE
    func performLogin() {
        isLoading = true
        // ... Login-Logik
        isLoading = false
    }
}
```

---

### 1.3 Single Source of Truth (SSOT)

#### Was bedeutet Single Source of Truth?

**Definition:** Jedes Datenelement sollte **genau eine** autoritative Quelle haben.

**Vorteile:**
- ✅ Keine Synchronisationsprobleme
- ✅ Einfachere Fehlersuche
- ✅ Konsistente Daten
- ✅ Weniger Bugs

#### Anti-Pattern: Mehrere Sources

```swift
// ❌ SCHLECHT: Duplicate State
struct ProfileView: View {
    @State private var localUsername = ""      // Source 1
    @ObservedObject var viewModel: ProfileVM   // Source 2
    // viewModel.username ist die echte Quelle!
    
    var body: some View {
        // Welcher Username ist korrekt? 🤔
        Text(localUsername)  // Oder viewModel.username?
    }
}
```

#### Best Practice: Single Source

```swift
// ✅ GUT: Single Source of Truth
struct ProfileView: View {
    @ObservedObject var viewModel: ProfileVM
    
    var body: some View {
        Text(viewModel.username)  // Eine Quelle = Eine Wahrheit
    }
}

@Observable
class ProfileVM {
    var username = ""  // DIE autoritative Quelle
}
```

---

### 1.4 Unidirectional Data Flow (UDF)

#### Das Prinzip

**Daten fließen nur in EINE Richtung:**

```
┌────────────────────────────────────────┐
│  Parent Component                      │
│  ┌────────────┐                        │
│  │   State    │                        │
│  └──────┬─────┘                        │
│         │ ↓ Props/Data                 │
│  ┌──────▼─────────────┐                │
│  │  Child Component   │                │
│  │                    │                │
│  │  ┌──────────────┐  │                │
│  │  │   Action     │──┼─── Events ────▶│
│  │  └──────────────┘  │      ↑         │
│  └────────────────────┘      │         │
│                              │         │
└──────────────────────────────┼─────────┘
                               │
                          State Update
```

**Regeln:**
1. **Daten fließen nach unten** (Parent → Child)
2. **Events fließen nach oben** (Child → Parent)
3. **State lebt im Parent** (oder ViewModel)
4. **Children sind "dumb"** (kennen keine Logik)

#### Beispiel

```swift
// SwiftUI: Unidirectional Data Flow
struct ParentView: View {
    @State private var count = 0  // State im Parent
    
    var body: some View {
        VStack {
            // Daten fließen nach unten ↓
            ChildView(
                currentCount: count,
                // Events fließen nach oben ↑
                onIncrement: { count += 1 },
                onDecrement: { count -= 1 }
            )
        }
    }
}

struct ChildView: View {
    let currentCount: Int           // Empfängt Daten
    let onIncrement: () -> Void     // Sendet Events
    let onDecrement: () -> Void
    
    var body: some View {
        HStack {
            Button("-") { onDecrement() }
            Text("\(currentCount)")
            Button("+") { onIncrement() }
        }
    }
}
```

---

## 2. RECOMPOSITION & RE-RENDERING

### 2.1 Was passiert bei State-Änderungen?

#### SwiftUI: Body Rebuild ≠ View Rebuild

**Wichtig zu verstehen:**

```
State Change → body wird aufgerufen → Diff berechnet → Nur Änderungen angewendet
```

**Beispiel:**
```swift
struct ContentView: View {
    @State private var count = 0
    @State private var name = "Max"
    
    var body: some View {  // ← Wird bei JEDER State-Änderung aufgerufen
        VStack {
            Text("Count: \(count)")      // ← Wird NUR bei count-Änderung gerendert
            Text("Name: \(name)")         // ← Wird NUR bei name-Änderung gerendert
        }
    }
}
```

**Was passiert:**
1. User klickt Button → `count` ändert sich
2. SwiftUI ruft `body` auf → Erstellt neuen View-Tree
3. SwiftUI vergleicht alten & neuen Tree (Diffing)
4. Nur `Text("Count: ...")` wird im UI aktualisiert
5. `Text("Name: ...")` bleibt unverändert

#### Jetpack Compose: Smart Recomposition

**Compose ist noch intelligenter:**

```
State Change → Nur betroffene Composables werden recomposed
```

**Beispiel:**
```kotlin
@Composable
fun ContentScreen() {
    var count by remember { mutableStateOf(0) }
    var name by remember { mutableStateOf("Max") }
    
    Column {
        Text("Count: $count")     // ← Recompose nur bei count-Änderung
        Text("Name: $name")       // ← Recompose nur bei name-Änderung
        Button(onClick = { count++ }) { Text("Click") }
    }
}
```

**Compose Slot Table:**
- Compose merkt sich welche Composables welchen State lesen
- Bei State-Änderung werden nur die betroffenen Composables neu berechnet
- Extrem effizient!

### 2.2 Performance-Visualisierung

```
┌─────────────────────────────────────────────────┐
│ SwiftUI View Hierarchy                          │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌─────────────────────────────────┐            │
│  │ VStack (body wird aufgerufen)   │            │
│  │  ├─ Text("Count: 5") ✓ UPDATE   │            │
│  │  ├─ Text("Name: Max")           │            │
│  │  └─ Button                      │            │
│  └─────────────────────────────────┘            │
│                                                 │
│  Body Rebuild ≠ Full View Rebuild               │
│  Nur geänderte Elemente werden gerendert        │
└─────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────┐
│ Jetpack Compose Slot Table                      │
├─────────────────────────────────────────────────┤
│                                                 │
│  Slot 0: Column                                 │
│  Slot 1: Text("Count") → reads: count ✓ RECOMP  │
│  Slot 2: Text("Name")  → reads: name            │
│  Slot 3: Button        → reads: count           │
│                                                 │
│  Smart Recomposition:                           │
│  Nur Slots die geänderten State lesen           │
└─────────────────────────────────────────────────┘
```

---

## 3. PERFORMANCE-OPTIMIERUNG

### 3.1 SwiftUI Performance

#### Derived State statt Duplicate State

```swift
// ❌ SCHLECHT: Redundanter State
@State private var items: [Item] = []
@State private var itemCount: Int = 0      // Duplicate!
@State private var hasItems: Bool = false   // Duplicate!

// ✅ GUT: Computed Properties
@State private var items: [Item] = []

var itemCount: Int { items.count }
var hasItems: Bool { !items.isEmpty }
```

#### State minimieren

```swift
// ❌ SCHLECHT: Zu viel State
@State private var firstName = ""
@State private var lastName = ""
@State private var fullName = ""  // Wird automatisch berechnet werden kann

// ✅ GUT: Minimaler State
@State private var firstName = ""
@State private var lastName = ""
var fullName: String { "\(firstName) \(lastName)" }
```

### 3.2 Compose Performance

#### Remember für teure Berechnungen

```kotlin
@Composable
fun ExpensiveList(items: List<Item>) {
    // ❌ SCHLECHT: Berechnung bei jedem Recompose
    val filteredItems = items.filter { it.isActive }
    
    // ✅ GUT: Remember cached das Ergebnis
    val filteredItems = remember(items) {
        items.filter { it.isActive }
    }
    
    LazyColumn {
        items(filteredItems) { item ->
            ItemRow(item)
        }
    }
}
```

#### derivedStateOf für abhängige States

```kotlin
@Composable
fun ScrollableList() {
    val listState = rememberLazyListState()
    
    // ❌ SCHLECHT: Recompose bei jedem Scroll-Event
    val isAtTop = listState.firstVisibleItemIndex == 0
    
    // ✅ GUT: Nur Recompose wenn sich Wert ändert
    val isAtTop by remember {
        derivedStateOf {
            listState.firstVisibleItemIndex == 0
        }
    }
}
```

### 3.3 Performance-Tabelle

| Konzept | SwiftUI | Jetpack Compose |
|---------|---------|-----------------|
| **Recomposition Scope** | Body-Level | Composable-Level |
| **Diffing** | Virtual Tree Diff | Slot Table |
| **State Tracking** | Dependency Tracking | Read/Write Observation |
| **Optimization** | Automatic | Automatic + derivedStateOf |
| **Performance** | Sehr gut | Exzellent |

---

## 4. ZUSAMMENFASSUNG

### Die Kernprinzipien deklarativer UIs

1. **UI = f(State)**
   - UI ist immer eine direkte Funktion des aktuellen States

2. **Unidirectional Data Flow**
   - Daten fließen nur in eine Richtung
   - Events fließen zurück

3. **Single Source of Truth**
   - Eine autoritative Quelle pro Datenelement
   - Keine Duplikation von State

4. **Automatische Updates**
   - Framework kümmert sich um Re-Rendering
   - Entwickler beschreibt nur das "Was"

5. **Intelligentes Re-Rendering**
   - Nur betroffene Komponenten werden aktualisiert
   - Performance durch Smart Diffing/Recomposition

### Vorteile deklarativer UIs

✅ **Weniger Code** - Fokus auf das Wesentliche
✅ **Weniger Bugs** - Keine manuellen Update-Fehler
✅ **Bessere Lesbarkeit** - Code = UI-Beschreibung
✅ **Einfacheres Testing** - UI ist pure Funktion
✅ **Bessere Performance** - Framework-Optimierungen
✅ **Modernere DX** - Entwicklerfreundlichkeit

---

## 5. WEITERFÜHRENDE KONZEPTE

### State Hoisting

```
Child Component kennt State nicht
         ↑
         │ Events
         │
    ┌────┴────┐
    │  State  │ ← Lebt hier
    └─────────┘
         │
         │ Data
         ↓
Child Component empfängt Daten
```

### Composition over Inheritance

- Kleine, wiederverwendbare Komponenten
- Komponenten werden kombiniert, nicht erweitert
- Flexiblere Architektur

### Reactive Programming

- State-Änderungen propagieren automatisch
- Asynchrone Datenströme (Flows, Combine)
- Observable-Pattern eingebaut

---

**Next Steps:** 
- [State Management Basics]
- [State Management Advanced]
- [MVVM Konzepte]
