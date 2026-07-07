# Конспект: Side-эффекты (Effects) в Jetpack Compose

## Зачем нужны эффекты

Compose-функции должны быть **чистыми (side-effect free)** — вызываться много раз, в любом порядке, пропускаться при рекомпозиции (skip), выполняться параллельно на нескольких потоках при отрисовке нескольких composable. Поэтому напрямую внутри `@Composable`-функции **нельзя**:

- запускать корутины
- подписываться/отписываться от внешних источников (слушатели, сенсоры, callback)
- менять внешнее (non-Compose) состояние
- делать сетевые запросы, работу с БД и т.д.

Для всего этого существуют специальные **effect-функции** — они дают контролируемую точку входа для "побочных эффектов", привязанную к жизненному циклу композиции.

---

## LaunchedEffect

Запускает **корутину**, привязанную к композиции. Стартует при входе composable в композицию, отменяется при выходе из композиции или при изменении ключей.

```kotlin
@Composable
fun UserProfile(userId: String) {
    var user by remember { mutableStateOf<User?>(null) }

    LaunchedEffect(userId) {
        user = repository.loadUser(userId) // suspend-функция
    }
    // ...
}
```

- **Ключи (keys)** — при изменении хотя бы одного ключа предыдущая корутина отменяется и запускается новая.
- `LaunchedEffect(Unit)` / `LaunchedEffect(true)` — запустить один раз при входе в композицию (аналог `onCreate`-подобной логики), не перезапускать при рекомпозициях.
- Используется для: загрузки данных при появлении экрана, показа `Snackbar`, анимаций через `Animatable`, однократной навигации (например, `LaunchedEffect(Unit) { navController.navigate(...) }`).

---

## rememberCoroutineScope

Даёт `CoroutineScope`, привязанный к точке композиции, но **не запускает корутину сам** — используется, когда нужно стартовать корутину **из callback'а** (`onClick` и т.п.), а не при входе в композицию.

```kotlin
@Composable
fun MyButton() {
    val scope = rememberCoroutineScope()
    Button(onClick = {
        scope.launch {
            snackbarHostState.showSnackbar("Clicked!")
        }
    }) { Text("Click") }
}
```

**Разница с LaunchedEffect:**

||`LaunchedEffect`|`rememberCoroutineScope`|
|---|---|---|
|Когда запускается|Автоматически при входе в композицию / смене ключей|Вручную, из обработчика события|
|Где используется|Внутри тела composable|Внутри callback (onClick и т.д.)|

---

## DisposableEffect

Как `LaunchedEffect`, но предназначен для эффектов, которые требуют **явной очистки (cleanup)** — например, подписка/отписка на слушатель, регистрация/дерегистрация `BroadcastReceiver`, listener у сенсора и т.д. Обязательно должен заканчиваться блоком `onDispose { }`.

```kotlin
@Composable
fun rememberNetworkState(context: Context): State<Boolean> {
    val state = remember { mutableStateOf(true) }

    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(c: Context, i: Intent) {
                state.value = isConnected(context)
            }
        }
        context.registerReceiver(receiver, IntentFilter(CONNECTIVITY_ACTION))

        onDispose {
            context.unregisterReceiver(receiver) // обязательная очистка
        }
    }
    return state
}
```

- `onDispose` вызывается при выходе composable из композиции **или** при смене ключей (перед перезапуском эффекта).
- Забытый `onDispose` — частая причина утечек памяти в Compose (аналог забытого `unregisterReceiver`/`removeListener` в классическом Android).

---

## SideEffect

Для публикации Compose-состояния во **внешний non-Compose объект** после каждой успешной рекомпозиции. В отличие от `LaunchedEffect`, не корутина, выполняется синхронно.

```kotlin
@Composable
fun Analytics(userId: String) {
    SideEffect {
        analyticsTracker.setUserId(userId) // синхронизация с внешней системой
    }
}
```

Используется редко, обычно для интеграции с внешними (не-Compose) SDK/аналитикой, которым нужно "знать" текущее состояние.

---

## rememberUpdatedState

Решает проблему "устаревшего замыкания" (stale closure) в `LaunchedEffect`, который не должен перезапускаться при смене какого-то значения, но должен видеть его актуальную версию.

```kotlin
@Composable
fun Greeting(onTimeout: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) { // не перезапускается при смене onTimeout
        delay(5000)
        currentOnTimeout() // всегда актуальная версия лямбды
    }
}
```

Без `rememberUpdatedState` при обновлении `onTimeout` из внешнего кода `LaunchedEffect(Unit)` (который не перезапускается, т.к. ключ `Unit` не меняется) продолжил бы держать **старую** ссылку на лямбду.

---

## produceState

Конвертирует не-Compose асинхронный источник данных (callback, Flow, suspend-функцию) в Compose `State`, инкапсулируя корутину/подписку внутри.

```kotlin
@Composable
fun loadUser(id: String): State<User?> = produceState<User?>(initialValue = null, id) {
    value = repository.loadUser(id) // suspend-функция
}
```

По сути — обёртка над `LaunchedEffect` + `mutableStateOf`, удобна для написания собственных "state-producing" composable-функций.

---

## derivedStateOf

**Не про побочные эффекты во внешний мир**, а про оптимизацию рекомпозиции: используется, когда состояние вычисляется из другого состояния, но должно триггерить рекомпозицию **реже**, чем меняется исходное состояние.

```kotlin
val listState = rememberLazyListState()
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
```

Без `derivedStateOf` — `showButton` пересчитывался бы при каждом изменении `firstVisibleItemIndex` (например, на каждый пиксель скролла), а с ним — рекомпозиция произойдёт только когда **изменится сам результат** (`true`/`false`).

---

## snapshotFlow

Конвертирует Compose `State`/`Snapshot`-совместимое значение **в `Flow`**, чтобы можно было использовать операторы Flow (`debounce`, `distinctUntilChanged`, `filter` и т.д.) поверх Compose-состояния.

```kotlin
LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .distinctUntilChanged()
        .collect { index -> /* реакция на изменение */ }
}
```

Обычно используется **внутри** `LaunchedEffect`, а не сам по себе.

---

## Сводная таблица

|Effect|Для чего|Suspend?|Нужна очистка?|
|---|---|---|---|
|`LaunchedEffect`|Запуск корутины при входе в композицию/смене ключей|✅|Отмена автоматическая|
|`rememberCoroutineScope`|Запуск корутины из callback'а|✅|Отмена при выходе из композиции|
|`DisposableEffect`|Подписка/отписка на внешний источник|❌|✅ обязательно `onDispose`|
|`SideEffect`|Публикация состояния во внешний non-Compose объект|❌|❌|
|`rememberUpdatedState`|Актуализация значения внутри "долгоживущего" эффекта|❌|❌|
|`produceState`|Конвертация асинхронного источника в `State`|✅|Как у LaunchedEffect|
|`derivedStateOf`|Оптимизация — вычисляемое состояние, реже триггерящее рекомпозицию|❌|❌|
|`snapshotFlow`|Compose State → Flow|❌ (используется внутри corutine)|Как у корутины|

---

## Частые ошибки

- Использование `LaunchedEffect(Unit)`, когда на самом деле нужно реагировать на смену параметра (например, `userId`) — эффект не перезапустится при смене данных.
- Забытый `onDispose` в `DisposableEffect` — утечки листенеров/ресиверов.
- Обращение к "старому" значению лямбды внутри долгоживущего `LaunchedEffect` без `rememberUpdatedState` (stale closure).
- Использование `remember { derivedStateOf {...} }` без обёртки `remember` — теряется мемоизация, `derivedStateOf` пересоздаётся при каждой рекомпозиции.
- Побочные эффекты (сетевые вызовы, запись в БД) напрямую в теле `@Composable`, а не в `LaunchedEffect`/`produceState` — нарушение принципа "composable должен быть чистым".

## Частые вопросы на собеседовании

- Чем `LaunchedEffect` отличается от `rememberCoroutineScope`?
- Зачем нужен `DisposableEffect` и что будет, если забыть `onDispose`?
- Что такое stale closure и как решает `rememberUpdatedState`?
- Для чего нужен `derivedStateOf`, и чем он отличается от простого `remember`?
- Что произойдёт, если сделать сетевой запрос прямо в теле composable-функции без эффекта?
- Как передать значения из Flow в Compose UI (`collectAsState`/`collectAsStateWithLifecycle`) и в чём разница между ними?