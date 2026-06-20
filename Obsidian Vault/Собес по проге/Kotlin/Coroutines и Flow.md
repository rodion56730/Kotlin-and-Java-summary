Подготовил тебе нормальный «разжёванный» конспект по корутинам и Flow с упором именно на Android-разработку, собеседования и реальные кейсы.  
Я специально объясню **не только ЧТО**, но и **ЗАЧЕМ**, потому что именно это обычно ломает понимание.

---

# Корутины и Flow в Android — подробный конспект

# 1. Зачем вообще нужны корутины

До корутин в Android были:

- `Thread`
    
- `AsyncTask`
    
- `RxJava`
    
- callbacks (`callback hell`)
    

Проблемы:

```kotlin
api.getUser(
    onSuccess = {
        api.getPosts(
            onSuccess = {
                api.getFriends(
                    onSuccess = {

                    }
                )
            }
        )
    }
)
```

Код превращался в лестницу из колбэков.

---

## Что дают корутины

Корутины позволяют писать асинхронный код как обычный синхронный:

```kotlin
val user = api.getUser()
val posts = api.getPosts()
val friends = api.getFriends()
```

НО:

- UI не блокируется
    
- главный поток свободен
    
- приложение не зависает
    

---

# 2. Что такое suspend на самом деле

Вот главный миф:

❌ `suspend` = новый поток

Это НЕ так.

---

## Как работает suspend

Когда корутина доходит до:

```kotlin
delay(1000)
```

она:

1. сохраняет своё состояние
    
2. освобождает поток
    
3. через 1 секунду продолжает выполнение
    

---

## Аналогия

Представь повара.

### Без корутин

Повар:

- поставил чайник
    
- стоит и ждёт 5 минут
    

Все остальные клиенты ждут.

---

### С корутинами

Повар:

- поставил чайник
    
- записал «вернуться через 5 минут»
    
- пошёл готовить другое
    

Вот это и делает `suspend`.

---

# 3. Что компилятор делает под капотом

Ты пишешь:

```kotlin
suspend fun loadUser(): User
```

Компилятор превращает примерно в:

```kotlin
fun loadUser(continuation: Continuation<User>): Any
```

---

## Что хранит Continuation

Continuation хранит:

- место остановки
    
- локальные переменные
    
- состояние функции
    

Это как save game в игре.

---

# 4. Почему корутины лёгкие

## Thread дорогой

Создание Thread:

- память
    
- переключение контекста ОС
    
- нагрузка на CPU
    

10000 потоков → приложение умрёт.

---

## Coroutine дешёвая

Корутина:

- просто объект
    
- хранит состояние
    
- не создаёт поток
    

Можно создать сотни тысяч корутин.

---

# 5. Dispatchers — где выполняется код

Dispatcher = менеджер потоков.

---

# Dispatchers.Main

Главный поток UI.

Только тут:

- обновление UI
    
- Compose state
    
- RecyclerView
    
- Navigation
    

```kotlin
withContext(Dispatchers.Main) {
    textView.text = "Готово"
}
```

---

# Dispatchers.IO

Для:

- сети
    
- Room
    
- файлов
    
- Retrofit
    
- чтения диска
    

```kotlin
withContext(Dispatchers.IO) {
    api.loadUsers()
}
```

---

## Почему отдельный IO dispatcher

Сеть = ожидание.

Поток большую часть времени ничего не делает.

Поэтому Kotlin разрешает много IO потоков.

---

# Dispatchers.Default

Для тяжёлых вычислений:

- сортировка
    
- парсинг JSON
    
- image processing
    
- DiffUtil
    
- криптография
    

```kotlin
withContext(Dispatchers.Default) {
    val result = bigList.sorted()
}
```

---

## Важное правило

### IO → ждём

### Default → считаем

---

# 6. Реальный Android пример

---

## Repository

```kotlin
class UserRepository(
    private val api: UserApi
) {

    suspend fun getUser(): User {
        return withContext(Dispatchers.IO) {
            api.getUser()
        }
    }
}
```

---

## ViewModel

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    fun loadUser() {
        viewModelScope.launch {

            val user = repository.getUser()

            _state.value = user
        }
    }
}
```

---

# Почему это правильно

## ViewModel знает:

- когда запускать
    

## Repository знает:

- где выполнять
    

Это хорошее разделение ответственности.

---

# 7. launch vs async

Это один из самых популярных вопросов.

---

# launch

Когда результат не нужен.

```kotlin
viewModelScope.launch {
    repository.saveUser()
}
```

Возвращает:

```kotlin
Job
```

---

# async

Когда нужен результат.

```kotlin
val deferred = async {
    repository.getUser()
}

val user = deferred.await()
```

Возвращает:

```kotlin
Deferred<T>
```

---

# Главное отличие

|launch|async|
|---|---|
|без результата|с результатом|
|fire-and-forget|future/promise|
|Job|Deferred|

---

# 8. Зачем нужен async в Android

## Параллельные запросы

Допустим:

- загрузка пользователя
    
- загрузка постов
    

---

## Плохо

```kotlin
val user = api.getUser()
val posts = api.getPosts()
```

Если:

- user = 1 сек
    
- posts = 1 сек
    

Итого:  
2 секунды.

---

## Хорошо

```kotlin
coroutineScope {

    val userDeferred = async {
        api.getUser()
    }

    val postsDeferred = async {
        api.getPosts()
    }

    val user = userDeferred.await()
    val posts = postsDeferred.await()
}
```

Итого:  
1 секунда.

---

# 9. Structured Concurrency

Самая важная концепция корутин.

---

# Проблема старых потоков

Можно было создать поток:

```java
new Thread()
```

и потерять ссылку.

Поток живёт вечно → memory leak.

---

# Kotlin решает это через scope

Все корутины живут внутри scope.

---

## В Android

### lifecycleScope

Живёт пока живёт Activity/Fragment.

---

### viewModelScope

Живёт пока живёт ViewModel.

---

# Пример

```kotlin
viewModelScope.launch {

    while (true) {
        delay(1000)
        println("Работаем")
    }
}
```

Когда экран уничтожится:

- ViewModel очистится
    
- корутина отменится автоматически
    

---

# Без этого был бы leak

Корутина продолжала бы:

- делать запросы
    
- обновлять UI
    
- держать ссылки на Activity
    

---

# 10. Cancellation (отмена)

Корутины отменяются кооперативно.

---

# Что значит кооперативно

Корутина САМА должна проверять отмену.

---

## suspend функции делают это автоматически

```kotlin
delay()
withContext()
await()
```

проверяют отмену.

---

# Проблема CPU циклов

```kotlin
while(true) {

}
```

Такой цикл невозможно отменить.

---

# Правильно

```kotlin
while(isActive) {

}
```

---

# Реальный Android пример

## Поиск

Пользователь:

- вводит "a"
    
- потом "ab"
    
- потом "abc"
    

Старые запросы нужно отменять.

---

## Flow + collectLatest

```kotlin
searchFlow.collectLatest {

    repository.search(it)
}
```

Если пришёл новый текст:

- старый запрос отменяется
    

Очень важно для:

- поиска
    
- autocomplete
    
- debounce
    

---

# 11. Flow — что это такое

Flow = асинхронный поток данных.

---

# Пример

```kotlin
flow {
    emit(1)
    emit(2)
    emit(3)
}
```

---

# Аналогия

Flow похож на:

- трубу
    
- conveyor belt
    
- stream
    

---

# Почему Flow нужен в Android

Потому что данные меняются со временем:

- база данных
    
- поиск
    
- websocket
    
- UI state
    
- location updates
    

---

# 12. Cold Flow (холодный)

Flow по умолчанию холодный.

---

# Что это значит

Flow ничего не делает пока нет collect.

---

## Пример

```kotlin
val flow = flow {
    println("START")
    emit(1)
}
```

Пока нет:

```kotlin
flow.collect()
```

ничего не произойдёт.

---

# Аналогия

YouTube видео:

- пока не нажал play
    
- видео не идёт
    

---

# 13. collect

Collect запускает flow.

```kotlin
flow.collect {
    println(it)
}
```

---

# 14. map/filter

Очень похожи на коллекции.

---

# map

```kotlin
flow
    .map {
        it.toUiModel()
    }
```

---

# Реальный Android пример

DTO → UI model

```kotlin
.map { userDto ->
    UserUi(
        name = userDto.name,
        avatar = userDto.avatar
    )
}
```

---

# filter

```kotlin
.filter {
    it.isNotEmpty()
}
```

---

# Реальный пример поиска

```kotlin
searchFlow
    .filter {
        it.length > 2
    }
```

Не искать пока строка короткая.

---

# 15. debounce

Очень важный оператор.

---

# Проблема

Пользователь печатает:

```text
a
ab
abc
```

Не надо делать 3 запроса.

---

# Решение

```kotlin
searchFlow
    .debounce(300)
```

Flow подождёт:

- пока пользователь перестанет печатать
    

---

# Реальный production пример

```kotlin
searchFlow
    .debounce(300)
    .distinctUntilChanged()
    .collectLatest {
        repository.search(it)
    }
```

---

# Что делает distinctUntilChanged

Не отправляет одинаковые значения.

---

# 16. collectLatest

Супер важный оператор.

---

## Проблема

Запрос "android":

- выполняется 2 секунды
    

Пользователь уже ввёл:  
"android kotlin"

Старый запрос больше не нужен.

---

# collectLatest

```kotlin
.collectLatest {

}
```

отменит старую обработку.

---

# 17. StateFlow

Самый важный Flow в Android сейчас.

Особенно с Compose.

---

# Что такое StateFlow

Flow который:

- хранит состояние
    
- всегда имеет value
    
- отдаёт последнее значение
    

---

# Реальный пример

```kotlin
private val _uiState =
    MutableStateFlow(UiState())

val uiState =
    _uiState.asStateFlow()
```

---

# Почему StateFlow идеален для UI

UI всегда должен знать:

- текущее состояние
    

Например:

- loading
    
- success
    
- error
    

---

# Реальный Compose пример

```kotlin
val state by viewModel.uiState.collectAsState()
```

Compose автоматически:

- подписывается
    
- перерисовывается
    

---

# 18. SharedFlow

Для событий.

---

# Разница между состоянием и событием

## State

"Что сейчас"

```text
Loading
Success
Error
```

---

## Event

"Что произошло"

```text
Показать toast
Открыть экран
Показать snackbar
```

---

# Почему StateFlow плох для событий

StateFlow хранит последнее значение.

После поворота экрана:

- toast покажется снова

---

# SharedFlow решает это

```kotlin
private val _events =
    MutableSharedFlow<Event>()
```

---

# Реальный пример

```kotlin
_events.emit(
    Event.ShowToast("Ошибка")
)
```

---

# 19. Room + Flow

Один из самых мощных кейсов.

---

# DAO

```kotlin
@Query("SELECT * FROM users")
fun getUsers(): Flow<List<User>>
```

---

# Что происходит

Room:

- слушает изменения БД
- автоматически emit новые данные
---

# UI обновится автоматически

```kotlin
viewModel.users.collect {
    adapter.submitList(it)
}
```
---

# 20. combine

Объединение flow.

---

# Реальный пример

Есть:

- список пользователей
    
- строка поиска
    

---

## combine

```kotlin
combine(
    usersFlow,
    searchFlow
) { users, search ->

    users.filter {
        it.name.contains(search)
    }
}
```

---

# Что круто

При изменении:

- поиска ИЛИ
    
- пользователей
    

результат обновится автоматически.

---

# 21. zip vs combine

Очень любят спрашивать.

---

# zip

Ждёт пары.

```text
A: 1 --- 2 --- 3
B: a --- b --- c

Результат:
1a
2b
3c
```

---

# combine

Берёт последние значения.

```text
A: 1 --- 2
B: a

combine:
1a
2a
```

---

# 22. buffer

---

# Проблема

Flow генерирует данные быстрее чем UI обрабатывает.

---

# buffer()

```kotlin
flow.buffer()
```

Позволяет:

- producer работать быстрее
    
- consumer догонять позже
    

---

# 23. conflate

Берёт только последнее значение.

---

# Реальный пример

GPS updates.

Тебе не нужны:

- все 1000 координат
    

Нужна:

- последняя.
    

---

# 24. Архитектурный production пример

# ViewModel

```kotlin
class SearchViewModel(
    private val repository: Repository
) : ViewModel() {

    private val query =
        MutableStateFlow("")

    val uiState = query
        .debounce(300)
        .filter {
            it.isNotBlank()
        }
        .distinctUntilChanged()
        .flatMapLatest {
            repository.search(it)
        }
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5000),
            emptyList()
        )

    fun updateQuery(text: String) {
        query.value = text
    }
}
```

---

# Что здесь происходит

---

## debounce

Не спамим запросами.

---

## filter

Игнор пустых строк.

---

## distinctUntilChanged

Не повторяем одинаковые запросы.

---

## flatMapLatest

Если пришёл новый запрос:

- старый Flow отменяется
    

---

## stateIn

Превращает cold Flow в StateFlow.

---

# 25. Частые ошибки Junior Android разработчиков

---

# Ошибка 1

```kotlin
GlobalScope.launch
```

❌ Никогда.

Почему:

- нет lifecycle
    
- утечки
    
- не отменяется
    

---

# Ошибка 2

Сеть в Main.

```kotlin
Dispatchers.Main
```

для Retrofit запроса.

→ лаги UI.

---

# Ошибка 3

StateFlow для событий.

→ повторные navigation/toast.

---

# Ошибка 4

Не обрабатывать ошибки.

---

# Правильно

```kotlin
catch {

}
```

или

```kotlin
try-catch
```

---

# 26. Что спрашивают на собеседованиях

---

# Частые вопросы

---

## Чем coroutine лучше thread?

- легче
    
- дешевле
    
- structured concurrency
    
- cancellation
    
- проще async код
    

---

## Чем StateFlow отличается от SharedFlow?

|StateFlow|SharedFlow|
|---|---|
|хранит state|события|
|всегда value|может не иметь|
|replay = 1|configurable|
|UI state|navigation/toast|

---

## Чем launch отличается от async?

|launch|async|
|---|---|
|Job|Deferred|
|без результата|с результатом|

---

## Почему suspend не блокирует поток?

Потому что:

- continuation
    
- сохранение состояния
    
- освобождение потока
    

---

## Что такое cold flow?

Flow стартует только при collect.

---

# 27. Как всё это выглядит в современной Android архитектуре

```text
UI (Compose/Fragment)
        ↓
ViewModel
        ↓
StateFlow
        ↓
Repository
        ↓
Retrofit / Room
```

---

# 28. Главное понимание

## Корутины — это:

не про потоки.

Они про:

- управление асинхронностью
    
- структурированную отмену
    
- удобный код
    

---

## Flow — это:

не просто список.

Это:

- поток изменений во времени
    

---

# 29. Мини-шпаргалка

---

## suspend

«Можно поставить на паузу»

---

## launch

«Запустить и не ждать результат»

---

## async

«Запустить и получить результат»

---

## Flow

«Асинхронный поток данных»

---

## StateFlow

«Текущее состояние UI»

---

## SharedFlow

«Одноразовые события»

---

## collectLatest

«Новое важнее старого»

---

## debounce

«Подожди пока пользователь закончит ввод»

---

## Dispatchers.IO

Сеть / БД / файлы

---

## Dispatchers.Default

CPU вычисления

---

## viewModelScope

Автоотмена при уничтожении экрана

---

## Краткая шпаргалка

- **`suspend`** — не поток, а «закладка». Компилятор превращает в `Continuation`.
- **`Dispatchers.IO`** — сеть и файлы. **`Default`** — тяжёлые вычисления. **`Main`** — UI.
- **Structured Concurrency** — дерево скоупов. Отменился родитель → отменились все дети.
- **`launch`** — "выстрелил и забыл". **`async`** — нужен результат. **`withContext`** — переключить поток.
- **`flow`** — холодный (как YouTube-ролик, стартует при просмотре).
- **`StateFlow`** — горячий, хранит состояние (как статус в мессенджере).
- **`SharedFlow`** — горячий, рассылает события (как пуш-уведомление).
- **`collectLatest`** — "если пришло новое — забудь про старое".