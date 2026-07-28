## Jetpack Compose: вопросы и ловушки на собеседовании

---

## Что такое Compose и как работает рекомпозиция

Compose — декларативный UI-фреймворк. Вместо того чтобы изменять существующие View через методы (`setText`, `setVisibility`), ты описываешь как UI **должен выглядеть** при конкретном состоянии, а Compose сам решает что перерисовать.

**Рекомпозиция** — повторный вызов composable-функции при изменении состояния. Compose не перерисовывает всё дерево целиком — только те функции, чьи входные данные изменились. Это называется **умная рекомпозиция (smart recomposition)**.

**Ловушка:** рекомпозиция может происходить очень часто и в любой момент. Composable-функция должна быть **чистой** — без побочных эффектов, без обращения к внешнему состоянию напрямую.

```kotlin
// ПЛОХО — побочный эффект прямо в теле функции
@Composable
fun MyScreen() {
    viewModel.loadData() // вызовется при каждой рекомпозиции!
    Text("Hello")
}

// ХОРОШО — побочный эффект в LaunchedEffect
@Composable
fun MyScreen() {
    LaunchedEffect(Unit) {
        viewModel.loadData() // вызовется один раз
    }
    Text("Hello")
}
```

---

## State и remember

`mutableStateOf` создаёт наблюдаемое состояние — при изменении Compose знает какие функции нужно перекомпозировать. `remember` сохраняет значение между рекомпозициями.

```kotlin
// Без remember — состояние сбрасывается при каждой рекомпозиции
@Composable
fun Counter() {
    var count = mutableStateOf(0) // ПЛОХО — новый объект при каждой рекомпозиции

    var count by remember { mutableStateOf(0) } // ХОРОШО
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

**`rememberSaveable`** — сохраняет состояние ещё и при пересоздании Activity (поворот экрана), аналог `onSaveInstanceState`. Работает автоматически с примитивами и Parcelable; для кастомных типов нужен кастомный `Saver`.

**Ловушка — потеря состояния:**

```kotlin
// count сбросится при повороте экрана
var count by remember { mutableStateOf(0) }

// count переживёт поворот
var count by rememberSaveable { mutableStateOf(0) }
```

---

## State Hoisting

Паттерн выноса состояния наверх. Composable-функция становится **stateless** — получает состояние и колбэк через параметры. Это делает её переиспользуемой и тестируемой.

```kotlin
// Stateful — владеет состоянием, сложнее тестировать
@Composable
fun CounterStateful() {
    var count by remember { mutableStateOf(0) }
    CounterStateless(count) { count++ }
}

// Stateless — только отображает, легко тестировать и переиспользовать
@Composable
fun CounterStateless(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}
```

**Вопрос на собеседовании:** «Зачем поднимать состояние наверх?» — переиспользование, тестируемость, единый источник правды, возможность шарить состояние между несколькими composable.

---

## Side Effects

Побочные эффекты в Compose — операции которые взаимодействуют с внешним миром (сеть, БД, аналитика). Для них есть специальные API:

**`LaunchedEffect(key)`** — запускает корутину. Перезапускается при изменении `key`. При выходе из композиции корутина отменяется.

```kotlin
LaunchedEffect(userId) {
    // перезапустится если userId изменится
    viewModel.loadUser(userId)
}
```

**`SideEffect`** — выполняется после каждой успешной рекомпозиции. Для синхронизации с non-Compose кодом.

```kotlin
SideEffect {
    analytics.setScreen("ProfileScreen") // каждый раз после рекомпозиции
}
```

**`DisposableEffect(key)`** — для эффектов которые нужно очищать (подписки, listener'ы):

```kotlin
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event -> }
    lifecycleOwner.lifecycle.addObserver(observer)

    onDispose {
        lifecycleOwner.lifecycle.removeObserver(observer) // очистка
    }
}
```

**`rememberCoroutineScope`** — получить scope привязанный к composable, для запуска корутин из колбэков (onClick и т.д.):

```kotlin
val scope = rememberCoroutineScope()
Button(onClick = {
    scope.launch { viewModel.save() } // запуск из колбэка
}) { Text("Сохранить") }
```

**Ловушка — неправильный выбор эффекта:**

```kotlin
// ПЛОХО — LaunchedEffect не для синхронной работы с non-Compose кодом
LaunchedEffect(Unit) {
    analytics.log("screen_open") // лучше SideEffect
}

// ПЛОХО — SideEffect для корутин
SideEffect {
    scope.launch { api.loadData() } // лучше LaunchedEffect
}
```

---

## derivedStateOf

Используется когда состояние вычисляется из другого состояния, и рекомпозиция должна происходить только когда изменился **результат**, а не источник.

```kotlin
// ПЛОХО — рекомпозиция при каждом изменении списка
val isButtonEnabled = items.isNotEmpty()

// ХОРОШО — рекомпозиция только когда результат изменился (false→true или true→false)
val isButtonEnabled by remember {
    derivedStateOf { items.isNotEmpty() }
}
```

**Ловушка:** `derivedStateOf` не нужен везде — только когда источник меняется часто, а результат редко. Избыточное использование усложняет код без пользы.

---

## Производительность и стабильность типов

Compose пропускает рекомпозицию если параметры не изменились — но только для **стабильных типов**. Нестабильный тип заставляет Compose перекомпозировать функцию каждый раз.

Стабильны автоматически: примитивы, `String`, `@Stable`/`@Immutable` классы, `data class` только с val-полями стабильных типов.

Нестабильны: классы с `var`-полями, обычные `List`, `Map` (интерфейсы), классы из сторонних библиотек.

```kotlin
// ПЛОХО — List нестабилен, рекомпозиция всегда
@Composable
fun MyList(items: List<Item>) { }

// ХОРОШО — ImmutableList из kotlinx.collections.immutable стабилен
@Composable
fun MyList(items: ImmutableList<Item>) { }

// Или пометить data class как @Immutable если уверен в неизменяемости
@Immutable
data class UiState(val items: List<Item>)
```

---

## CompositionLocal

Механизм неявной передачи данных вниз по дереву без явной передачи через параметры. Аналог React Context.

```kotlin
val LocalUserRole = compositionLocalOf<UserRole> { error("No role provided") }

// Предоставляем значение
CompositionLocalProvider(LocalUserRole provides UserRole.ADMIN) {
    ChildScreen() // и все вложенные имеют доступ
}

// Читаем в любом вложенном composable
@Composable
fun ChildScreen() {
    val role = LocalUserRole.current
}
```

**`compositionLocalOf`** — при изменении перекомпозирует только потребителей. **`staticCompositionLocalOf`** — при изменении перекомпозирует **всё** дерево под провайдером. Использовать только для редко меняющихся значений (тема, навигация).

**Ловушка:** злоупотребление CompositionLocal делает поток данных неявным и код сложнее читать. Использовать только для кросс-секционных данных (тема, локализация, аналитика).

---

## Часто задаваемые вопросы

**«В чём разница между remember и rememberSaveable?»** `remember` — переживает рекомпозицию. `rememberSaveable` — ещё и пересоздание Activity.

**«Когда рекомпозиция не произойдёт?»** Когда все параметры функции стабильны и не изменились — Compose пропустит вызов.

**«Чем Compose лучше View system?»** Меньше кода, нет `findViewById`, декларативный подход, встроенная анимация, легче тестировать, нет проблемы рассинхронизации View и данных.

**«Как правильно работать с Flow в Compose?»** Через `collectAsState()` или `collectAsStateWithLifecycle()` (предпочтительнее — автоматически отписывается когда экран уходит в фон):

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

**«Как избежать лишних рекомпозиций при кликах?»** Оборачивать лямбды в `remember` чтобы ссылка на лямбду не менялась:

```kotlin
// ПЛОХО — новая лямбда при каждой рекомпозиции
Button(onClick = { viewModel.onAction(item.id) })

// ХОРОШО — стабильная ссылка
val onClick = remember(item.id) { { viewModel.onAction(item.id) } }
Button(onClick = onClick)
```

---

## Главные ловушки в сводке

| Ловушка                                         | Проблема                         | Решение                              |
| ----------------------------------------------- | -------------------------------- | ------------------------------------ |
| Побочный эффект в теле функции                  | Вызов при каждой рекомпозиции    | `LaunchedEffect` / `SideEffect`      |
| `mutableStateOf` без `remember`                 | Сброс состояния при рекомпозиции | Обернуть в `remember`                |
| `remember` вместо `rememberSaveable`            | Потеря состояния при повороте    | `rememberSaveable`                   |
| Нестабильные типы в параметрах                  | Лишние рекомпозиции              | `@Immutable`, `ImmutableList`        |
| `derivedStateOf` везде                          | Усложнение кода без пользы       | Только когда источник меняется часто |
| `staticCompositionLocalOf` для частых изменений | Перекомпозиция всего дерева      | `compositionLocalOf`                 |
| Новая лямбда при каждой рекомпозиции            | Лишние рекомпозиции дочерних     | `remember { lambda }`                |