## 1. Что такое поток

Поток — это единица выполнения внутри процесса. У каждого Android-приложения есть минимум один поток — **Main Thread** (он же UI Thread).
![[Pasted image 20260606140226.png]]

## 2. Main Thread — почему он особенный

В Android всё UI работает в одном потоке. Это жёсткое правило системы.

```kotlin
// Если делаешь сеть на Main Thread — получишь NetworkOnMainThreadException
// Если делаешь тяжёлые вычисления — UI зависает (ANR через 5 секунд)

// ANR = Application Not Responding
// система убивает приложение если Main Thread заблокирован > 5 сек
```

Почему один поток для UI? Потому что Android View не потокобезопасны. Если два потока одновременно трогают один `TextView` — гарантированный краш.

---

## 3. Создание потока вручную — Thread

```kotlin
// Самый базовый способ
val thread = Thread {
    // этот код выполняется в новом потоке
    val result = doHeavyWork()

    // Хочешь обновить UI — нужно вернуться на Main Thread
    runOnUiThread {
        textView.text = result
    }
}
thread.start()
```

Через `Runnable` явно:

```kotlin
val runnable = Runnable {
    println("Выполняется в потоке: ${Thread.currentThread().name}")
}

val thread = Thread(runnable, "my-background-thread")
thread.start()
```

Получить имя текущего потока — полезно для дебага:

```kotlin
Thread.currentThread().name  // "main", "Thread-2", и т.д.
```

---

## 4. Проблемы голого Thread

```kotlin
class MyActivity : AppCompatActivity() {

    override fun onCreate(...) {
        val thread = Thread {
            while (true) {
                Thread.sleep(1000)
                // делаем что-то...
            }
        }
        thread.start()

        // Activity уничтожается — поток продолжает жить
        // держит ссылку на Activity — memory leak
    }
}
```

Три главные проблемы:

- Нет привязки к lifecycle — поток живёт после уничтожения экрана
- Нельзя легко отменить
- Вернуться на UI Thread нужно вручную через `runOnUiThread` или `Handler`

---

## 5. Handler + Looper — как устроен Main Thread изнутри

Это важно понимать, потому что корутины работают поверх этого же механизма.

```kotlin
// Main Thread изнутри выглядит примерно так:
// Looper — это бесконечный цикл, который читает очередь сообщений
// Handler — отправляет сообщения в эту очередь

val handler = Handler(Looper.getMainLooper())

// Запустить что-то на Main Thread из любого потока:
handler.post {
    textView.text = "Обновлено с фонового потока"
}

// С задержкой:
handler.postDelayed({
    showToast("Через 2 секунды")
}, 2000)
```

Именно так `Dispatchers.Main` в корутинах переключается на UI Thread — он использует `Handler(Looper.getMainLooper())` под капотом.

---

## 6. ThreadPoolExecutor — пул потоков

Создавать новый `Thread` на каждый запрос дорого. Пул переиспользует потоки:

```kotlin
// Фиксированный пул — 4 потока
val executor = Executors.newFixedThreadPool(4)

executor.execute {
    val result = api.loadData()
    handler.post { updateUI(result) }
}

// Не забыть завершить когда не нужен
executor.shutdown()
```

`Dispatchers.IO` в корутинах — это по сути умный `ThreadPoolExecutor`, который сам управляет количеством потоков.

---

## 7. Как это эволюционировало в Android
![[Pasted image 20260606140153.png]]

## 8. AsyncTask — почему убрали (важно для собеса)

```kotlin
// Так делали раньше — НЕ ДЕЛАЙ ТАК
class LoadUserTask(val activity: MainActivity) : AsyncTask<Void, Void, User>() {

    override fun doInBackground(vararg params: Void?): User {
        return api.getUser() // фоновый поток
    }

    override fun onPostExecute(result: User) {
        activity.textView.text = result.name // UI поток
    }
}

// Использование:
LoadUserTask(this).execute()
```

Почему deprecated и удалён:

- Держит сильную ссылку на `Activity` → утечка памяти при повороте экрана
- По умолчанию задачи выполняются последовательно, не параллельно
- Нет нормальной отмены — `cancel()` не останавливает `doInBackground`
- Нельзя нормально обработать ошибки

---

## 9. Современный подход — корутины с lifecycle

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState = _uiState.asStateFlow()

    fun loadUser() {
        // viewModelScope автоматически отменяется когда ViewModel уничтожается
        viewModelScope.launch {
            _uiState.value = UiState.Loading

            try {
                // withContext переключает на пул IO-потоков
                val user = withContext(Dispatchers.IO) {
                    repository.getUser()
                }
                _uiState.value = UiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}
```

```kotlin
// Fragment — lifecycleScope привязан к жизненному циклу
class UserFragment : Fragment() {

    override fun onViewCreated(view: View, ...) {
        viewLifecycleOwner.lifecycleScope.launch {
            // repeatOnLifecycle — собирает только когда экран активен
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UiState.Loading -> showLoading()
                        is UiState.Success -> showUser(state.user)
                        is UiState.Error   -> showError(state.message)
                    }
                }
            }
        }
    }
}
```

---

## 10. Когда Thread всё ещё уместен

Корутины — это обёртка над потоками. Иногда нужен именно поток напрямую:

```kotlin
// Долгоживущий фоновый сервис — например, аудио-плеер
class AudioService : Service() {

    private val thread = HandlerThread("AudioThread").apply { start() }
    private val handler = Handler(thread.looper)

    fun playTrack(url: String) {
        handler.post {
            // декодирование аудио в отдельном потоке
            audioPlayer.play(url)
        }
    }

    override fun onDestroy() {
        thread.quitSafely()  // корректное завершение
    }
}
```

`HandlerThread` — это Thread со своим Looper, удобен для последовательной обработки задач в одном фоновом потоке.

---

## 11. Частые вопросы на собесе

**Что такое ANR и почему возникает?** Application Not Responding — система показывает диалог если Main Thread заблокирован больше 5 секунд (для Activity) или 10 секунд (для Broadcast). Причина — тяжёлые операции или сеть на Main Thread.

**Почему нельзя обновлять UI из фонового потока?** Android View не потокобезопасны. Внутри каждый View проверяет `checkThread()` — если вызов не из Main Thread, бросает `CalledFromWrongThreadException`.

**Чем корутина отличается от Thread?** Thread — ресурс ОС, занимает ~1 МБ памяти и требует переключения контекста. Корутина — объект в JVM, занимает несколько килобайт, переключается без участия ОС. 10 000 потоков убьют приложение, 10 000 корутин — норма.

**Что такое `Dispatchers.IO` под капотом?** Это `ThreadPoolExecutor` с пулом до 64 потоков (или по числу CPU если их больше). Kotlin сам управляет размером пула.

---

## Шпаргалка

```
Main Thread   → только UI, никаких блокировок
Thread        → устарел, нет lifecycle, ручное управление
HandlerThread → Thread + Looper, для последовательных задач
Handler       → отправить задачу в очередь потока
AsyncTask     → удалён в API 30, не использовать
viewModelScope → корутина, живёт пока жив ViewModel
lifecycleScope → корутина, живёт пока жив View
Dispatchers.IO → пул потоков для сети и файлов
```