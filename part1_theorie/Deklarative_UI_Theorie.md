# 🧠 Part 1 – Grundlagen der Deklarativen UI

## 🎯 Ziel
Verstehen, was deklarative UI-Entwicklung bedeutet, wie sie sich von imperativer unterscheidet, und wie moderne Frameworks wie **SwiftUI** (iOS) und **Jetpack Compose** (Android) das Prinzip umsetzen.

---

## 🔹 Deklarativ vs. Imperativ

| Ansatz | Fokus | Beispiel | Beschreibung |
|--------|--------|-----------|---------------|
| **Deklarativ** | *Was soll angezeigt werden?* | `Text("Hello, World!")` | UI wird beschrieben, nicht manuell gesteuert. SwiftUI & Compose folgen diesem Prinzip. |
| **Imperativ** | *Wie wird es angezeigt?* | `label.text = "Hello, World!"` | Entwickler ändern den Zustand und das Layout aktiv im Code. |

### 🧩 Beispielvergleich

**Kotlin (imperativ):**
```kotlin
val textView = findViewById<TextView>(R.id.textView)
val button = findViewById<Button>(R.id.button)
var count = 0
button.setOnClickListener {
    count++
    textView.text = "Count: $count"
}
```

**Kotlin (deklarativ mit Jetpack Compose):**
```kotlin
var count by remember { mutableStateOf(0) }
Column {
    Text("Count: $count")
    Button(onClick = { count++ }) {
        Text("Increment")
    }
}
```

**SwiftUI (deklarativ):**
```swift
@State private var count = 0

var body: some View {
    VStack {
        Text("Count: \(count)")
        Button("Increment") { count += 1 }
    }
}
```

---

## ✅ Vorteile der Deklarativen UI

1. **Intuitiver Code** – Lesbarer, näher an der mentalen Vorstellung des UIs.  
2. **Weniger Boilerplate** – Fokus auf *was*, nicht *wie*.  
3. **Automatische Zustandsaktualisierung** – Änderungen am Zustand werden automatisch reflektiert.  
4. **Bessere Wartbarkeit** – UI ist eine reine Funktion des Zustands.  
5. **Performance-Optimierungen** – Frameworks rendern nur geänderte Komponenten neu.  

---

## 🔄 Unidirectional Data Flow (UDF)
> Deklarative Frameworks folgen einem **einheitlichen Datenfluss**:
>
> **State → UI → User Action → State**
>
> Das garantiert vorhersehbares Verhalten und macht Debugging einfacher.

```swift
// SwiftUI
struct Counter: View {
    @State private var count = 0
    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Add") { count += 1 }
        }
    }
}
```

```kotlin
// Compose
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) { Text("Add") }
    }
}
```

---

## ⚡ Performance-Tipps

- **SwiftUI:** „Body rebuild ≠ View rebuild“ – nur veränderte Subviews werden neu berechnet.  
- **Compose:** *Smart Recomposition* über die Slot Table; nur betroffene Composables werden neu gezeichnet.  
- **Praxis:** Zustand granular halten (nicht ganze Objekte beobachten, sondern Properties).  

---

## ⚠️ Häufige Fehler (Pitfalls)

- Zustand falsch platziert (global statt `@State`/`remember`)  
- Mehrere Quellen für denselben Zustand → inkonsistente UI  
- ViewModels in Subviews neu instanziiert → State-Verlust bei Rebuilds  

✅ **Tipp:** *Single Source of Truth* & Übergabe via Bindings/Observables.

---

## 📚 SwiftUI – Grundkomponenten

```swift
VStack(spacing: 20) {
    Text("Hello World").font(.title)
    Button("Tap me") { print("Tapped") }
}
```

**Layouts:** `VStack`, `HStack`, `ZStack`, `NavigationStack`, `ScrollView`  
**Zustand:** `@State`, `@Binding`, `@Observable` (iOS 17+), `@Environment(Type.self)`

---

## 📱 Jetpack Compose – Grundkomponenten

```kotlin
Column(verticalArrangement = Arrangement.spacedBy(16.dp)) {
    Text("Hello World", fontSize = 24.sp)
    Button(onClick = { /* TODO */ }) { Text("Tap me") }
}
```

**Layouts:** `Column`, `Row`, `Box`  
**Zustand:** `remember`, `mutableStateOf`, `rememberSaveable`  
**Shared State:** `StateFlow`, `ViewModel`

---

## 🧩 Vergleich SwiftUI vs. Jetpack Compose

| Kategorie | SwiftUI | Jetpack Compose |
|------------|----------|----------------|
| Zustand | `@State`, `@Observable`, `@Environment` | `remember`, `StateFlow`, `rememberSaveable` |
| Architektur | MVVM | MVVM / MVI |
| Sprache | Swift | Kotlin |
| Rendering | Value-basierte Views | Composable Functions |
| Lifecycle | iOS Scene / SwiftUI App | Android Lifecycle (ViewModelScope) |
| Modernes State-System | Observation Framework (iOS 17+) | Compose Runtime 1.7+ (SnapshotFlow) |

---

## 💬 Zusammenfassung
Deklarative Frameworks definieren das UI als Funktion des Zustands. Das führt zu klarerem, testbarem Code – ein Kernprinzip moderner Mobile-Entwicklung.
