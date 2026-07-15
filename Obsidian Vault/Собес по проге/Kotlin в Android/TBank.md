# Конспекты для подготовки к собеседованию Junior Android Developer (T-Bank)

---

# 1. BroadcastReceiver

## Что это

`BroadcastReceiver` — компонент Android, который получает и обрабатывает системные или кастомные широковещательные сообщения (broadcasts). Позволяет приложению реагировать на события: изменение сети, зарядка батареи, входящее SMS, завершение загрузки файла, кастомные события внутри приложения и т.д.

## Типы Broadcast

- **Системные (System broadcasts)** — отправляются самой ОС: `ACTION_BOOT_COMPLETED`, `CONNECTIVITY_ACTION`, `ACTION_BATTERY_LOW`, `ACTION_POWER_CONNECTED` и т.д.
- **Кастомные (Custom broadcasts)** — отправляются вашим или сторонним приложением через `sendBroadcast()`, `sendOrderedBroadcast()`.

## Виды по способу доставки

1. **Normal broadcast** (`sendBroadcast()`) — асинхронный, все ресиверы получают его одновременно, порядок не гарантирован.
2. **Ordered broadcast** (`sendOrderedBroadcast()`) — доставляется ресиверам по очереди согласно приоритету (`android:priority`), каждый может модифицировать результат или прервать цепочку (`abortBroadcast()`).
3. **Local broadcast** — через `LocalBroadcastManager` (устарел, начиная с AndroidX рекомендуется заменять на `LiveData`, `Flow`, `EventBus` или `SharedFlow`), работает только внутри приложения, безопаснее и быстрее системного.

## Способы регистрации

1. **Статическая регистрация (в Manifest)**

```xml
<receiver android:name=".MyReceiver" android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
    </intent-filter>
</receiver>
```

- С Android 8.0 (API 26) большинство _implicit_ системных broadcast нельзя регистрировать статически (ограничения фоновых процессов) — работают только явно задокументированные исключения.
- Начиная с Android 13 (API 33) обязательно указывать `RECEIVER_EXPORTED` или `RECEIVER_NOT_EXPORTED` при динамической регистрации.

2. **Динамическая регистрация (в коде)**

```kotlin
val receiver = MyReceiver()
val filter = IntentFilter(Intent.ACTION_BATTERY_LOW)
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    registerReceiver(receiver, filter, RECEIVER_NOT_EXPORTED)
} else {
    registerReceiver(receiver, filter)
}
```

- Живёт, пока жив компонент, который его зарегистрировал (обычно Activity/Fragment) — важно вызывать `unregisterReceiver()` в `onPause`/`onStop`/`onDestroy`, иначе — утечка памяти.

## Жизненный цикл

- Метод `onReceive(context, intent)` выполняется в **главном (UI) потоке**.
- Выполнение ограничено по времени (~10 секунд), после чего система может показать ANR (Application Not Responding).
- **Нельзя** делать долгие/асинхронные операции напрямую внутри `onReceive` — после выхода из метода ресивер может быть уничтожен системой.
- Правильный подход для длительной работы: делегировать в `WorkManager`, `JobIntentService` (устарел) или стартовать `Service`/`ForegroundService`.

## Ограничения фоновых broadcast (начиная с Android 8+)

Google ужесточил работу с implicit broadcasts, чтобы экономить батарею и память:

- Нельзя регистрировать большинство implicit broadcast receiver в манифесте.
- Рекомендуется использовать `JobScheduler`/`WorkManager` вместо broadcast для фоновых задач.

## Пример кастомного broadcast

```kotlin
// Отправка
val intent = Intent("com.example.MY_EVENT").apply {
    putExtra("data", "hello")
    setPackage(packageName) // ограничить рамками своего приложения (безопасность)
}
sendBroadcast(intent)

// Получение
class MyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val data = intent.getStringExtra("data")
    }
}
```

## Частые вопросы на собеседовании

- Чем `sendBroadcast` отличается от `sendOrderedBroadcast`?
- Почему нельзя делать сетевые запросы в `onReceive`?
- Почему `LocalBroadcastManager` считается устаревшим и чем его заменить?
- Какие ограничения появились в Android 8 и 13 для broadcast receiver?
- В чём разница между `exported=true/false`?

---

# 2. FragmentManager

## Что это

`FragmentManager` — класс, управляющий стеком фрагментов (back stack), их транзакциями (добавление, удаление, замена) внутри `FragmentActivity`/`Fragment`. Каждая `FragmentActivity` имеет свой `FragmentManager`, а у каждого `Fragment` есть **child FragmentManager** для вложенных фрагментов.

## Основные операции — FragmentTransaction

```kotlin
supportFragmentManager.beginTransaction()
    .replace(R.id.container, MyFragment())
    .addToBackStack("tag")
    .commit()
```

- `add()` — добавить фрагмент, не убирая текущий
- `replace()` — удалить текущий и добавить новый (фактически `remove` + `add`)
- `remove()` — удалить фрагмент
- `show()/hide()` — управлять видимостью без пересоздания
- `attach()/detach()` — временно отсоединить View, но сохранить состояние во `FragmentManager`
- `addToBackStack(name)` — сохранить транзакцию в back stack (для обработки кнопки "Назад")
- `commit()` — асинхронно ставит транзакцию в очередь на выполнение (в очередь главного потока)
- `commitNow()` — выполняет синхронно немедленно (нельзя использовать с `addToBackStack`)
- `commitAllowingStateLoss()` — как `commit()`, но не выбрасывает исключение, если состояние Activity уже сохранено (может привести к потере состояния — использовать осторожно)

## FragmentManager back stack

- Хранит транзакции, а не сами фрагменты как такового "стека" — при нажатии "Назад" система откатывает последнюю транзакцию в стеке.
- `popBackStack()` — программно вернуться на шаг назад.
- `OnBackStackChangedListener` — слушать изменения стека.

## Жизненный цикл фрагмента (важно знать порядок!)

```
onAttach() → onCreate() → onCreateView() → onViewCreated() 
→ onStart() → onResume()
   ...
→ onPause() → onStop() → onDestroyView() → onDestroy() → onDetach()
```

- `onCreateView()` — создание View, может вызываться несколько раз (например, при возврате из back stack), поэтому не стоит хранить тяжёлые ссылки на View здесь без очистки.
- `onDestroyView()` — View уничтожена, но сам объект Fragment ещё жив (важно обнулять ViewBinding здесь во избежание утечек памяти).
- `viewLifecycleOwner` vs `this` (Fragment как LifecycleOwner) — для `LiveData.observe()` и других View-related подписок нужно использовать `viewLifecycleOwner`, а не сам фрагмент, иначе возможны утечки/креши после `onDestroyView`.

## FragmentStateAdapter / коммуникация между фрагментами

- **Communication**: через `ViewModel`, разделяемую на уровне Activity (`by activityViewModels()`), через `Fragment Result API` (`setFragmentResult`/`setFragmentResultListener` — современная замена `targetFragment`), либо через интерфейсы, реализуемые в Activity.
- Раньше использовали `targetFragment`/`setTargetFragment` — сейчас **deprecated**, заменено на Fragment Result API.

## Частые ошибки/грабли

- **IllegalStateException: Can not perform this action after onSaveInstanceState** — попытка сделать `commit()` после того, как состояние Activity уже сохранено (например, в асинхронном callback). Решается через `commitAllowingStateLoss()` (не всегда безопасно) или правильным контролем жизненного цикла.
- Утечки памяти через View в `onCreateView`, если не очищать binding в `onDestroyView`.
- Дублирование фрагментов при повороте экрана, если не проверять `savedInstanceState == null` перед первой транзакцией в Activity.

## FragmentManager vs NavController (Navigation Component)

- `Navigation Component` — обёртка над `FragmentManager`, упрощает навигацию через граф (`nav_graph.xml`), Safe Args для передачи аргументов, единая обработка back stack, deep links.
- Под капотом Navigation всё равно использует `FragmentManager` и транзакции.

## Частые вопросы на собеседовании

- Чем `add` отличается от `replace`?
- Что такое back stack фрагментов, как он работает?
- Разница между `commit()`, `commitNow()`, `commitAllowingStateLoss()`?
- Почему нужно использовать `viewLifecycleOwner`, а не `this` в Fragment?
- Как правильно передавать данные между фрагментами?
- Что делать с `IllegalStateException` после `onSaveInstanceState`?

---

# 3. Обзор тем для собеседования Junior Android Developer

У вас уже неплохо закрыты темы Kotlin и часть Android/Kotlin. Вот темы, которые часто спрашивают на junior-собеседованиях (в т.ч. в T-Bank), но которых нет в вашем списке — рекомендую добавить:

## Android Core / Компоненты

- **Activity Lifecycle** — полный цикл, `onSaveInstanceState`, `configuration changes` (поворот экрана), `onRestoreInstanceState`
- **Service** — Started vs Bound Service, Foreground Service, отличие от `WorkManager`
- **ContentProvider** — назначение, `ContentResolver`, `Uri`
- **Intent** — explicit vs implicit, `Intent Filters`, `PendingIntent`
- **Context** — виды контекста (`Application`, `Activity`), утечки памяти через контекст
- **Process death & state restoration** — `SavedStateHandle`, чем отличается от `onSaveInstanceState`

## Многопоточность и асинхронность

- `Handler`, `Looper`, `HandlerThread`
- `WorkManager` — для отложенных/гарантированных фоновых задач
- Разница между `Thread`, `Coroutines`, `RxJava` (базово, для junior обычно спрашивают "почему корутины лучше потоков")
- ANR (Application Not Responding) — что это и как избежать

## UI

- **RecyclerView** — `ViewHolder`, `Adapter`, `DiffUtil`, `ListAdapter`
- **ViewBinding vs DataBinding vs findViewById**
- **ConstraintLayout** основы
- Разница `Fragment` vs `Activity` — когда что использовать

## Хранение данных

- `SharedPreferences` vs `DataStore` (Preferences/Proto)
- `Room` (у вас уже есть) — миграции, `@Relation`, DAO
- Кэширование (memory/disk cache)

## Сеть

- `Retrofit` (есть) + `OkHttp` — интерцепторы, таймауты
- Обработка ошибок сети, повторные запросы

## Архитектура и паттерны

- **SOLID** принципы
- Паттерны: Singleton, Factory, Observer, Builder, Strategy
- MVVM vs MVP vs MVI (у вас есть Clean+MVVM — стоит понимать разницу с MVI)
- Dependency Injection — Hilt/Dagger vs Koin, разница между DI-фреймворками

## Тестирование (есть в списке, но уточнить)

- Unit-тесты (`JUnit`, `MockK`/`Mockito`)
- UI-тесты (`Espresso`)
- `TestCoroutineDispatcher`/`runTest` для тестирования корутин

## Разное, что часто спрашивают

- `Parcelable` vs `Serializable` (есть) — почему `Parcelable` эффективнее
- `sealed class` vs `enum class` — где какой использовать
- `inline`, `noinline`, `reified` функции
- Memory leaks — типичные причины и как их находить (`LeakCanary`)
- Gradle basics — что такое `build.gradle`, `Dependency`, флейворы (`buildTypes`, `productFlavors`)
- Git — базовые команды, `merge` vs `rebase`
- ProGuard/R8 — зачем нужен, что делает

## Рекомендация по приоритету для T-Bank Junior

Судя по типичным списку вопросов в банковских Android-собеседованиях, чаще всего спрашивают:

1. Жизненный цикл Activity/Fragment (детально, с сценариями поворота экрана)
2. Coroutines/Flow (у вас есть — но готовьтесь к вопросам про `structured concurrency`, `Dispatchers`, `cancellation`)
3. RecyclerView + DiffUtil
4. DI (Hilt) — базовые аннотации (`@Inject`, `@Module`, `@Provides`, `@HiltViewModel`)
5. MVVM/Clean Architecture — про слои (`data/domain/presentation`)
6. Memory leaks и как их избегать
7. Многопоточность — Handler/Looper базово + Coroutines глубже

---

_Совет: на junior-собеседовании важнее не зазубрить определения, а уметь объяснить "зачем" и привести практический пример или грабли, с которыми сталкивались._