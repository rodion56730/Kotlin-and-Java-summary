
## Что это такое

TEA — архитектурный паттерн из языка Elm (функциональный язык для фронтенда). В Android его реализацией по сути является **MVI** — они основаны на одних и тех же принципах. Понимание TEA помогает глубже понять откуда взялся MVI и почему он устроен именно так.

---

## Три составляющих

```
Model — состояние приложения
View  — отображение состояния
Update — функция изменения состояния
```

Цикл работы:

```
User Action → Message → Update(Model, Message) → новый Model → View(Model)
      ↑______________________________________________|
```

Однонаправленный поток, неизменяемое состояние, чистые функции — те же идеи что в MVI.

---

## Соответствие TEA → MVI → Android

|TEA|MVI|Android реализация|
|---|---|---|
|Model|State|`data class UiState`|
|Message|Intent|`sealed class Intent`|
|Update|Reducer|`fun reduce(state, intent)`|
|View|UI|Composable / Fragment|
|Command|Effect/SideEffect|`sealed class Effect`|

---

## Полная реализация TEA на Kotlin

### Основные типы

```kotlin
// Model — полное состояние экрана, неизменяемый
data class CounterModel(
    val count: Int = 0,
    val isLoading: Boolean = false,
    val error: String? = null
)

// Message — все возможные события (от пользователя или системы)
sealed class CounterMessage {
    object Increment : CounterMessage()
    object Decrement : CounterMessage()
    object Reset : CounterMessage()
    object LoadFromServer : CounterMessage()
    data class Loaded(val value: Int) : CounterMessage()
    data class LoadFailed(val error: String) : CounterMessage()
}

// Command — побочные эффекты (сеть, навигация, показ диалога)
// В отличие от MVI где Effect одноразовый — Command явно описывает что нужно сделать
sealed class CounterCommand {
    object FetchCounterValue : CounterCommand()
    data class ShowToast(val message: String) : CounterCommand()
    object NavigateBack : CounterCommand()
}
```

### Update — чистая функция (сердце TEA)

```kotlin
// Update принимает текущий Model и Message
// Возвращает новый Model + список Command которые нужно выполнить
data class UpdateResult(
    val model: CounterModel,
    val commands: List<CounterCommand> = emptyList()
)

fun update(model: CounterModel, message: CounterMessage): UpdateResult {
    return when (message) {

        is CounterMessage.Increment -> UpdateResult(
            model = model.copy(count = model.count + 1)
            // нет команд — просто меняем состояние
        )

        is CounterMessage.Decrement -> UpdateResult(
            model = model.copy(
                count = (model.count - 1).coerceAtLeast(0) // не меньше 0
            )
        )

        is CounterMessage.Reset -> UpdateResult(
            model = model.copy(count = 0),
            commands = listOf(CounterCommand.ShowToast("Сброшено"))
        )

        is CounterMessage.LoadFromServer -> UpdateResult(
            model = model.copy(isLoading = true, error = null),
            commands = listOf(CounterCommand.FetchCounterValue) // говорим runtime что нужно сделать
        )

        is CounterMessage.Loaded -> UpdateResult(
            model = model.copy(isLoading = false, count = message.value)
        )

        is CounterMessage.LoadFailed -> UpdateResult(
            model = model.copy(isLoading = false, error = message.error),
            commands = listOf(CounterCommand.ShowToast("Ошибка: ${message.error}"))
        )
    }
}
```

### Runtime — исполнитель команд

Ключевое отличие TEA от простого MVI — **явный Runtime** который исполняет команды и возвращает результат обратно как Message:

```kotlin
class TeaRuntime(
    private val api: CounterApi,
    private val dispatch: (CounterMessage) -> Unit // канал для возврата результата
) {
    fun execute(command: CounterCommand) {
        when (command) {
            is CounterCommand.FetchCounterValue -> {
                CoroutineScope(Dispatchers.IO).launch {
                    try {
                        val value = api.getCounter()
                        // результат возвращается как Message обратно в Update
                        dispatch(CounterMessage.Loaded(value))
                    } catch (e: Exception) {
                        dispatch(CounterMessage.LoadFailed(e.message ?: "Unknown error"))
                    }
                }
            }
            is CounterCommand.ShowToast -> {
                // выполняется платформой
            }
            is CounterCommand.NavigateBack -> {
                // выполняется платформой
            }
        }
    }
}
```

### ViewModel как TEA-хост

```kotlin
class CounterViewModel(private val api: CounterApi) : ViewModel() {

    private val _model = MutableStateFlow(CounterModel())
    val model: StateFlow<CounterModel> = _model.asStateFlow()

    private val _commands = MutableSharedFlow<CounterCommand>(extraBufferCapacity = 10)
    val commands: SharedFlow<CounterCommand> = _commands.asSharedFlow()

    private val runtime = TeaRuntime(api) { message ->
        // Runtime возвращает Message — прогоняем через Update
        dispatch(message)
    }

    // Единственная точка входа — все события через dispatch
    fun dispatch(message: CounterMessage) {
        val current = _model.value
        val result = update(current, message) // чистая функция

        _model.value = result.model

        // Передаём команды в Runtime на исполнение
        result.commands.forEach { command ->
            viewModelScope.launch {
                _commands.emit(command)
            }
            runtime.execute(command)
        }
    }
}
```

### UI — Compose

```kotlin
@Composable
fun CounterScreen(viewModel: CounterViewModel) {
    val model by viewModel.model.collectAsStateWithLifecycle()
    val context = LocalContext.current

    // Обрабатываем команды платформы
    LaunchedEffect(Unit) {
        viewModel.commands.collect { command ->
            when (command) {
                is CounterCommand.ShowToast ->
                    Toast.makeText(context, command.message, Toast.LENGTH_SHORT).show()
                is CounterCommand.NavigateBack -> {
                    // навигация
                }
                else -> Unit
            }
        }
    }

    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        if (model.isLoading) {
            CircularProgressIndicator()
        }

        model.error?.let {
            Text(text = it, color = Color.Red)
        }

        Text(
            text = "${model.count}",
            style = MaterialTheme.typography.displayLarge
        )

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { viewModel.dispatch(CounterMessage.Decrement) }) {
                Text("-")
            }
            Button(onClick = { viewModel.dispatch(CounterMessage.Increment) }) {
                Text("+")
            }
        }

        OutlinedButton(onClick = { viewModel.dispatch(CounterMessage.Reset) }) {
            Text("Сбросить")
        }

        OutlinedButton(onClick = { viewModel.dispatch(CounterMessage.LoadFromServer) }) {
            Text("Загрузить с сервера")
        }
    }
}
```

---

## Главное отличие TEA от MVI

В MVI команды (эффекты) обычно просто отправляются в Channel и UI сам решает что делать. В TEA есть явный **Runtime** — отдельный слой который исполняет команды и возвращает результат обратно как Message в цикл:

```
MVI:  Intent → ViewModel → State + Effect → UI (обрабатывает Effect сам)

TEA:  Message → Update → Model + Command
                              ↓
                           Runtime (исполняет Command)
                              ↓
                           Message (возвращает результат)
                              ↓
                           Update (снова)
```

Это делает TEA полностью **тестируемым без UI** — достаточно тестировать функцию `update` и `runtime` по отдельности.

---

## Тестирование — главное преимущество

```kotlin
class CounterUpdateTest {

    @Test
    fun `increment increases count by 1`() {
        val model = CounterModel(count = 5)
        val result = update(model, CounterMessage.Increment)

        assertEquals(6, result.model.count)
        assertTrue(result.commands.isEmpty())
    }

    @Test
    fun `loadFromServer sets isLoading and returns fetch command`() {
        val model = CounterModel()
        val result = update(model, CounterMessage.LoadFromServer)

        assertTrue(result.model.isLoading)
        assertEquals(listOf(CounterCommand.FetchCounterValue), result.commands)
    }

    @Test
    fun `loaded message updates count and stops loading`() {
        val model = CounterModel(isLoading = true)
        val result = update(model, CounterMessage.Loaded(42))

        assertFalse(result.model.isLoading)
        assertEquals(42, result.model.count)
    }

    @Test
    fun `reset returns show toast command`() {
        val model = CounterModel(count = 10)
        val result = update(model, CounterMessage.Reset)

        assertEquals(0, result.model.count)
        assertTrue(result.commands.any { it is CounterCommand.ShowToast })
    }
}
```

Никаких моков, никаких корутин, никакого Android — просто вход и выход чистой функции.

---

## Коротко о главном

TEA и MVI — одна идея в разных обёртках. TEA чуть строже: явный Runtime, команды всегда возвращают результат как Message, всё через одну функцию `update`. MVI в Android — практическая адаптация TEA под ViewModel, Flow и Compose с чуть меньшей строгостью но той же философией: **один поток данных, одно состояние, предсказуемое поведение**.