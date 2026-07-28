## Clean Architecture в Android

---

## Основная идея

Clean Architecture — подход к организации кода, при котором приложение делится на независимые слои. Главное правило — **Dependency Rule**: зависимости направлены только внутрь. Внутренние слои ничего не знают о внешних.

```
┌─────────────────────────────┐
│         Presentation        │  ← знает о Domain
│  ┌───────────────────────┐  │
│  │        Domain         │  │  ← не знает ни о чём
│  │  ┌─────────────────┐  │  │
│  │  │      Data       │  │  │  ← знает о Domain
│  │  └─────────────────┘  │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

---

## Три слоя

### Domain — ядро приложения

Не зависит ни от Android, ни от фреймворков. Чистый Kotlin. Содержит:

- **Entity** — бизнес-модели (не data-классы для сети или БД, а именно бизнес-сущности)
- **Repository Interface** — контракт, как получать данные (реализация в Data-слое)
- **UseCase (Interactor)** — одна бизнес-операция

```kotlin
// Entity
data class User(
    val id: String,
    val name: String,
    val email: String
)

// Repository Interface — только контракт, без реализации
interface UserRepository {
    suspend fun getUser(id: String): User
    suspend fun updateUser(user: User)
}

// UseCase — одна операция, одна ответственность
class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(userId: String): User {
        return repository.getUser(userId)
    }
}

class UpdateUserNameUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(userId: String, name: String) {
        val user = repository.getUser(userId)
        repository.updateUser(user.copy(name = name))
    }
}
```

### Data — реализация получения данных

Зависит от Domain (реализует его интерфейсы). Содержит:

- **Repository Implementation** — реализует интерфейс из Domain
- **Remote DataSource** — работа с сетью (Retrofit)
- **Local DataSource** — работа с БД (Room)
- **DTO/Mapper** — конвертация между сетевыми/DB моделями и Domain-моделями

```kotlin
// DTO — модель для сети
data class UserDto(
    @SerializedName("id") val id: String,
    @SerializedName("full_name") val fullName: String,
    @SerializedName("email") val email: String
)

// Mapper — конвертация DTO → Domain Entity
fun UserDto.toDomain() = User(
    id = id,
    name = fullName,
    email = email
)

// Repository Implementation
class UserRepositoryImpl(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) : UserRepository {

    override suspend fun getUser(id: String): User {
        return try {
            val dto = remoteDataSource.getUser(id)
            localDataSource.saveUser(dto)
            dto.toDomain()
        } catch (e: IOException) {
            // сеть недоступна — берём из кеша
            localDataSource.getUser(id).toDomain()
        }
    }
}
```

### Presentation — UI и ViewModel

Зависит от Domain (вызывает UseCase). Не знает о Data-слое напрямую.

```kotlin
class ProfileViewModel(
    private val getUserUseCase: GetUserUseCase,
    private val updateUserNameUseCase: UpdateUserNameUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(ProfileState())
    val state = _state.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val user = getUserUseCase(userId) // вызов UseCase, не Repository
                _state.update { it.copy(user = user, isLoading = false) }
            } catch (e: Exception) {
                _state.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }
}
```

---

## Структура пакетов

Два популярных подхода:

**По слоям (layer-first):**

```
app/
├── data/
│   ├── remote/
│   ├── local/
│   └── repository/
├── domain/
│   ├── model/
│   ├── repository/
│   └── usecase/
└── presentation/
    ├── profile/
    └── feed/
```

**По фичам (feature-first) — предпочтительнее для больших проектов:**

```
app/
├── feature_profile/
│   ├── data/
│   ├── domain/
│   └── presentation/
├── feature_feed/
│   ├── data/
│   ├── domain/
│   └── presentation/
└── core/
    ├── network/
    └── database/
```

---

## Частые ловушки

**1. UseCase на каждый геттер — избыточно**

```kotlin
// ПЛОХО — бессмысленный UseCase без логики
class GetUsersUseCase(private val repository: UserRepository) {
    suspend operator fun invoke() = repository.getUsers() // просто делегирует
}

// Такой UseCase не нужен — ViewModel может обращаться к Repository напрямую
// UseCase оправдан когда есть реальная бизнес-логика
```

**2. Android-зависимости в Domain**

```kotlin
// ПЛОХО — Domain знает об Android
class GetUserUseCase(val context: Context) { } // нарушение Dependency Rule

// Domain — чистый Kotlin, без Context, без Android SDK
```

**3. Прямое использование DTO в UI**

```kotlin
// ПЛОХО — UI зависит от сетевой модели
_state.update { it.copy(user = userDto) } // UserDto в Presentation

// ХОРОШО — маппинг в Data-слое, UI работает с Domain Entity
_state.update { it.copy(user = userDto.toDomain()) }
```

**4. Repository знает о UseCase**

```kotlin
// ПЛОХО — обратная зависимость
class UserRepository(private val useCase: SomeUseCase) // нарушение Dependency Rule
```

**5. Бизнес-логика в Repository**

```kotlin
// ПЛОХО — логика принадлежит UseCase, не Repository
class UserRepositoryImpl : UserRepository {
    override suspend fun getUser(id: String): User {
        val user = remote.getUser(id)
        if (user.age < 18) throw IllegalStateException("Too young") // бизнес-логика!
    }
}

// ХОРОШО — в UseCase
class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(id: String): User {
        val user = repository.getUser(id)
        if (user.age < 18) throw IllegalStateException("Too young")
        return user
    }
}
```

---

## Вопросы на собеседовании

**«Зачем UseCase если можно вызвать Repository из ViewModel?»** UseCase инкапсулирует бизнес-логику, которую можно переиспользовать в нескольких ViewModel. Тестируется независимо от UI и Android. Если логика тривиальна — UseCase не нужен.

**«Почему Domain не зависит от Data?»** Чтобы можно было подменить реализацию данных (сеть, БД, mock) без изменения бизнес-логики. Достигается через интерфейс Repository в Domain и его реализацию в Data.

**«Сколько моделей нужно?»** Минимум две: DTO (сеть/БД) и Domain Entity. Иногда три — добавляется UI-модель если Entity не подходит для отображения напрямую.

---
