# CoroutineScope, viewModelScope, lifecycleScope

---

## 1. Зачем вообще нужен Scope

Представь что ты запустил корутину для загрузки данных. Пользователь закрыл экран. Что происходит с корутиной?

Без scope — **ничего**. Она продолжает работать, держит ссылку на уничтоженный экран, пытается обновить несуществующий UI. Это утечка памяти.

Scope решает это: **корутина живёт ровно столько, сколько живёт её scope**.
![[Pasted image 20260606142228.png]]

## 2. Что внутри CoroutineScope

Scope — это просто интерфейс с одним полем:

```kotlin
interface CoroutineScope {
    val coroutineContext: CoroutineContext
}
```

`CoroutineContext` — это контейнер, в котором лежат:

```kotlin
coroutineContext[Job]          // контролирует жизнь и отмену
coroutineContext[Dispatcher]   // на каком потоке выполняться
coroutineContext[CoroutineName] // имя для дебага
```

`Job` — это главное. Именно он знает о всех дочерних корутинах и умеет их отменять.

---

## 3. Создать Scope вручную

Чтобы понять готовые scope — сначала посмотрим как устроено изнутри:

```kotlin
// Создаём scope вручную
val scope = CoroutineScope(
    Dispatchers.Main + Job()
)

scope.launch {
    delay(1000)
    println("Выполнилось")
}

// Когда нужно — отменяем всё
scope.cancel()  // все корутины внутри отменяются
```

Именно это делают `viewModelScope` и `lifecycleScope` — создают scope и вызывают `cancel()` в нужный момент жизненного цикла.

---

## 4. viewModelScope — для ViewModel

Это scope, который **автоматически отменяется когда ViewModel уничтожается**.

```kotlin
class UserViewModel : ViewModel() {

    fun loadUser() {
        viewModelScope.launch {
            val user = repository.getUser()
            _uiState.value = UiState.Success(user)
        }
    }

    // НЕ НУЖНО переопределять onCleared() для отмены
    // viewModelScope делает это сам
}
```

Как это работает под капотом — в `ViewModel` уже встроено:

```kotlin
// Внутри AndroidX ViewModel — упрощённо
val ViewModel.viewModelScope: CoroutineScope
    get() {
        return CloseableCoroutineScope(
            SupervisorJob() + Dispatchers.Main.immediate
        )
    }

override fun onCleared() {
    viewModelScope.cancel()  // вот где отмена происходит
}
```

Ключевые моменты:

- Dispatcher по умолчанию — `Dispatchers.Main`
- Job — `SupervisorJob` (об этом ниже)
- Отменяется в `onCleared()` — когда экран уничтожен насовсем (не поворот)

---

## 5. lifecycleScope — для Fragment и Activity

Привязан к Lifecycle компонента. Отменяется когда `onDestroy()`.

```kotlin
class UserFragment : Fragment() {

    override fun onViewCreated(view: View, ...) {

        lifecycleScope.launch {
            val user = viewModel.getUser()
            binding.userName.text = user.name
        }
    }
}
```

Но здесь есть важный нюанс. `lifecycleScope` живёт до `onDestroy`, но при повороте экрана Fragment пересоздаётся — View уничтожается и создаётся заново. Если корутина обращается к View — нужен более точный контроль.

---

## 6. repeatOnLifecycle — самый правильный способ

```kotlin
class UserFragment : Fragment() {

    override fun onViewCreated(view: View, ...) {

        viewLifecycleOwner.lifecycleScope.launch {

            // repeatOnLifecycle запускает блок когда STARTED
            // и отменяет когда STOPPED (экран ушёл в фон)
            // при возврате — запускает снова
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    render(state)
                }
            }
        }
    }
}
```

Разница между просто `lifecycleScope.launch` и с `repeatOnLifecycle`:

```
Без repeatOnLifecycle:
  Экран в фоне → корутина продолжает работать, обрабатывает данные впустую

С repeatOnLifecycle(STARTED):
  Экран в фоне → корутина приостановлена
  Экран на переднем плане → корутина возобновляется
```

---

## 7. viewModelScope vs lifecycleScope — когда что

```kotlin
// viewModelScope — бизнес-логика, сетевые запросы
class UserViewModel : ViewModel() {

    fun saveUser(user: User) {
        viewModelScope.launch {
            // переживёт поворот экрана
            repository.saveUser(user)
        }
    }
}

// lifecycleScope — работа с UI
class UserFragment : Fragment() {

    override fun onViewCreated(...) {
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // только пока экран виден
                viewModel.uiState.collect { render(it) }
            }
        }
    }
}
```

Правило простое: если код касается UI — `lifecycleScope`. Если только данные — `viewModelScope`.

---

## 8. coroutineScope и supervisorScope — внутри корутины

Это функции (не классы), которые создают вложенный scope внутри уже запущенной корутины.

### coroutineScope

```kotlin
viewModelScope.launch {

    // coroutineScope ждёт завершения всех дочерних корутин
    // если одна упала — отменяются все остальные
    coroutineScope {
        val userDeferred = async { repository.getUser() }
        val postsDeferred = async { repository.getPosts() }

        val user = userDeferred.await()
        val posts = postsDeferred.await()

        // сюда попадём только когда оба запроса завершились
        _state.value = UiState(user, posts)
    }
}
```

### supervisorScope

```kotlin
viewModelScope.launch {

    // supervisorScope — если одна упала, остальные продолжают
    supervisorScope {
        launch {
            // если этот упадёт — второй не отменится
            loadMainContent()
        }
        launch {
            // необязательная часть
            loadRecommendations()
        }
    }
}
```

Разница:

```
coroutineScope   → один упал → все упали (атомарность)
supervisorScope  → один упал → остальные продолжают (независимость)
```

---

## 9. Job vs SupervisorJob

Это то, что лежит внутри scope и управляет отменой.

```kotlin
// Job — обычный
val scope = CoroutineScope(Job())
scope.launch { throw Exception("ошибка") }
// ВСЕ корутины в scope отменяются

// SupervisorJob — изолирует ошибки
val scope = CoroutineScope(SupervisorJob())
scope.launch { throw Exception("ошибка") }
// Только эта корутина отменяется, остальные продолжают
```

`viewModelScope` использует `SupervisorJob` — поэтому если один запрос упал, другие корутины в ViewModel продолжают работать.

---

## 10. GlobalScope — почему никогда не использовать

```kotlin
// ❌ Так делать нельзя
GlobalScope.launch {
    repository.loadData()
}
```

Почему плохо:

- Не привязан ни к чему — живёт всё время пока живёт приложение
- Не отменяется при уходе с экрана
- Держит ссылки на объекты — утечки памяти
- Невозможно контролировать и тестировать

---

## 11. Полная картина
![[Pasted image 20260606142300.png]]

## Шпаргалка для собеса

```
CoroutineScope   → интерфейс, внутри — Job + Dispatcher
Job              → контролирует жизнь, отменяет детей
SupervisorJob    → как Job, но ошибка одного не убивает остальных

viewModelScope   → умирает с ViewModel (не с поворотом экрана)
lifecycleScope   → умирает с onDestroy()
repeatOnLifecycle→ пауза в фоне, возобновление при возврате

coroutineScope { }  → параллельные задачи, атомарная отмена
supervisorScope { } → параллельные задачи, независимая отмена

GlobalScope      → запрещено, нет lifecycle
```