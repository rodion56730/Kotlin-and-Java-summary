**XML Layout** — это классический способ описания UI в Android. Ты пишешь разметку в `.xml`-файлах, система создаёт из них дерево объектов `View`, а ты управляешь этими объектами из кода. Это **императивный** подход: ты сам говоришь "найди этот элемент и измени его".

---

# Android XML Layout: Конспект для собеседования

## 1. Как это работает под капотом

Когда ты пишешь `setContentView(R.layout.activity_main)`, происходит следующее:

1. **Inflate** — система читает XML и создаёт дерево объектов `View` / `ViewGroup` в памяти.
2. **Measure** — каждый `ViewGroup` спрашивает детей: "какой тебе нужен размер?". Проходов может быть несколько.
3. **Layout** — каждому элементу назначаются координаты на экране.
4. **Draw** — элементы рисуются на `Canvas`.

> Это происходит при каждом вызове `invalidate()` или изменении размеров.

---

## 2. Основные ViewGroup (контейнеры)

### LinearLayout

Располагает детей в **одну линию** (горизонтально или вертикально).

- Атрибут `android:orientation` — `horizontal` или `vertical`.
- Атрибут `android:layout_weight` — распределяет пространство пропорционально.

⚠️ **Проблема с `weight`**: При использовании `weight` `LinearLayout` делает **два прохода measure** для каждого дочернего элемента. При глубокой вложенности это даёт экспоненциальный рост нагрузки. Именно поэтому появился `ConstraintLayout`.

### RelativeLayout

Элементы позиционируются **относительно друг друга** или родителя.

```xml
<RelativeLayout>
    <TextView android:id="@+id/title" />
    <!-- "разместить кнопку ниже title" -->
    <Button android:layout_below="@id/title" />
</RelativeLayout>
```

Устарел — `ConstraintLayout` умеет всё то же, но быстрее.

### ConstraintLayout ✅ (современный стандарт)

Описывает **ограничения (constraints)** между элементами. Делает **один проход measure** вместо нескольких — решает проблему вложенности.

```xml
<ConstraintLayout>
    <Button
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />
</ConstraintLayout>
```

**Ключевые фичи**:

- `Guidelines` — невидимые линии-ориентиры.
- `Chains` — равномерное распределение элементов.
- `Barrier` — граница, которая двигается по самому широкому элементу группы.
- `MotionLayout` — анимации прямо в XML.

### FrameLayout

Накладывает элементы **друг на друга** (Z-слои). Обычно используется как контейнер для `Fragment` или для наложения элементов.

### CoordinatorLayout

Расширенный `FrameLayout` из Material Design. Умеет координировать поведение дочерних элементов — например, скрывать `AppBar` при прокрутке списка.

---

## 3. Работа с View из кода

### ViewBinding (современный способ) ✅

Генерирует класс для каждого XML-файла. Безопасный доступ к элементам без `findViewById`.

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Прямой доступ, без каста и без риска NPE
        binding.usernameTextView.text = "Привет"
        binding.loginButton.setOnClickListener { /* ... */ }
    }
}
```

### DataBinding

Похоже на ViewBinding, но позволяет **прямо в XML** привязывать данные из ViewModel. Тяжелее, генерирует больше кода — в большинстве случаев достаточно ViewBinding.

```xml
<!-- DataBinding: данные прямо в разметке -->
<TextView android:text="@{viewModel.userName}" />
```

### findViewById (устаревший способ) ❌

```kotlin
val button = findViewById<Button>(R.id.loginButton) // можно получить null
```

Проблемы: нет проверки типа на этапе компиляции, можно получить `NullPointerException`.

---

## 4. RecyclerView — список элементов

Аналог `LazyColumn` в Compose. Рисует только видимые элементы, переиспользует `View`-объекты при прокрутке.

**Три обязательных части**:

1. **`RecyclerView`** в XML — контейнер.
2. **`Adapter`** — знает, как создать и заполнить одну ячейку.
3. **`LayoutManager`** — знает, как расположить ячейки (линейно, сеткой).

```kotlin
recyclerView.adapter = MyAdapter(items)
recyclerView.layoutManager = LinearLayoutManager(this)
```

**`DiffUtil`** — утилита для умного обновления списка. Сравнивает старый и новый список и применяет только нужные изменения (вместо `notifyDataSetChanged()` который мигает всем списком).

```kotlin
// Современный способ через ListAdapter + DiffUtil
class MyAdapter : ListAdapter<Item, MyViewHolder>(DiffCallback()) {
    class DiffCallback : DiffUtil.ItemCallback<Item>() {
        override fun areItemsTheSame(old: Item, new: Item) = old.id == new.id
        override fun areContentsTheSame(old: Item, new: Item) = old == new
    }
}
```

---

## 5. Проблема глубокой вложенности ⚠️

Это **классический вопрос на собеседовании**.

**Что происходит при глубокой вложенности:**

- Каждый `ViewGroup` при фазе `measure` обходит всех своих детей.
- `LinearLayout` с `weight` делает **два прохода** measure.
- При вложенности N уровней таких `LinearLayout` — число проходов растёт как **2^N**.

```
LinearLayout (weight)        → 2 прохода
  └── LinearLayout (weight)  → 4 прохода
        └── LinearLayout (weight)  → 8 проходов
              └── TextView   → 8 вызовов measure на одном элементе!
```

**Как решали:**

- `ConstraintLayout` — один проход, заменяет любую вложенность.
- `merge` тег — убирает лишний корневой элемент при `include`.
- `ViewStub` — элемент, который вообще не inflate до явного вызова.

---

## 6. Жизненный цикл View во Fragment

Это отдельная боль XML — у Fragment **два** жизненных цикла: сам Fragment и его View.

```
onAttach → onCreate → onCreateView → onViewCreated → onStart → onResume
                                                              ↓
                                              onPause → onStop → onDestroyView → onDestroy → onDetach
```

**Главное правило**: работай с View только между `onViewCreated` и `onDestroyView`. Binding нужно обнулять в `onDestroyView`, иначе утечка памяти.

```kotlin
private var _binding: FragmentHomeBinding? = null
private val binding get() = _binding!!

override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, ...) =
    FragmentHomeBinding.inflate(inflater, container, false).also { _binding = it }.root

override fun onDestroyView() {
    super.onDestroyView()
    _binding = null // Обязательно! Иначе Fragment держит View в памяти
}
```

---

## 7. include, merge, ViewStub

- **`<include>`** — вставляет один XML в другой. Переиспользование разметки.
- **`<merge>`** — убирает лишний корневой `ViewGroup` при `include`. Уменьшает глубину дерева.
- **`<ViewStub>`** — "заглушка", которая не создаёт View до явного вызова `.inflate()`. Ускоряет первоначальную загрузку экрана.

```xml
<!-- ViewStub: тяжёлый блок ошибки не создаётся, пока не нужен -->
<ViewStub
    android:id="@+id/errorStub"
    android:layout="@layout/layout_error"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
```

```kotlin
binding.errorStub.inflate() // Только когда реально нужно
```

---

## Сводная таблица

|**Компонент**|**Для чего**|**Статус**|
|---|---|---|
|`LinearLayout`|Линейное расположение|Устарел для сложных макетов|
|`RelativeLayout`|Позиционирование относительно других|Устарел|
|`ConstraintLayout`|Плоские сложные макеты|✅ Современный стандарт|
|`FrameLayout`|Наложение слоёв, контейнер для Fragment|✅|
|`CoordinatorLayout`|Координация Material-компонентов|✅|
|`RecyclerView`|Списки|✅|
|`ViewBinding`|Доступ к View без `findViewById`|✅|
|`DataBinding`|Привязка данных прямо в XML|Нишевый|
|`findViewById`|Поиск View по ID|❌ Устарел|

---

## Ключевые фразы для собеседования

- **"LinearLayout с weight делает два прохода measure"** — объясняет, почему вложенность дорогая.
- **"`ConstraintLayout` — один проход"** — почему он появился.
- **"ViewBinding безопаснее `findViewById`"** — проверка типов на этапе компиляции, нет NPE.
- **"Binding нужно обнулять в `onDestroyView`"** — про утечки памяти во Fragment.
- **"`DiffUtil` вместо `notifyDataSetChanged()`"** — умное обновление списка.
- **"`ViewStub` — ленивое создание View"** — оптимизация первого открытия экрана.