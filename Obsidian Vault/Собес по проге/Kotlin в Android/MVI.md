## MVI в Android: вопросы и ловушки на собеседовании

---

## Основная идея

MVI строится на трёх принципах:

- **Однонаправленный поток данных** — данные текут только в одну сторону: Intent → State → UI
- **Единственный источник правды** — один объект `State` описывает весь экран целиком
- **Неизменяемость состояния** — State не мутируется, а заменяется новым объектом

```
User Action → Intent → ViewModel → Reducer → State → UI
                ↑__________________________________|
```

---

## Базовая реализация

```kotlin
// Состояние экрана — единый объект
data class ProfileState(
    val isLoading: Boolean = false,
    val user: User? = null,
    val error: String? = null
)

// Намерения пользователя
sealed class ProfileIntent {
    object LoadProfile : ProfileIntent()
    data class UpdateName(val name: String) : ProfileIntent()
    object Logout : ProfileIntent()
}

// Одноразовые события (показ снэкбара, навигация)
sealed class ProfileEffect {
    object NavigateToLogin : ProfileEffect()
    data class ShowError(val message: String) : ProfileEffect()
}
```

```kotlin
class ProfileViewModel : ViewModel() {

    private val _state = MutableStateFlow(ProfileState())
    val state: StateFlow<ProfileState> = _state.asStateFlow()

    // Канал для одноразовых эффектов
    private val _effect = Channel<ProfileEffect>()
    val effect = _effect.receiveAsFlow()

    fun handleIntent(intent: ProfileIntent) {
        when (intent) {
            is ProfileIntent.LoadProfile -> loadProfile()
            is ProfileIntent.UpdateName -> updateName(intent.name)
            is ProfileIntent.Logout -> logout()
        }
    }

    private fun loadProfile() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            try {
                val user = repository.getUser()
                _state.update { it.copy(isLoading = false, user = user) }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }

    private fun logout() {
        viewModelScope.launch {
            repository.logout()
            _effect.send(ProfileEffect.NavigateToLogin)
        }
    }
}
```

```kotlin
// UI — подписывается на state и отправляет intent'ы
@Composable
fun ProfileScreen(viewModel: ProfileViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val context = LocalContext.current

    // Обработка одноразовых эффектов
    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is ProfileEffect.NavigateToLogin -> { /* навигация */ }
                is ProfileEffect.ShowError -> Toast.makeText(context, effect.message, Toast.LENGTH_SHORT).show()
            }
        }
    }

    when {
        state.isLoading -> CircularProgressIndicator()
        state.error != null -> ErrorView(state.error!!)
        state.user != null -> UserProfile(state.user!!)
    }

    Button(onClick = { viewModel.handleIntent(ProfileIntent.LoadProfile) }) {
        Text("Загрузить")
    }
}
```

---

## State vs Effect — главный вопрос на собеседовании

Самая частая тема: **что класть в State, а что в Effect**.

**State** — постоянное состояние экрана. При пересоздании экрана State должен восстановить UI в точности:

- `isLoading`, `error`, `data`
- Текст в полях ввода
- Выбранный элемент списка

**Effect (SideEffect)** — одноразовое событие которое не должно повторяться при пересоздании:

- Показ Toast / Snackbar
- Навигация
- Показ диалога
- Вибрация

**Ловушка — навигация через State:**

```kotlin
// ПЛОХО — при повороте экрана навигация сработает повторно
data class State(val navigateToHome: Boolean = false)

// ХОРОШО — навигация через одноразовый Effect
sealed class Effect {
    object NavigateToHome : Effect()
}
```

---

## Почему Channel для Effect, а не StateFlow

```kotlin
// StateFlow — хранит последнее значение, новый подписчик получит его сразу
// Плохо для эффектов — новый подписчик (после поворота) получит старый эффект
private val _effect = MutableStateFlow<Effect?>(null) // ПЛОХО

// Channel — каждое событие доставляется ровно один раз
private val _effect = Channel<Effect>() // ХОРОШО
val effect = _effect.receiveAsFlow()
```

**SharedFlow** тоже используют для эффектов с `replay = 0` — тогда новые подписчики не получат старые события. Это альтернатива Channel.

---

## Reducer — чистая функция изменения состояния

В строгой реализации MVI изменение состояния выносится в отдельную чистую функцию — Reducer. Принимает текущее состояние и результат операции, возвращает новое состояние:

```kotlin
sealed class ProfileResult {
    object Loading : ProfileResult()
    data class Success(val user: User) : ProfileResult()
    data class Failure(val error: String) : ProfileResult()
}

fun reduce(state: ProfileState, result: ProfileResult): ProfileState {
    return when (result) {
        is ProfileResult.Loading -> state.copy(isLoading = true, error = null)
        is ProfileResult.Success -> state.copy(isLoading = false, user = result.user)
        is ProfileResult.Failure -> state.copy(isLoading = false, error = result.error)
    }
}
```

Чистая функция легко тестируется без моков и корутин:

```kotlin
@Test
fun `loading result sets isLoading true`() {
    val state = reduce(ProfileState(), ProfileResult.Loading)
    assertTrue(state.isLoading)
    assertNull(state.error)
}
```

---

## Частые ловушки

**1. Слишком большой State**

```kotlin
// ПЛОХО — один State на всё приложение
data class AppState(
    val profile: User?,
    val feed: List<Post>,
    val notifications: List<Notification>,
    val settings: Settings,
    // ...
)

// ХОРОШО — отдельный State на каждый экран/фичу
data class FeedState(val posts: List<Post>, val isLoading: Boolean)
data class ProfileState(val user: User?, val isLoading: Boolean)
```

**2. Логика в UI вместо ViewModel**

```kotlin
// ПЛОХО — бизнес-логика в composable
@Composable
fun Screen(viewModel: ViewModel) {
    val state by viewModel.state.collectAsState()
    if (state.user?.role == "admin") { // логика в UI
        AdminPanel()
    }
}

// ХОРОШО — ViewModel решает что показывать
data class State(val showAdminPanel: Boolean)
```

**3. Мутирование State вместо copy**

```kotlin
// ПЛОХО — мутация, Compose/Flow не увидит изменение
_state.value.isLoading = true

// ХОРОШО — новый объект через copy
_state.update { it.copy(isLoading = true) }
```

**4. Intent на каждый символ в поле ввода**

```kotlin
// ПЛОХО — тяжёлая обработка при каждом вводе
onValueChange = { viewModel.handleIntent(SearchIntent.Query(it)) }
// + в ViewModel сразу делает сетевой запрос

// ХОРОШО — debounce в ViewModel
handleIntent(SearchIntent.Query(query))
// внутри:
searchFlow
    .debounce(300)
    .distinctUntilChanged()
    .flatMapLatest { api.search(it) }
```

**5. Не отменять предыдущий запрос**

```kotlin
// ПЛОХО — несколько запросов летят параллельно
private fun search(query: String) {
    viewModelScope.launch { /* запрос */ }
}

// ХОРОШО — отменяет предыдущий
private var searchJob: Job? = null
private fun search(query: String) {
    searchJob?.cancel()
    searchJob = viewModelScope.launch { /* запрос */ }
}
// Или через flatMapLatest в Flow
```

---

## MVI vs MVP vs MVVM

||MVP|MVVM|MVI|
|---|---|---|---|
|Поток данных|Двунаправленный|Двунаправленный|Однонаправленный|
|Состояние|Набор вызовов View|LiveData/StateFlow|Единый объект State|
|Тестируемость|Хорошая|Хорошая|Отличная|
|Предсказуемость|Средняя|Средняя|Высокая|
|Boilerplate|Средний|Низкий|Высокий|
|Отладка|Сложнее|Средне|Легко (весь State виден)|

---

## Вопросы на собеседовании

**«Чем MVI отличается от MVVM?»** В MVVM нет строгого контракта на State — ViewModel может иметь несколько LiveData/Flow. В MVI один объект State описывает весь экран, поток данных строго однонаправленный, Intent явно описывает все действия пользователя.

**«Как обрабатывать ошибки в MVI?»** Ошибка — часть State (`error: String?`). Показ Snackbar/Toast — одноразовый Effect через Channel.

**«Как тестировать MVI?»** Тестировать Reducer как чистую функцию (без корутин, без моков). ViewModel тестировать через `Turbine` — библиотека для тестирования Flow:

```kotlin
@Test
fun `load profile emits loading then success`() = runTest {
    viewModel.state.test {
        viewModel.handleIntent(ProfileIntent.LoadProfile)
        assertEquals(true, awaitItem().isLoading)
        assertEquals(mockUser, awaitItem().user)
    }
}
```