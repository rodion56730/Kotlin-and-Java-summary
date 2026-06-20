Jetpack Compose — это современный декларативный фреймворк для создания UI в Android. Если раньше мы писали UI в XML (императивно: "найди кнопку и поменяй ей текст"), то в Compose мы просто описываем, как интерфейс должен выглядеть в зависимости от текущих данных.

---

# Jetpack Compose: Основы декларативного UI

## 1. Главная концепция: Декларативность

Вы описываете интерфейс как функцию от состояния. Если состояние (данные) меняется, Compose автоматически перерисовывает нужные части экрана. Этот процесс называется **Рекомпозиция (Recomposition)**.

- **Аннотация `@Composable`**: Ставится перед функцией. Говорит компилятору, что эта функция превращает данные в элементы интерфейса.
    
- **Чистые функции**: Компоненты должны быть быстрыми и не иметь побочных эффектов (не менять глобальные переменные).
    

---

## 2. Состояние (State) и Remember

Чтобы Compose "заметил" изменение данных и перерисовал экран, нужно использовать специальные обертки.

- **`MutableState`**: Объект, который хранит значение. Когда значение меняется, срабатывает рекомпозиция.
    
- **`remember`**: Позволяет функции "запомнить" значение при перерисовке. Без него переменная будет сбрасываться в начальное значение при каждой рекомпозиции.
    
- **`rememberSaveable`**: То же самое, что `remember`, но переживает **пересоздание Activity** (поворот экрана, смена темы). Обычный `remember` — нет.
    

Разберём разницу на двух отдельных примерах, чтобы было понятно, что именно "переживает", а что нет.

**`remember` — сбросится при повороте экрана:**

```kotlin
@Composable
fun CounterWithRemember() {
    // При повороте телефона count обнулится до 0
    var count by remember { mutableStateOf(0) }

    Column {
        Text("Счётчик: $count")
        Button(onClick = { count++ }) {
            Text("Нажать")
        }
    }
}
```

**`rememberSaveable` — переживёт поворот экрана:**

```kotlin
@Composable
fun CounterWithSaveable() {
    // При повороте телефона count сохранится
    var count by rememberSaveable { mutableStateOf(0) }

    Column {
        Text("Счётчик: $count")
        Button(onClick = { count++ }) {
            Text("Нажать")
        }
    }
}
```

> **Почему так?** При повороте экрана Android **пересоздаёт Activity** — всё дерево Composable-функций пересобирается с нуля. `remember` при этом теряет значение (живёт только пока живёт Composition). `rememberSaveable` под капотом сохраняет значение в **Bundle** — тот же механизм, что `onSaveInstanceState` в XML-мире — и восстанавливает его после пересоздания.

> **Частый вопрос на собесе**: "Чем `remember` отличается от `rememberSaveable`?" — `rememberSaveable` сохраняет в Bundle, переживает пересоздание Activity. `remember` — только рекомпозицию.

---

## 3. Основные компоненты (Layouts)

В Compose нет `LinearLayout` или `ConstraintLayout` в привычном виде. Используются простые контейнеры:

- **`Column`**: Располагает элементы вертикально (один под другим).
    
- **`Row`**: Располагает элементы горизонтально (в ряд).
    
- **`Box`**: Накладывает элементы друг на друга (Z-слои).
    
- **`LazyColumn`**: Аналог `RecyclerView`. Используется для списков. Рисует только то, что видно на экране.
    
- **`LazyRow`**: Горизонтальный аналог `LazyColumn`.
    

> **Ключевое**: `LazyColumn` vs `Column` — `Column` создаёт **все** элементы сразу, `LazyColumn` — только видимые. Для больших списков всегда `LazyColumn`.

---

## 4. Модификаторы (Modifiers)

Это универсальный инструмент для настройки компонентов. С их помощью задаются размеры, отступы, клики, фон и анимации.

**Важно:** Порядок модификаторов имеет значение!

```kotlin
// Разный результат:
Modifier.padding(16.dp).background(Color.Blue) // фон НЕ включает отступ
Modifier.background(Color.Blue).padding(16.dp) // фон включает отступ (больше закрашено)
```

---

## 5. State Hoisting (Вынос состояния вверх)

Чтобы сделать компоненты переиспользуемыми и тестируемыми, состояние обычно передается "сверху вниз", а события — "снизу вверх".

1. **State goes down**: Родитель передает значение дочернему компоненту.
2. **Events go up**: Дочерний компонент вызывает callback, чтобы родитель изменил состояние.

```kotlin
// Плохо: состояние внутри — компонент не переиспользовать
@Composable
fun BadInput() {
    var text by remember { mutableStateOf("") }
    TextField(value = text, onValueChange = { text = it })
}

// Хорошо: состояние снаружи — компонент управляемый и тестируемый
@Composable
fun GoodInput(value: String, onValueChange: (String) -> Unit) {
    TextField(value = value, onValueChange = onValueChange)
}
```

---

## 6. Side Effects — эффекты вне Compose ⚠️

Это тема, которую часто спрашивают. `@Composable`-функция должна быть чистой, но иногда нужно запустить что-то "снаружи" — запрос в сеть, таймер, подписка на поток.

- **`LaunchedEffect(key)`**: Запускает корутину при первой композиции или при изменении `key`. Отменяется, когда компонент уходит из композиции.

```kotlin
@Composable
fun Screen(userId: String) {
    LaunchedEffect(userId) {
        // Перезапустится при каждом изменении userId
        viewModel.loadUser(userId)
    }
}
```

- **`DisposableEffect(key)`**: Для подписок, которые нужно **отменить**. Есть блок `onDispose {}`.

```kotlin
@Composable
fun LocationTracker() {
    DisposableEffect(Unit) {
        val listener = startListening()
        onDispose {
            listener.stop() // Вызовется при уходе из экрана
        }
    }
}
```

- **`SideEffect`**: Выполняется при **каждой** успешной рекомпозиции. Для синхронизации с не-Compose кодом.
    
- **`rememberCoroutineScope`**: Даёт `CoroutineScope`, привязанный к композиции. Нужен, когда корутину надо запустить по событию (например, по нажатию кнопки), а не сразу.
    

```kotlin
@Composable
fun MyScreen() {
    val scope = rememberCoroutineScope()
    Button(onClick = {
        scope.launch { /* запрос по нажатию */ }
    }) { Text("Загрузить") }
}
```

---

## 7. Жизненный цикл Composable

Composable проходит три стадии:

1. **Enter Composition** — функция вызывается впервые, элемент появляется на экране.
2. **Recomposition** — функция вызывается повторно при изменении `State`. Может происходить много раз.
3. **Leave Composition** — элемент убирается с экрана, ресурсы освобождаются.

> **Важно**: Compose может **пропускать** рекомпозицию (`skip`), если все входные параметры не изменились. Это называется **Smart Recomposition**.

---

## 8. Производительность: как не сломать рекомпозицию

- **Не создавай лямбды и объекты прямо в теле `@Composable`** — они пересоздаются каждый раз и ломают Smart Recomposition.

```kotlin
// Плохо: новый объект при каждой рекомпозиции
LazyColumn {
    items(list, key = { it.id }) { item ->
        ItemCard(item, onClick = { doSomething(item) }) // лямбда пересоздаётся
    }
}

// Лучше: вынести onClick или использовать remember
```

- **`key` в `LazyColumn`**: Если передать `key = { it.id }`, Compose сможет отслеживать элементы при перестановке и не перерисовывать их зря.
    
- **`@Stable` / `@Immutable`**: Аннотации для классов данных. Подсказывают компилятору, что объект не изменится — можно пропустить рекомпозицию.
    

---

## 9. Пример сложного компонента списка

```kotlin
@Composable
fun UserList(names: List<String>) {
    LazyColumn(modifier = Modifier.fillMaxSize()) {
        items(names) { name ->
            UserCard(name = name)
        }
    }
}

@Composable
fun UserCard(name: String) {
    var isSelected by remember { mutableStateOf(false) }

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp)
            .clickable { isSelected = !isSelected },
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text(
                text = name,
                modifier = Modifier.padding(16.dp),
                color = if (isSelected) Color.Red else Color.Black
            )
        }
    }
}
```

---

## Сводная таблица терминов

|**Термин**|**Что это?**|
|---|---|
|**Recomposition**|Процесс обновления UI при изменении State.|
|**Smart Recomposition**|Compose пропускает функции, чьи параметры не изменились.|
|**remember**|Сохраняет значение между рекомпозициями.|
|**rememberSaveable**|Как `remember`, но переживает поворот экрана.|
|**State Hoisting**|Вынос состояния вверх для переиспользуемости.|
|**LaunchedEffect**|Корутина, привязанная к жизни composable.|
|**DisposableEffect**|Эффект с блоком `onDispose` для очистки ресурсов.|
|**CompositionLocal**|Передача данных через дерево без явных параметров (темы, локаль).|
|**Modifier**|Цепочка инструкций по внешнему виду и поведению.|

---

### Ключевые фразы для собеседования

- **"UI — это функция от State"** — главная идея Compose.
- **`remember` vs `rememberSaveable`** — второй переживает смерть Activity (Bundle).
- **Side effects** нельзя делать прямо в теле `@Composable` — только через `LaunchedEffect` и компанию.
- **Порядок модификаторов** важен — `padding` до и после `background` дают разный результат.
- **`LazyColumn` ≠ `Column`** — первый ленивый, второй создаёт все элементы сразу.

---

## Словарик Android-терминов

Слова, которые встречаются в объяснениях, но сами по себе могут быть непонятны.

---

### Bundle

Контейнер для передачи небольших данных между компонентами Android (Activity, Fragment). Работает как словарь: ключ строка — значение любого примитивного типа.

```kotlin
// Так данные кладут в Bundle (например, при передаче в Fragment)
val bundle = Bundle()
bundle.putString("user_name", "Алексей")
bundle.putInt("user_age", 25)

// Так достают обратно
val name = bundle.getString("user_name") // "Алексей"
val age = bundle.getInt("user_age")      // 25
```

> В контексте `rememberSaveable`: Android сохраняет Bundle при повороте экрана через `onSaveInstanceState`. `rememberSaveable` использует этот же механизм "под капотом", поэтому значение переживает пересоздание Activity.

---

### Activity

Один "экран" Android-приложения. Точка входа, которую запускает система. Управляет жизненным циклом экрана: создание, пауза, возобновление, уничтожение.

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { MyApp() } // Здесь запускается Compose
    }
}
```

> **Важно**: При **повороте экрана** Android **уничтожает и пересоздаёт Activity** заново. Это причина, почему `remember` теряет данные — всё дерево Compose пересобирается с нуля.

---

### Корутина (Coroutine)

Способ писать асинхронный код в Kotlin без колбэков. Можно думать о ней как о "лёгком потоке" — она выполняет долгие операции (сеть, база данных), не блокируя основной поток UI.

```kotlin
// Без корутины — заблокирует UI, приложение "зависнет"
fun loadData() {
    val result = api.fetchUser() // долгий запрос в сеть
    showResult(result)
}

// С корутиной — UI не блокируется
fun loadData() {
    viewModelScope.launch {       // запускаем корутину
        val result = api.fetchUser() // выполнится в фоне
        showResult(result)           // вернётся на главный поток
    }
}
```

> В контексте Compose: `LaunchedEffect` — это и есть запуск корутины, привязанной к жизни composable.

---

### CoroutineScope

"Владелец" корутин. Когда scope уничтожается — все его корутины автоматически отменяются. Защищает от утечек памяти.

```kotlin
// viewModelScope — живёт пока жива ViewModel
viewModelScope.launch { /* отменится при очистке ViewModel */ }

// lifecycleScope — живёт пока жив экран
lifecycleScope.launch { /* отменится при destroy экрана */ }

// rememberCoroutineScope в Compose — живёт пока composable в дереве
val scope = rememberCoroutineScope()
scope.launch { /* отменится при уходе composable из экрана */ }
```

---

### Callback (Обратный вызов)

Функция, которую ты передаёшь кому-то, чтобы он вызвал её позже — когда что-то произойдёт.

```kotlin
// onClick — это callback: ты говоришь кнопке "когда нажмут, вызови вот это"
Button(onClick = { /* этот код вызовется при нажатии */ }) {
    Text("Нажми")
}

// В State Hoisting родитель передаёт callback дочернему компоненту:
@Composable
fun Parent() {
    var text by remember { mutableStateOf("") }
    Child(
        value = text,
        onValueChange = { newValue -> text = newValue } // callback
    )
}
```

---

### Composition / Recomposition / Decomposition

Три стадии жизни composable-функции:

```kotlin
@Composable
fun MyComponent() {
    // 1. COMPOSITION — функция вызвана впервые, элемент появился на экране

    var count by remember { mutableStateOf(0) }

    // 2. RECOMPOSITION — count изменился, функция вызвана повторно
    //    Compose обновит только то, что зависит от count

    Button(onClick = { count++ }) { Text("$count") }

    // 3. DECOMPOSITION — компонент ушёл с экрана, ресурсы освобождены
    //    LaunchedEffect и DisposableEffect получат сигнал на отмену
}
```

---

### Inflate (Надуть)

Термин из XML-мира: процесс чтения `.xml`-файла разметки и создания реальных объектов `View` в памяти.

```kotlin
// Явный inflate через ViewBinding
val binding = ActivityMainBinding.inflate(layoutInflater)
// Теперь binding.root — это живой View-объект, готовый к отображению
```

> В Compose аналога нет — там нет отдельного "надувания". Composable-функции сразу описывают UI и рисуют его.

---

### Lambda (Лямбда)

Функция без имени, которую можно передать как аргумент. В Kotlin используется везде — клики, колбэки, трансформации.

```kotlin
// Обычная функция
fun double(x: Int): Int = x * 2

// Та же функция как лямбда
val double = { x: Int -> x * 2 }

// Лямбда как аргумент (onClick принимает лямбду)
Button(onClick = { count++ }) { Text("Нажать") }
//              ^^^^^^^^^^^^
//              это и есть лямбда — функция без имени, переданная аргументом
```

---

### State (Состояние)

Любые данные, от которых зависит внешний вид UI. Если State изменился — UI должен обновиться.

```kotlin
// State — это обёртка над данными
var isLoading by remember { mutableStateOf(false) }
var userName by remember { mutableStateOf("") }
var errorMessage by remember { mutableStateOf<String?>(null) }

// UI "реагирует" на State:
if (isLoading) {
    CircularProgressIndicator()
} else {
    Text(userName)
}
```