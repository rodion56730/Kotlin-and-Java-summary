**Архитектура** — это не про то, как написать функцию, а про то, **как организовать весь проект**. На Junior+ всегда спрашивают про MVVM и Clean Architecture, потому что без них код превращается в "спагетти", которое невозможно тестировать и поддерживать.

---

# Архитектура: MVVM и Clean Architecture

## 1. Зачем вообще нужна архитектура?

Без архитектуры Activity знает про сеть, БД, бизнес-логику и UI одновременно — это **God Object**. Последствия:

- Невозможно написать Unit-тесты (Activity нельзя запустить без эмулятора).
- Изменение одной части ломает другую.
- Код нельзя переиспользовать.

**Решение**: разделить код на слои с чёткими обязанностями.

---

## 2. MVVM (Model — View — ViewModel)

Это паттерн для слоя **представления (UI)**. Он отвечает только за связь экрана с данными.

```
View (Fragment/Activity/Composable)
    ↕ наблюдает за StateFlow
ViewModel
    ↕ вызывает методы
Model (Repository, данные)
```

**Обязанности каждого:**

- **View** — только рисует UI и отправляет события пользователя во ViewModel. Никакой логики.
- **ViewModel** — держит состояние экрана, обрабатывает события от View, вызывает Repository. Не знает ни о каком `Context`, `View` или `Activity`.
- **Model / Repository** — источник данных. Не знает ничего о UI.

```kotlin
// ViewModel: только логика состояния экрана
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState = _uiState.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            userRepository.getUser(id)
                .onSuccess { _uiState.value = UiState.Success(it) }
                .onFailure { _uiState.value = UiState.Error(it.message) }
        }
    }
}

// View (Compose): только рисует
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    when (state) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> Text((state as UiState.Success).user.name)
        is UiState.Error -> Text("Ошибка")
    }
}
```

---

## 3. Clean Architecture — слои поверх MVVM

MVVM описывает только UI-слой. **Clean Architecture** добавляет остальные слои и правило: **зависимости направлены только внутрь** (к центру).

```
┌─────────────────────────────────┐
│  Presentation (UI + ViewModel)  │  ← знает о Domain
├─────────────────────────────────┤
│  Domain (бизнес-логика)         │  ← не знает ни о чём
├─────────────────────────────────┤
│  Data (сеть, БД, репозитории)   │  ← знает о Domain
└─────────────────────────────────┘
```

**Главное правило**: Domain-слой не зависит ни от Android, ни от Retrofit, ни от Room. Это чистый Kotlin — его можно тестировать без эмулятора.

---

## 4. Domain Layer — сердце архитектуры

### Repository Interface

Domain описывает **контракт** (интерфейс), Data — реализацию. Это называется **Dependency Inversion**.

```kotlin
// Domain слой: только интерфейс
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun saveUser(user: User)
}

// Data слой: реализация
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
    private val dao: UserDao
) : UserRepository {
    override suspend fun getUser(id: String): Result<User> = try {
        val user = api.fetchUser(id).toDomainModel()
        dao.saveUser(user.toEntity())
        Result.success(user)
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

### Use Case (Interactor)

Каждый UseCase — это **одно действие** бизнес-логики. Классически — один метод `invoke`.

```kotlin
// Зачем UseCase если есть Repository?
// Если логика сложнее "просто получить данные" — UseCase её инкапсулирует.
class GetUserWithPostsUseCase @Inject constructor(
    private val userRepository: UserRepository,
    private val postsRepository: PostsRepository
) {
    suspend operator fun invoke(userId: String): Result<UserWithPosts> {
        val user = userRepository.getUser(userId).getOrElse { return Result.failure(it) }
        val posts = postsRepository.getPostsByUser(userId).getOrElse { emptyList() }
        return Result.success(UserWithPosts(user, posts))
    }
}

// Во ViewModel:
class UserViewModel @Inject constructor(
    private val getUserWithPosts: GetUserWithPostsUseCase
) : ViewModel() {
    fun load(id: String) {
        viewModelScope.launch {
            getUserWithPosts(id) // Вызывается как функция через operator invoke
                .onSuccess { ... }
        }
    }
}
```

> **Когда UseCase нужен**: если логика используется в нескольких ViewModel, или если логика сложнее простого CRUD. Для простого `getUser` → `repository.getUser()` — UseCase избыточен.

---

## 5. Data Layer — маппинг моделей

Каждый слой должен иметь **свои модели данных**. Это защищает от изменений: если сервер поменял JSON — меняется только Data-слой.

```kotlin
// Network model (Data layer) — то, что приходит из API
data class UserDto(
    val user_id: String,   // snake_case из JSON
    val full_name: String,
    val avatar_url: String
)

// Domain model — чистая бизнес-сущность
data class User(
    val id: String,
    val name: String,
    val avatarUrl: String
)

// Database model (Data layer) — то, что хранится в Room
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val avatarUrl: String
)

// Маппинг (extension functions)
fun UserDto.toDomainModel() = User(id = user_id, name = full_name, avatarUrl = avatar_url)
fun User.toEntity() = UserEntity(id = id, name = name, avatarUrl = avatarUrl)
fun UserEntity.toDomainModel() = User(id = id, name = name, avatarUrl = avatarUrl)
```

---

## 6. Полная структура проекта

```
app/
├── presentation/          ← UI + ViewModel
│   ├── home/
│   │   ├── HomeFragment.kt
│   │   ├── HomeViewModel.kt
│   │   └── HomeUiState.kt
│   └── detail/
│       ├── DetailFragment.kt
│       └── DetailViewModel.kt
│
├── domain/                ← Бизнес-логика (чистый Kotlin)
│   ├── model/
│   │   └── User.kt
│   ├── repository/
│   │   └── UserRepository.kt  (интерфейс)
│   └── usecase/
│       └── GetUserUseCase.kt
│
└── data/                  ← Сеть + БД + реализации
    ├── remote/
    │   ├── UserApi.kt
    │   └── dto/UserDto.kt
    ├── local/
    │   ├── UserDao.kt
    │   └── entity/UserEntity.kt
    ├── repository/
    │   └── UserRepositoryImpl.kt  (реализация)
    └── di/
        ├── NetworkModule.kt
        └── DatabaseModule.kt
```

---

## 7. UDF — Unidirectional Data Flow

Это принцип, который работает поверх MVVM. Данные текут **только в одну сторону**.

```
User Action (клик)
    ↓
ViewModel.onEvent()
    ↓
Repository / UseCase
    ↓
ViewModel._uiState.value = ...
    ↓
UI перерисовывается
```

**Никогда** UI не меняет данные напрямую. Только через ViewModel.

```kotlin
// Антипаттерн: UI напрямую меняет данные
_uiState.value.items.add(newItem) // ❌ — мутация снаружи ViewModel

// Правильно: UI отправляет событие во ViewModel
viewModel.onAddItem(newItem)      // ✅
// ViewModel сама решает, как изменить состояние
```

---

## Сводная таблица

|**Слой**|**Знает о**|**Не знает о**|**Содержит**|
|---|---|---|---|
|**Presentation**|Domain|Data, Android SDK деталях|ViewModel, Fragment, Composable|
|**Domain**|Ничём|Android, Retrofit, Room|UseCase, Repository интерфейсы, модели|
|**Data**|Domain|Presentation|API, DAO, Repository реализации|

---

## Ключевые фразы для собеседования

- **"Domain-слой не зависит от Android"** — его можно запустить в JVM-тесте без эмулятора.
- **"Dependency Inversion: Domain объявляет интерфейс, Data реализует"** — почему зависимость идёт внутрь.
- **"Каждый слой имеет свои модели"** — маппинг защищает от изменений в API или БД.
- **"UseCase — одно действие бизнес-логики"** — когда нужен и когда избыточен.
- **"UDF: данные текут вниз, события — вверх"** — предсказуемость состояния.
- **"ViewModel не знает о Context, View, Activity"** — иначе утечки памяти и невозможность тестирования.