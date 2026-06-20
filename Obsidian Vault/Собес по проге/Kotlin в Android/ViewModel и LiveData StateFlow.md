Это связующее звено в современном Android: **ViewModel** хранит данные, а **StateFlow** доставляет их в **Compose**, обеспечивая реактивность и защиту от потери данных при повороте экрана.

---

# ViewModel и StateFlow в Android

В современной архитектуре (MVVM) роль **ViewModel** — подготовить данные для экрана, а роль **UI (Compose)** — просто отобразить их.

## 1. Зачем нужна ViewModel?

В Android `Activity` и `Fragment` могут уничтожаться и создаваться заново (например, при смене темы или повороте экрана).

- **ViewModel** живет в памяти до тех пор, пока экран не будет закрыт окончательно.
    
- Она не должна ничего знать о классах UI (`Activity`, `View`, `Context`), иначе возникнут утечки памяти.
    

## 2. Использование StateFlow для состояния экрана

Вместо `LiveData` (старый стандарт) сейчас используют `StateFlow`. Это "горячий" поток данных, который всегда хранит последнее состояние.

- **MutableStateFlow**: Используется внутри ViewModel для изменения данных.
    
- **StateFlow**: Публичная версия только для чтения, на которую подписывается UI.
    

---

## 3. Жизненный цикл и viewModelScope

Для запуска корутин во ViewModel используется встроенная область видимости `viewModelScope`.

- Все корутины, запущенные в этом скоупе, **автоматически отменяются**, когда ViewModel очищается (экран закрыт). Это избавляет от ручного управления памятью.
    

---

## 4. Пример архитектуры для Obsidian

Этот пример показывает классический паттерн: загрузка списка данных с обработкой состояния.

### Шаг 1: ViewModel

```Kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

// Описываем состояние экрана через sealed class или data class
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<String>) : UiState()
    data class Error(val message: String) : UiState()
}

class MyViewModel : ViewModel() {

    // Внутреннее состояние (изменяемое)
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    // Внешнее состояние (только для чтения)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    init {
        loadData()
    }

    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                // Имитация запроса в сеть
                val result = fetchDataFromApi() 
                _uiState.value = UiState.Success(result)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}
```

### Шаг 2: UI (Jetpack Compose)

Чтобы подписаться на `StateFlow` в Compose, используется метод `collectAsStateWithLifecycle()`. Он автоматически останавливает сбор данных, когда приложение свернуто, экономя ресурсы.

```Kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = viewModel()) {
    // Подписка на поток данных с учетом жизненного цикла
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    when (val s = state) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> LazyColumn {
            items(s.data) { item -> Text(item) }
        }
        is UiState.Error -> Text("Ошибка: ${s.message}")
    }
}
```

---
Хотя **StateFlow** сейчас считается современным стандартом в Kotlin-разработке, **LiveData** — это фундамент, на котором построено огромное количество существующих проектов. Понимать её работу критически важно, особенно если тебе придется поддерживать старый код или интегрироваться с библиотеками, которые её используют.
# LiveData в деталях

**LiveData** — это наблюдаемый класс хранилища данных (Observable data holder), который, в отличие от обычного `Flow`, **учитывает жизненный цикл** компонентов Android (Activity, Fragment, Service).

## 1. Главная особенность: Lifecycle Awareness

LiveData знает, в каком состоянии находится твой экран (Started, Resumed, Stopped).

- **Не доставляет обновления**, если подписчик (экран) находится в фоне (в состоянии `Stopped` или `Destroyed`). Это предотвращает краши из-за попыток обновить UI, которого уже нет.
    
- **Автоматическая отписка**: Когда Activity уничтожается, LiveData сама удаляет подписчика. Никаких утечек памяти.
    
- **Актуальные данные**: Если экран вернулся из фона в активное состояние, он тут же получает последнее актуальное значение из LiveData.
    

---

## 2. Основные типы

- **`LiveData<T>`**: Только для чтения. На неё подписывается UI.
    
- **`MutableLiveData<T>`**: Позволяет изменять значение. Обычно живет внутри ViewModel.
    

### Как менять значения:

Есть два способа обновить данные в `MutableLiveData`:

1. **`value = ...` (или `setValue`)**: Используется только из **главного (UI) потока**. Обновляет значение мгновенно.
    
2. **`postValue(...)`**: Используется для обновления из **фонового потока**. Она «перекидывает» задачу в главный поток и обновляет данные там.
    

---

## 3. Transformations (Манипуляции с LiveData)

Для LiveData есть вспомогательный класс `Transformations`, который позволяет менять поток данных аналогично операторам Flow.

- **`map`**: Преобразует результат.
    
    Kotlin
    
    ```
    val userLiveData: MutableLiveData<User> = ...
    val userName: LiveData<String> = userLiveData.map { it.name }
    ```
    
- **`switchMap`**: Используется, когда изменение одной LiveData должно запустить создание другой (например, смена ID пользователя запускает новый запрос в БД).
    

---

## 4. Пример кода для Obsidian

```Kotlin
class UserViewModel : ViewModel() {
    // Приватная изменяемая LiveData
    private val _userId = MutableLiveData<String>()
    
    // Публичная только для чтения
    val userId: LiveData<String> get() = _userId

    // Transformations: когда меняется ID, автоматически меняется и приветствие
    val welcomeMessage: LiveData<String> = _userId.map { id ->
        "Welcome, user #$id"
    }

    fun setUserId(id: String) {
        // Если мы в главном потоке
        _userId.value = id 
        
        // Если мы в фоновом потоке (например, в корутине)
        // _userId.postValue(id)
    }
}

// В Activity/Фрагменте:
viewModel.welcomeMessage.observe(viewLifecycleOwner) { message ->
    // Этот код выполнится только когда Activity активно
    textView.text = message
}
```

---

## 5. LiveData vs StateFlow: Когда и что?

|**Свойство**|**LiveData**|**StateFlow**|
|---|---|---|
|**Зависимость от Android**|Да (часть Jetpack)|Нет (чистый Kotlin)|
|**Жизненный цикл**|Встроено по умолчанию|Нужен `repeatOnLifecycle` в UI|
|**Начальное значение**|Не обязательно (может быть null)|**Обязательно**|
|**Потоки**|Только Main для UI|Гибкая настройка через Dispatchers|
|**Для чего лучше**|Простой UI, старые проекты|Сложная асинхронность, KMP|

---

## 6. Важные нюансы (Подводные камни)

- **Отсутствие многопоточности**: LiveData всегда доставляет данные в главный поток. Если тебе нужно делать сложные вычисления между этапами, LiveData превратит код в кашу.
    
- **Потеря данных**: Если ты вызываешь `postValue` несколько раз очень быстро до того, как главный поток успел их обработать, подписчик получит **только последнее** значение. Все промежуточные будут потеряны.
    
- **MediatorLiveData**: Это специальный подвид LiveData, который может объединять несколько других источников LiveData в один (например, слушать и БД, и сетевой статус одновременно).
    

---

### Ключевые моменты для конспекта:

- LiveData — это "старая гвардия", она простая и надежная для связи UI и ViewModel.
    
- Используй `MutableLiveData` внутри, `LiveData` — снаружи.
    
- `observe(viewLifecycleOwner) { ... }` — золотой стандарт подписки во фрагментах.
    
## 5. Сравнение StateFlow и LiveData

|**Характеристика**|**LiveData**|**StateFlow**|
|---|---|---|
|**Библиотека**|Android Lifecycle|Kotlin Coroutines (Pure Kotlin)|
|**Начальное значение**|Не обязательно|**Обязательно**|
|**Потоки (Threads)**|Только Main Thread|Любой поток (`Dispatchers`)|
|**Операторы**|Мало (Transformations)|Огромное количество (map, filter, combine)|

---

## Ключевые моменты для конспекта:

1. **Single Source of Truth**: UI всегда берет данные только из одного источника (StateFlow во ViewModel).
    
2. **Unidirectional Data Flow (UDF)**: Данные текут вниз (из ViewModel в UI), а события — вверх (клики из UI вызывают методы ViewModel).
    
3. **Encapsulation**: Всегда делайте `MutableStateFlow` приватным, чтобы UI не мог напрямую менять данные, а только подписывался на них.
    

---

**Что дальше?** Мы закрыли связку Logic + UI.

Есть ли желание разобрать **Dependency Injection (Hilt)** — как передавать зависимости во ViewModel (например, репозитории или БД)? Это финальный штрих в серьезной Android-архитектуре.