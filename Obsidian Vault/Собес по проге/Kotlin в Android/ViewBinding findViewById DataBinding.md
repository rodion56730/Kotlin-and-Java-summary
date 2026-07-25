# ViewBinding, findViewById и DataBinding

## findViewById

Классический способ получить ссылку на View по её идентификатору из XML-макета. Метод обходит дерево View в поисках элемента с нужным `id` и возвращает его. Основная проблема — отсутствие типобезопасности и null-safety: можно указать неверный тип (`TextView` вместо `Button`) или `id` из другого макета — ошибка проявится только в рантайме с `ClassCastException` или `NullPointerException`. Также порождает много boilerplate-кода при большом количестве элементов. Актуален только в старых проектах или при работе с простыми динамическими View, где ViewBinding недоступен.

```kotlin
val button = findViewById<Button>(R.id.myButton) // NPE если id не найден
button.setOnClickListener { }
```

---

## ViewBinding

ViewBinding — механизм, генерирующий binding-класс для каждого XML-макета на этапе компиляции. Binding-класс содержит типизированные ссылки на все View с `id` в макете. Полностью типобезопасен и null-safe: невозможно обратиться к View из чужого макета или получить неверный тип — ошибка будет поймана компилятором, а не в рантайме.

**Подключение** в `build.gradle`:

```kotlin
android {
    buildFeatures { viewBinding = true }
}
```

**Использование в Activity:**

```kotlin
private lateinit var binding: ActivityMainBinding

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ActivityMainBinding.inflate(layoutInflater)
    setContentView(binding.root)
    binding.myButton.setOnClickListener { }
}
```

**Использование во Fragment** — важно не держать binding дольше жизненного цикла View, иначе утечка памяти:

```kotlin
private var _binding: FragmentMainBinding? = null
private val binding get() = _binding!!

override fun onCreateView(...): View {
    _binding = FragmentMainBinding.inflate(inflater, container, false)
    return binding.root
}

override fun onDestroyView() {
    super.onDestroyView()
    _binding = null // обязательно!
}
```

Если макет нужно исключить из генерации binding-класса — добавить `tools:viewBindingIgnore="true"` в корневой тег XML. ViewBinding не поддерживает двустороннее связывание данных и выражения в XML — только доступ к View.

---

## DataBinding

DataBinding — более мощный инструмент: позволяет напрямую привязывать данные (переменные, LiveData, ViewModel) к XML-макету через выражения, исключая написание boilerplate-кода в Activity/Fragment для обновления UI.

**Подключение:**

```kotlin
android {
    buildFeatures { dataBinding = true }
}
```

**Макет** оборачивается в тег `<layout>`, внутри объявляются `<data>` с переменными:

```xml
<layout>
    <data>
        <variable name="user" type="com.example.User"/>
    </data>
    <TextView android:text="@{user.name}" />
</layout>
```

**Односторонняя привязка** `@{expression}` — данные идут из модели в View. **Двусторонняя привязка** `@={expression}` — данные синхронизируются в обоих направлениях (например, `EditText` и поле модели).

**Binding Adapters** — аннотация `@BindingAdapter` позволяет описать кастомную логику привязки для любого атрибута View:

```kotlin
@BindingAdapter("imageUrl")
fun loadImage(view: ImageView, url: String) {
    Glide.with(view).load(url).into(view)
}
// В XML: app:imageUrl="@{viewModel.avatarUrl}"
```

**LiveData** поддерживается нативно: если передать `lifecycleOwner` в binding, LiveData-переменные автоматически обновляют UI при изменении:

```kotlin
binding.lifecycleOwner = viewLifecycleOwner
binding.viewModel = viewModel
```

**Отличия от ViewBinding:**

||ViewBinding|DataBinding|
|---|---|---|
|Генерация класса|Для всех макетов|Только для `<layout>`|
|Выражения в XML|Нет|Да (`@{}`, `@={}`)|
|Привязка данных|Нет|Да (LiveData, ViewModel)|
|Скорость компиляции|Быстрее|Медленнее (annotation processing)|
|Сложность|Простой|Выше (логика в XML сложнее дебажить)|

На практике ViewBinding предпочтительнее для большинства экранов — он проще и быстрее компилируется. DataBinding оправдан в проектах, где активно используется паттерн MVVM с ViewModel и LiveData, и важно минимизировать код в Fragment/Activity.

---
