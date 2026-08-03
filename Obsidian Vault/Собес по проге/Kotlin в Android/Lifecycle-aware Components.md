 **В Compose механизмы жизненного цикла работают иначе, но они по-прежнему критически важны.**

В Compose мы не используем `onStart/onStop` напрямую, потому что у функций-компонентов (Composable) нет таких методов. Вместо этого Compose предлагает свои инструменты, которые «подружены» с жизненным циклом Android.

Вот как Lifecycle-aware концепции превращаются в Compose-код:

---

## 1. Сбор данных: `collectAsStateWithLifecycle`

Это самый важный момент. Если в обычном Kotlin вы используете `collect`, то в Compose для связи с `StateFlow`нужно использовать именно этот метод.

- **Зачем?** Он автоматически перестает собирать данные, когда приложение уходит в фон, и возобновляет, когда пользователь возвращается.
    
- **Библиотека:** `androidx.lifecycle:lifecycle-runtime-compose`.

```Kotlin
@Composable
fun MyScreen(viewModel: MyViewModel) {
    // Безопасный сбор данных с учетом жизненного цикла
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // UI автоматически перерисуется при обновлении uiState
}
```

---

## 2. Эффекты: `DisposableEffect` и `LaunchedEffect`

Если вам нужно выполнить действие при «рождении» компонента и очистить ресурсы при его «смерти» (аналог `onCreate` и `onDestroy`), используются эффекты.

---

### LaunchedEffect (Для корутин и одноразовых действий)

Запускает корутину при входе компонента в композицию. Автоматически отменяет её при выходе или при изменении ключа. Это основной способ запускать асинхронный код при появлении экрана.

```kotlin
@Composable
fun UserScreen(userId: String, viewModel: UserViewModel) {

    // Unit как ключ — выполнится один раз при появлении компонента
    LaunchedEffect(Unit) {
        viewModel.loadInitialData()
    }

    // userId как ключ — перезапустится каждый раз когда userId изменится
    // предыдущая корутина будет отменена перед запуском новой
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    // Несколько ключей — перезапускается если изменился хотя бы один
    LaunchedEffect(userId, viewModel) {
        viewModel.loadUser(userId)
    }
}
```

**Показ Snackbar — классический пример:**

```kotlin
@Composable
fun FormScreen(viewModel: FormViewModel) {
    val snackbarHostState = remember { SnackbarHostState() }
    val errorMessage by viewModel.errorMessage.collectAsState()

    // Реагируем на появление ошибки
    LaunchedEffect(errorMessage) {
        errorMessage?.let {
            snackbarHostState.showSnackbar(it) // suspend-функция — только в корутине
            viewModel.clearError()
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) {
        // контент
    }
}
```

**Навигация по одноразовому событию:**

```kotlin
@Composable
fun LoginScreen(viewModel: LoginViewModel, onLoginSuccess: () -> Unit) {
    val effect by viewModel.effect.collectAsState(initial = null)

    LaunchedEffect(effect) {
        when (effect) {
            is LoginEffect.NavigateToHome -> onLoginSuccess()
            else -> Unit
        }
    }
}
```

---

### DisposableEffect (Для очистки ресурсов)

Идеально подходит для регистрации слушателей, датчиков или подписок.

```kotlin
@Composable
fun SensorScreen(sensorManager: SensorManager) {
    DisposableEffect(Unit) {
        val listener = SensorEventListener { /* ... */ }
        sensorManager.registerListener(listener, ...)

        // Этот блок выполнится когда компонент исчезнет с экрана
        onDispose {
            sensorManager.unregisterListener(listener)
        }
    }
}
```

---

### Разница между LaunchedEffect и DisposableEffect

![[Pasted image 20260803190019.png]]

```kotlin
// LaunchedEffect — для suspend
LaunchedEffect(Unit) {
    delay(3000)            // можно
    api.loadData()         // можно — suspend
    flow.collect { }       // можно — suspend
}

// DisposableEffect — для синхронного кода с очисткой
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event -> }
    lifecycleOwner.lifecycle.addObserver(observer) // синхронно

    onDispose {
        lifecycleOwner.lifecycle.removeObserver(observer) // очистка
    }
}
```

**Ловушка — LaunchedEffect не подходит для очистки listener'ов:**

```kotlin
// ПЛОХО — нет гарантии очистки если корутина отменится до unregister
LaunchedEffect(Unit) {
    val listener = MyListener()
    someManager.register(listener)
    // если корутина отменится здесь — unregister не вызовется!
    someManager.unregister(listener)
}

// ХОРОШО — onDispose вызывается гарантированно
DisposableEffect(Unit) {
    val listener = MyListener()
    someManager.register(listener)
    onDispose { someManager.unregister(listener) }
}
```

---

## 3. Отслеживание Lifecycle внутри Compose

Иногда вам всё же нужно знать, когда Activity ушла в `onPause`. Для этого можно создать свой эффект, который слушает жизненный цикл.

```Kotlin
@Composable
fun LifecycleObserverExample() {
    val lifecycleOwner = LocalLifecycleOwner.current

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_PAUSE -> println("Пауза")
                Lifecycle.Event.ON_RESUME -> println("Снова в деле")
                else -> {}
            }
        }

        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

---

## 4. Почему `viewModelScope` все еще нужен?

Даже в Compose вся тяжелая логика (запросы в сеть, БД) остается во **ViewModel**. `viewModelScope` гарантирует, что если пользователь нажал «назад» и экран закрылся, все фоновые запросы отменятся немедленно. Это база, которая не меняется.

---

## Итог: Что добавить в конспект для Compose?

|**Задача**|**Инструмент в Compose**|
|---|---|
|**Сбор Flow**|`collectAsStateWithLifecycle()`|
|**Действие при старте/уходе**|`LaunchedEffect(Unit)` / `DisposableEffect`|
|**Доступ к LifecycleOwner**|`LocalLifecycleOwner.current`|
|**ViewModel**|`hiltViewModel()` (интеграция с Hilt)|

---

### Резюме:

> В Jetpack Compose мы не управляем жизненным циклом вручную. Мы используем **Lifecycle-aware**операторы (`collectAsStateWithLifecycle`), чтобы UI синхронизировался с состоянием приложения. Если мы забудем про жизненный цикл в Compose, приложение будет продолжать качать данные из сети даже в фоновом режиме, тратя заряд батареи.

---
