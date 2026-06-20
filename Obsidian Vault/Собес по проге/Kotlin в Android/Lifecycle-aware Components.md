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

Если вам нужно выполнить действие при «рождении» компонента и очистить ресурсы при его «смерти» (аналог `onCreate` и `onDestroy`), используются эффекты.

### `DisposableEffect` (Для очистки ресурсов)

Идеально подходит для регистрации слушателей, датчиков или подписок.

```Kotlin
@Composable
fun SensorScreen(sensorManager: SensorManager) {
    DisposableEffect(Unit) {
        val listener = SensorEventListener { /* ... */ }
        sensorManager.registerListener(listener, ...)

        // Этот блок выполнится, когда компонент исчезнет с экрана (onDispose)
        onDispose {
            sensorManager.unregisterListener(listener)
        }
    }
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
