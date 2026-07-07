# Конспект: Activity и Fragment

## Activity

### Что это

`Activity` — компонент Android, представляющий один экран с UI, с которым может взаимодействовать пользователь. Является точкой входа для взаимодействия пользователя с приложением, регистрируется в `AndroidManifest.xml`.

### Полный жизненный цикл Activity

```
onCreate() → onStart() → onResume()
                                 ↕ (foreground, пользователь взаимодействует)
                            onPause() → onStop()
                                 ↕
                         onRestart() → onStart() → onResume()   (возврат)
                                 ↓
                            onDestroy()
```

|Метод|Когда вызывается|Что делать|
|---|---|---|
|`onCreate()`|Создание Activity|Инициализация UI (`setContentView`), восстановление состояния из `savedInstanceState`|
|`onStart()`|Activity становится видимой|Подготовка UI-related ресурсов|
|`onResume()`|Activity в фокусе, пользователь может взаимодействовать|Запуск анимаций, камеры, сенсоров|
|`onPause()`|Activity частично перекрыта (например, диалог, всплывающее окно)|Остановка анимаций, сохранение несохранённых данных — метод короткий и быстрый, т.к. блокирует переход к следующей Activity|
|`onStop()`|Activity полностью не видна|Освобождение тяжёлых ресурсов, остановка обновлений UI|
|`onRestart()`|Перед повторным `onStart()` после `onStop()`|Редко переопределяется|
|`onDestroy()`|Activity уничтожается (финально или из-за system-initiated death)|Финальная очистка ресурсов|

### onSaveInstanceState / onRestoreInstanceState

- `onSaveInstanceState(Bundle)` вызывается **перед `onStop()`** (не гарантированно перед `onPause`, начиная с API 28 порядок именно такой), когда Activity может быть уничтожена системой, но пользователь ожидает вернуться (поворот экрана, нехватка памяти).
- **Не вызывается**, если пользователь явно закрывает Activity (например, нажал системную кнопку "Назад" с завершением экрана) — это осознанное поведение системы.
- Восстановление: `onCreate(savedInstanceState)` и/или `onRestoreInstanceState(savedInstanceState)` (второй вызывается после `onStart()`, используется реже).
- В `Bundle` можно класть только небольшие объёмы данных (примитивы, `Parcelable`, `Serializable`) — не предназначен для хранения больших объектов.

### Configuration changes (поворот экрана и т.д.)

- По умолчанию при повороте экрана Activity **полностью пересоздаётся** (`onDestroy()` → `onCreate()`), т.к. могут понадобиться другие ресурсы (layout-land, значения, drawable).
- Способы справиться:
    1. **ViewModel** — переживает пересоздание Activity (но не process death)
    2. `android:configChanges="orientation|screenSize"` в манифесте — Activity не пересоздаётся, а получает колбэк `onConfigurationChanged()` (используется редко, теряется автоматика ресурсов под конфигурацию)
    3. `onSaveInstanceState`/`Bundle` — для мелких значений UI-состояния

### Launch modes (`android:launchMode`)

- **standard** (по умолчанию) — каждый раз создаётся новый экземпляр
- **singleTop** — если Activity уже наверху стека, новый экземпляр не создаётся, вызывается `onNewIntent()`
- **singleTask** — в рамках task существует единственный экземпляр; при повторном запуске все Activity над ним в стеке удаляются
- **singleInstance** — Activity единственная в своём отдельном task

### Intent и передача данных

- **Explicit Intent** — указывает конкретный класс компонента (`Intent(this, SecondActivity::class.java)`)
- **Implicit Intent** — указывает action/category, система сама находит подходящий компонент (например, "поделиться", "открыть ссылку")
- Передача данных: `putExtra()`/`getXxxExtra()`, для сложных объектов — `Parcelable` (предпочтительно) или `Serializable`
- `startActivityForResult()` — **устарел**, заменён на `ActivityResultContracts`/`registerForActivityResult()` (Activity Result API)

### Task и Back Stack

- **Task** — стек Activity, с которым взаимодействует пользователь в рамках одной "цепочки" навигации.
- Управляется флагами `Intent.FLAG_ACTIVITY_*` (`FLAG_ACTIVITY_NEW_TASK`, `FLAG_ACTIVITY_CLEAR_TOP` и т.д.)

---

## Fragment

### Что это

`Fragment` — модульная, переиспользуемая часть UI, которая живёт внутри `FragmentActivity` (или другого фрагмента как child). Имеет собственный жизненный цикл, но зависит от жизненного цикла хост-Activity.

### Жизненный цикл Fragment

```
onAttach() → onCreate() → onCreateView() → onViewCreated()
→ onStart() → onResume()
      ...
→ onPause() → onStop() → onDestroyView() → onDestroy() → onDetach()
```

| Метод                  | Что происходит                                                                            |
| ---------------------- | ----------------------------------------------------------------------------------------- |
| `onAttach(context)`    | Fragment привязан к Activity, доступен `Context`                                          |
| `onCreate()`           | Инициализация нефрагментной логики (без View)                                             |
| `onCreateView()`       | Инфлейт (создание) View через `LayoutInflater`, возвращает `View?`                        |
| `onViewCreated()`      | View уже создана — здесь настраивают адаптеры, слушатели, наблюдение за `LiveData`/`Flow` |
| `onStart()/onResume()` | Fragment видим/активен                                                                    |
| `onPause()/onStop()`   | Fragment уходит из фокуса/скрывается                                                      |
| `onDestroyView()`      | **View уничтожена, но объект Fragment ещё жив** (важно обнулять ViewBinding!)             |
| `onDestroy()`          | Уничтожение самого Fragment (нефрагментного состояния)                                    |
| `onDetach()`           | Открепление от Activity                                                                   |

### Важный нюанс: два жизненных цикла

У `Fragment` есть **два** `LifecycleOwner`:

- `this` (сам Fragment) — живёт с момента `onCreate()` до `onDestroy()`
- `viewLifecycleOwner` — живёт только между `onCreateView()` и `onDestroyView()`

**Правило:** любые подписки, связанные с View (`LiveData.observe()`, `Flow.collect` в `lifecycleScope`), должны использовать `viewLifecycleOwner.lifecycle`, а не `this` — иначе возможны утечки памяти или креши при повторном создании View (например, при возврате из back stack).

```kotlin
// Правильно
viewModel.data.observe(viewLifecycleOwner) { ... }

// Неправильно (может привести к утечке/двойной подписке)
viewModel.data.observe(this) { ... }
```

### ViewBinding в Fragment — типичный паттерн

```kotlin
private var _binding: FragmentHomeBinding? = null
private val binding get() = _binding!!

override fun onCreateView(...): View {
    _binding = FragmentHomeBinding.inflate(inflater, container, false)
    return binding.root
}

override fun onDestroyView() {
    super.onDestroyView()
    _binding = null // обязательно, иначе утечка памяти
}
```

### Способы создания Fragment и передачи аргументов

- Через `newInstance()` фабричный метод + `arguments = bundleOf(...)` (не через конструктор с параметрами — фрагмент может быть пересоздан системой без сохранения параметров конструктора)

```kotlin
companion object {
    fun newInstance(id: Int) = MyFragment().apply {
        arguments = bundleOf("id" to id)
    }
}
```

### Коммуникация между Activity/Fragment и между фрагментами

1. **Общая ViewModel** на уровне Activity — `by activityViewModels()`
2. **Fragment Result API** (`setFragmentResult` / `setFragmentResultListener`) — современная замена deprecated `targetFragment`
3. Интерфейсы, реализуемые Activity (устаревающий, но всё ещё встречающийся подход)

### Fragment vs Activity — когда что использовать

- `Activity` — как правило, одна на весь экран/точку входа приложения (single-activity architecture — сейчас распространённый подход, где одна Activity + много Fragment, управляемых Navigation Component).
- `Fragment` — переиспользуемые части UI, удобны для табов, ViewPager, адаптивных layout (телефон/планшет), пошаговых сценариев (wizard).

### Частые ошибки

- Утечки памяти через неочищенный `ViewBinding` в `onDestroyView`
- Использование `this` вместо `viewLifecycleOwner` для observe
- `IllegalStateException` при `commit()` после `onSaveInstanceState` (см. конспект по FragmentManager)
- Дублирование фрагментов при повороте экрана из-за отсутствия проверки `savedInstanceState == null`

---

## Activity vs Fragment — сравнение

![[Pasted image 20260708015542.png]]
## Частые вопросы на собеседовании

- Расскажите полный жизненный цикл Activity и Fragment.
- Что происходит при повороте экрана? Как этого избежать/обработать правильно?
- Разница между `onPause` и `onStop`?
- Почему `onSaveInstanceState` не всегда вызывается?
- Что такое `viewLifecycleOwner` и зачем он нужен?
- Как правильно передавать аргументы во Fragment и почему не через конструктор?
- В чём разница launch modes у Activity?
- Как реализовать single-activity архитектуру и какие у неё плюсы/минусы?