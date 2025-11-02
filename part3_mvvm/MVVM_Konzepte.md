# MVVM - Model-View-ViewModel Pattern
## Architecture für SwiftUI & Jetpack Compose

---

## 1. MVVM ÜBERSICHT

### 1.1 Was ist MVVM?

**Model-View-ViewModel** ist ein Architekturmuster für UI-Anwendungen, das **Separation of Concerns** ermöglicht.

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  ┌────────────┐         ┌──────────────┐         ┌───────┐
│  │    View    │────────▶│  ViewModel   │────────▶│ Model │
│  │            │◀────────│              │◀────────│       │
│  └────────────┘         └──────────────┘         └───────┘
│       UI               Logik/State              Daten
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Kernprinzip:** Jede Schicht hat eine klare Verantwortung!

---

### 1.2 Die drei Komponenten

#### Model

**Verantwortung:** Daten und Geschäftslogik

```swift
// Model - Pure Data
struct User {
    let id: UUID
    var name: String
    var email: String
    var isActive: Bool
}

struct Product {
    let id: UUID
    let name: String
    let price: Double
    var stockCount: Int
}
```

```kotlin
// Model - Pure Data
data class User(
    val id: String,
    val name: String,
    val email: String,
    val isActive: Boolean
)

data class Product(
    val id: String,
    val name: String,
    val price: Double,
    val stockCount: Int
)
```

**Charakteristiken:**
- ✅ Einfache Datenstrukturen
- ✅ Keine UI-Logik
- ✅ Keine Framework-Abhängigkeiten
- ✅ Leicht testbar

---

#### ViewModel

**Verantwortung:** UI-Logik und State Management

```swift
// SwiftUI ViewModel
@Observable
class UserViewModel {
    // State
    var user: User?
    var isLoading = false
    var errorMessage: String?
    
    // Computed Properties
    var displayName: String {
        user?.name ?? "Unknown"
    }
    
    var canEdit: Bool {
        user?.isActive ?? false
    }
    
    // Actions
    func loadUser() async {
        isLoading = true
        errorMessage = nil
        
        do {
            user = try await userService.fetchUser()
        } catch {
            errorMessage = error.localizedDescription
        }
        
        isLoading = false
    }
    
    func updateName(_ newName: String) {
        guard var currentUser = user else { return }
        currentUser.name = newName
        user = currentUser
    }
}
```

```kotlin
// Jetpack Compose ViewModel
class UserViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    
    // State
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()
    
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
    
    private val _errorMessage = MutableStateFlow<String?>(null)
    val errorMessage: StateFlow<String?> = _errorMessage.asStateFlow()
    
    // Computed Properties
    val displayName: StateFlow<String> = user.map { it?.name ?: "Unknown" }
        .stateIn(viewModelScope, SharingStarted.Lazily, "Unknown")
    
    val canEdit: StateFlow<Boolean> = user.map { it?.isActive ?: false }
        .stateIn(viewModelScope, SharingStarted.Lazily, false)
    
    // Actions
    fun loadUser() {
        viewModelScope.launch {
            _isLoading.value = true
            _errorMessage.value = null
            
            try {
                val user = userRepository.getUser()
                _user.value = user
            } catch (e: Exception) {
                _errorMessage.value = e.message
            } finally {
                _isLoading.value = false
            }
        }
    }
    
    fun updateName(newName: String) {
        _user.value = _user.value?.copy(name = newName)
    }
}
```

**Charakteristiken:**
- ✅ Verwaltet UI State
- ✅ Enthält UI-Logik
- ✅ Kommuniziert mit Model/Services
- ✅ Keine direkte UI-Referenz
- ✅ Testbar ohne UI

---

#### View

**Verantwortung:** UI-Darstellung

```swift
// SwiftUI View
struct UserProfileView: View {
    @State private var viewModel = UserViewModel()
    
    var body: some View {
        VStack {
            if viewModel.isLoading {
                ProgressView()
            } else if let error = viewModel.errorMessage {
                ErrorView(message: error)
            } else {
                ProfileContent(
                    displayName: viewModel.displayName,
                    canEdit: viewModel.canEdit,
                    onNameChange: { viewModel.updateName($0) }
                )
            }
        }
        .task {
            await viewModel.loadUser()
        }
    }
}
```

```kotlin
// Jetpack Compose View
@Composable
fun UserProfileScreen(
    viewModel: UserViewModel = viewModel()
) {
    val user by viewModel.user.collectAsState()
    val isLoading by viewModel.isLoading.collectAsState()
    val errorMessage by viewModel.errorMessage.collectAsState()
    val displayName by viewModel.displayName.collectAsState()
    val canEdit by viewModel.canEdit.collectAsState()
    
    LaunchedEffect(Unit) {
        viewModel.loadUser()
    }
    
    Column {
        when {
            isLoading -> CircularProgressIndicator()
            errorMessage != null -> ErrorView(errorMessage!!)
            else -> ProfileContent(
                displayName = displayName,
                canEdit = canEdit,
                onNameChange = { viewModel.updateName(it) }
            )
        }
    }
}
```

**Charakteristiken:**
- ✅ Nur UI-Darstellung
- ✅ Keine Geschäftslogik
- ✅ Bindet an ViewModel
- ✅ Reagiert auf State-Änderungen

---

## 2. MVVM KOMPAKT-TABELLE

### Model ↔ ViewModel ↔ View

| Komponente | Verantwortung | SwiftUI | Jetpack Compose |
|------------|---------------|---------|-----------------|
| **Model** | Daten | `struct User` | `data class User` |
| **ViewModel** | State + Logik | `@Observable class VM` | `class VM : ViewModel()` |
| | State Declaration | `var name = ""` | `MutableStateFlow("")` |
| | Computed Props | `var computed: String { }` | `StateFlow<String>` mit `.map` |
| | Async Actions | `func load() async` | `viewModelScope.launch` |
| **View** | UI | `struct ContentView: View` | `@Composable fun Screen()` |
| | ViewModel Binding | `@State private var vm = VM()` | `val vm: VM = viewModel()` |
| | State Observation | Automatic | `vm.state.collectAsState()` |
| | Event Handling | `vm.action()` | `vm.action()` |

---

## 3. DATENFLUSS IN MVVM

### 3.1 Unidirectional Data Flow

```
┌───────────────────────────────────────────────┐
│                                               │
│                  ViewModel                    │
│  ┌───────────────────────────────────────┐    │
│  │           State                       │    │
│  │  • user                               │    │
│  │  • isLoading                          │    │
│  │  • errorMessage                       │    │
│  └───────────┬───────────────────────────┘    │
│              │                                │
│              │ Emit Changes                   │
│              ↓                                │
│  ┌──────────────────────────────────────┐     │
│  │           View                       │     │
│  │  Displays State                      │     │
│  │  User Interaction                    │     │
│  └──────────┬───────────────────────────┘     │
│             │                                 │
│             │ Trigger Actions                 │
│             ↓                                 │
│  ┌──────────────────────────────────────┐     │
│  │        Actions/Methods               │     │
│  │  • loadUser()                        │     │
│  │  • updateName()                      │     │
│  │  • delete()                          │     │
│  └──────────┬───────────────────────────┘     │
│             │                                 │
│             │ Update State                    │
│             ↓                                 │
│       Back to State                           │
└───────────────────────────────────────────────┘
```

---

### 3.2 Praktisches Beispiel: Login Flow

#### Model

```swift
struct LoginCredentials {
    let username: String
    let password: String
}

struct AuthToken {
    let token: String
    let expiresAt: Date
}
```

```kotlin
data class LoginCredentials(
    val username: String,
    val password: String
)

data class AuthToken(
    val token: String,
    val expiresAt: Long
)
```

---

#### ViewModel

```swift
@Observable
class LoginViewModel {
    // State
    var username = ""
    var password = ""
    var isLoading = false
    var errorMessage: String?
    var isAuthenticated = false
    
    // Computed
    var canLogin: Bool {
        !username.isEmpty && !password.isEmpty && !isLoading
    }
    
    // Action
    func login() async {
        guard canLogin else { return }
        
        isLoading = true
        errorMessage = nil
        
        do {
            let credentials = LoginCredentials(
                username: username,
                password: password
            )
            let token = try await authService.login(credentials)
            isAuthenticated = true
        } catch {
            errorMessage = "Login failed: \(error.localizedDescription)"
        }
        
        isLoading = false
    }
}
```

```kotlin
class LoginViewModel(
    private val authRepository: AuthRepository
) : ViewModel() {
    
    // State
    private val _username = MutableStateFlow("")
    val username: StateFlow<String> = _username.asStateFlow()
    
    private val _password = MutableStateFlow("")
    val password: StateFlow<String> = _password.asStateFlow()
    
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
    
    private val _errorMessage = MutableStateFlow<String?>(null)
    val errorMessage: StateFlow<String?> = _errorMessage.asStateFlow()
    
    private val _isAuthenticated = MutableStateFlow(false)
    val isAuthenticated: StateFlow<Boolean> = _isAuthenticated.asStateFlow()
    
    // Computed
    val canLogin: StateFlow<Boolean> = combine(
        username, password, isLoading
    ) { user, pass, loading ->
        user.isNotEmpty() && pass.isNotEmpty() && !loading
    }.stateIn(viewModelScope, SharingStarted.Lazily, false)
    
    // Actions
    fun updateUsername(value: String) {
        _username.value = value
    }
    
    fun updatePassword(value: String) {
        _password.value = value
    }
    
    fun login() {
        viewModelScope.launch {
            _isLoading.value = true
            _errorMessage.value = null
            
            try {
                val credentials = LoginCredentials(
                    username = _username.value,
                    password = _password.value
                )
                val token = authRepository.login(credentials)
                _isAuthenticated.value = true
            } catch (e: Exception) {
                _errorMessage.value = "Login failed: ${e.message}"
            } finally {
                _isLoading.value = false
            }
        }
    }
}
```

---

#### View

```swift
struct LoginView: View {
    @State private var viewModel = LoginViewModel()
    
    var body: some View {
        VStack(spacing: 20) {
            TextField("Username", text: $viewModel.username)
                .textFieldStyle(.roundedBorder)
                .textInputAutocapitalization(.never)
            
            SecureField("Password", text: $viewModel.password)
                .textFieldStyle(.roundedBorder)
            
            if let error = viewModel.errorMessage {
                Text(error)
                    .foregroundColor(.red)
                    .font(.caption)
            }
            
            Button("Login") {
                Task { await viewModel.login() }
            }
            .buttonStyle(.borderedProminent)
            .disabled(!viewModel.canLogin)
            
            if viewModel.isLoading {
                ProgressView()
            }
        }
        .padding()
    }
}
```

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = viewModel()
) {
    val username by viewModel.username.collectAsState()
    val password by viewModel.password.collectAsState()
    val isLoading by viewModel.isLoading.collectAsState()
    val errorMessage by viewModel.errorMessage.collectAsState()
    val canLogin by viewModel.canLogin.collectAsState()
    
    Column(
        modifier = Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        TextField(
            value = username,
            onValueChange = { viewModel.updateUsername(it) },
            label = { Text("Username") },
            modifier = Modifier.fillMaxWidth()
        )
        
        TextField(
            value = password,
            onValueChange = { viewModel.updatePassword(it) },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
        
        errorMessage?.let { error ->
            Text(
                text = error,
                color = MaterialTheme.colorScheme.error,
                style = MaterialTheme.typography.bodySmall
            )
        }
        
        Button(
            onClick = { viewModel.login() },
            enabled = canLogin,
            modifier = Modifier.fillMaxWidth()
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

## 4. MVVM BEST PRACTICES

### 4.1 Do's ✅

#### ViewModel

```swift
// ✅ Klare State-Definition
@Observable
class ProductViewModel {
    var products: [Product] = []
    var isLoading = false
    var error: Error?
}

// ✅ Single Responsibility
@Observable
class UserProfileViewModel {
    var profile: UserProfile?
    
    func updateProfile(_ profile: UserProfile) async {
        // Nur Profile-bezogene Logik
    }
}

// ✅ Computed Properties für derived state
@Observable
class CartViewModel {
    var items: [CartItem] = []
    
    var total: Double {
        items.reduce(0) { $0 + $1.price * Double($1.quantity) }
    }
}
```

```kotlin
// ✅ Immutable State mit StateFlow
class ProductViewModel : ViewModel() {
    private val _products = MutableStateFlow<List<Product>>(emptyList())
    val products: StateFlow<List<Product>> = _products.asStateFlow()
}

// ✅ Dependency Injection
class UserProfileViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    // ...
}

// ✅ Derived State
class CartViewModel : ViewModel() {
    private val _items = MutableStateFlow<List<CartItem>>(emptyList())
    val items: StateFlow<List<CartItem>> = _items.asStateFlow()
    
    val total: StateFlow<Double> = items.map { list ->
        list.sumOf { it.price * it.quantity }
    }.stateIn(viewModelScope, SharingStarted.Lazily, 0.0)
}
```

---

### 4.2 Don'ts ❌

```swift
// ❌ View-Referenz in ViewModel
@Observable
class BadViewModel {
    weak var view: UIViewController?  // NIEMALS!
}

// ❌ Platform-spezifische Imports
import UIKit  // ViewModel sollte UI-unabhängig sein

// ❌ Zu viel Verantwortung
@Observable
class GodViewModel {
    var users: [User] = []
    var products: [Product] = []
    var orders: [Order] = []
    var settings: Settings?
    // Zu viel! Aufspalten!
}
```

```kotlin
// ❌ Context in ViewModel (außer Application Context)
class BadViewModel(
    private val context: Context  // Activity/Fragment Context vermeiden!
) : ViewModel()

// ❌ Direct UI Updates
class BadViewModel : ViewModel() {
    fun updateUI() {
        // UI-Updates gehören in die View!
    }
}

// ❌ Exposed Mutable State
class BadViewModel : ViewModel() {
    val products = MutableStateFlow<List<Product>>(emptyList())
    // Sollte private sein, expose nur read-only StateFlow
}
```

---

## 5. TESTING MVVM

### 5.1 Warum MVVM testbar ist

**Vorteile:**
- ✅ ViewModel ist UI-unabhängig
- ✅ Pure Business Logic
- ✅ Klare Inputs/Outputs
- ✅ Mockable Dependencies

### 5.2 SwiftUI ViewModel Tests

```swift
import XCTest
@testable import MyApp

class LoginViewModelTests: XCTestCase {
    var viewModel: LoginViewModel!
    var mockAuthService: MockAuthService!
    
    override func setUp() {
        super.setUp()
        mockAuthService = MockAuthService()
        viewModel = LoginViewModel(authService: mockAuthService)
    }
    
    func testCanLoginWhenCredentialsValid() {
        // Given
        viewModel.username = "test@example.com"
        viewModel.password = "password123"
        
        // When / Then
        XCTAssertTrue(viewModel.canLogin)
    }
    
    func testCannotLoginWhenUsernameEmpty() {
        // Given
        viewModel.username = ""
        viewModel.password = "password123"
        
        // When / Then
        XCTAssertFalse(viewModel.canLogin)
    }
    
    func testLoginSuccess() async throws {
        // Given
        mockAuthService.shouldSucceed = true
        viewModel.username = "test@example.com"
        viewModel.password = "password123"
        
        // When
        await viewModel.login()
        
        // Then
        XCTAssertTrue(viewModel.isAuthenticated)
        XCTAssertNil(viewModel.errorMessage)
        XCTAssertFalse(viewModel.isLoading)
    }
    
    func testLoginFailure() async throws {
        // Given
        mockAuthService.shouldSucceed = false
        viewModel.username = "test@example.com"
        viewModel.password = "wrongpassword"
        
        // When
        await viewModel.login()
        
        // Then
        XCTAssertFalse(viewModel.isAuthenticated)
        XCTAssertNotNil(viewModel.errorMessage)
        XCTAssertFalse(viewModel.isLoading)
    }
}
```

---

### 5.3 Compose ViewModel Tests

```kotlin
import kotlinx.coroutines.test.*
import org.junit.Before
import org.junit.Test
import org.junit.Assert.*

class LoginViewModelTest {
    private lateinit var viewModel: LoginViewModel
    private lateinit var mockAuthRepository: MockAuthRepository
    
    @Before
    fun setup() {
        mockAuthRepository = MockAuthRepository()
        viewModel = LoginViewModel(mockAuthRepository)
    }
    
    @Test
    fun `canLogin is true when credentials valid`() = runTest {
        // Given
        viewModel.updateUsername("test@example.com")
        viewModel.updatePassword("password123")
        
        // When
        val canLogin = viewModel.canLogin.value
        
        // Then
        assertTrue(canLogin)
    }
    
    @Test
    fun `canLogin is false when username empty`() = runTest {
        // Given
        viewModel.updateUsername("")
        viewModel.updatePassword("password123")
        
        // When
        val canLogin = viewModel.canLogin.value
        
        // Then
        assertFalse(canLogin)
    }
    
    @Test
    fun `login success sets authenticated to true`() = runTest {
        // Given
        mockAuthRepository.shouldSucceed = true
        viewModel.updateUsername("test@example.com")
        viewModel.updatePassword("password123")
        
        // When
        viewModel.login()
        advanceUntilIdle()
        
        // Then
        assertTrue(viewModel.isAuthenticated.value)
        assertNull(viewModel.errorMessage.value)
        assertFalse(viewModel.isLoading.value)
    }
    
    @Test
    fun `login failure sets error message`() = runTest {
        // Given
        mockAuthRepository.shouldSucceed = false
        viewModel.updateUsername("test@example.com")
        viewModel.updatePassword("wrongpassword")
        
        // When
        viewModel.login()
        advanceUntilIdle()
        
        // Then
        assertFalse(viewModel.isAuthenticated.value)
        assertNotNull(viewModel.errorMessage.value)
        assertFalse(viewModel.isLoading.value)
    }
}
```

---

## 6. MVVM VARIATIONS & ALTERNATIVEN

### 6.1 MVVM bleibt der Standard

**MVVM ist weiterhin gültig und empfohlen** für die meisten Apps!

### 6.2 Moderne Alternativen (für Fortgeschrittene)

#### Container Pattern (SwiftUI)
- Dependency Injection auf App-Level
- Alternative für sehr große Apps
- Siehe: [State Management Advanced](State_Management_Advanced.md)

#### The Composable Architecture (TCA)
- Für komplexe State Management
- Predictable State Container
- Steep learning curve
- Optional, nicht im Scope dieses Kurses

#### MVI (Model-View-Intent)
- Für Compose manchmal verwendet
- Stärker auf Immutability fokussiert
- Alternative zu MVVM

```kotlin
// MVI Beispiel (optional)
sealed class LoginIntent {
    data class UpdateUsername(val username: String) : LoginIntent()
    data class UpdatePassword(val password: String) : LoginIntent()
    object Login : LoginIntent()
}

data class LoginState(
    val username: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)
```

**Hinweis:** MVI ist optional. MVVM ist einfacher und reicht für die meisten Fälle!

---

## 7. ZUSAMMENFASSUNG

### MVVM Kern-Prinzipien

1. **Separation of Concerns**
   - Model: Daten
   - ViewModel: Logik + State
   - View: UI

2. **Unidirectional Data Flow**
   - State → View → Action → State

3. **Testbarkeit**
   - ViewModel ist UI-unabhängig
   - Einfach zu mocken

4. **Maintainability**
   - Klare Struktur
   - Leicht zu erweitern

### Wann MVVM verwenden?

✅ **Immer empfohlen für:**
- Neue Apps
- Apps mit komplexer UI-Logik
- Apps die getestet werden sollen
- Teams mit mehreren Entwicklern

❌ **Optional für:**
- Sehr einfache Apps (1-2 Screens)
- Prototypen
- Einmal-Projekte

---

## 8. CHECKLISTE

Ist dein MVVM korrekt implementiert?

**ViewModel:**
- [ ] Keine UI-Referenzen (kein View/Context)
- [ ] Klare State-Definition
- [ ] Immutable State nach außen
- [ ] Testbar ohne UI
- [ ] Single Responsibility

**View:**
- [ ] Nur UI-Code
- [ ] Keine Geschäftslogik
- [ ] Bindet an ViewModel
- [ ] Triggert Actions
- [ ] Zeigt State an

**Model:**
- [ ] Pure Data Structures
- [ ] Keine UI-Dependencies
- [ ] Einfach serialisierbar
- [ ] Wiederverwendbar

---

