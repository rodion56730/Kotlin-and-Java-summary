# Компоненты Android

## Activity

Activity — компонент, представляющий один экран с пользовательским интерфейсом. Это точка входа для взаимодействия пользователя с приложением. Каждая Activity объявляется в `AndroidManifest.xml`, где также указываются Intent-фильтры — условия, при которых система может запустить этот экран (например, `ACTION_MAIN` + `CATEGORY_LAUNCHER` делает Activity стартовой точкой приложения).

**Жизненный цикл:**

- `onCreate` — Activity создаётся впервые; здесь инициализируется UI, ViewModel, привязываются данные
- `onStart` — Activity становится видимой, но ещё не в фокусе
- `onResume` — Activity в фокусе, принимает ввод пользователя; здесь запускаются анимации, камера
- `onPause` — Activity теряет фокус (другой компонент поверх); сохраняем критичные данные, останавливаем ресурсоёмкие операции
- `onStop` — Activity полностью скрыта; отписываемся от тяжёлых ресурсов
- `onDestroy` — Activity уничтожается (пользователь закрыл или система освобождает память)
- `onRestart` — Activity возвращается из состояния Stop

**Пересоздание** происходит при повороте экрана, смене языка, изменении конфигурации. Для сохранения состояния используют `onSaveInstanceState(Bundle)` и восстанавливают в `onCreate` или `onRestoreInstanceState`. Более современный подход — `ViewModel`, которая переживает пересоздание Activity.

**Запуск Activity** осуществляется через `Intent`:

```kotlin
startActivity(Intent(this, DetailActivity::class.java))
// С ожиданием результата (современный способ):
val launcher = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result -> }
```

**Back Stack** — система управляет стеком Activity: новая Activity кладётся сверху, кнопка «назад» снимает верхнюю. Поведение стека настраивается через `launchMode` в манифесте (`standard`, `singleTop`, `singleTask`, `singleInstance`) или флаги Intent (`FLAG_ACTIVITY_CLEAR_TOP`, `FLAG_ACTIVITY_NEW_TASK`).

**Task и Affinity** — Activity группируются в задачи (Task). По умолчанию все Activity одного приложения относятся к одной задаче. `taskAffinity` позволяет явно назначить Activity к конкретной задаче, что важно при сложной навигации между приложениями.

---

## Service

Service — компонент для выполнения длительных операций в фоне без пользовательского интерфейса. Работает в главном потоке приложения — тяжёлые операции нужно выносить в фоновый поток вручную (через Thread, Coroutines, RxJava). Объявляется в `AndroidManifest.xml`.

**Три типа Service:**

**Started Service** — запускается через `startService()` / `startForegroundService()`, живёт до явной остановки через `stopSelf()` или `stopService()`. Подходит для операций, которые нужно выполнить до конца независимо от компонента, который их запустил (загрузка файла, синхронизация).

**Bound Service** — клиенты привязываются через `bindService()` и получают объект `IBinder` для прямого взаимодействия с сервисом. Сервис живёт, пока есть хотя бы один клиент; при отвязке всех клиентов уничтожается. Удобен для длительных соединений (воспроизведение музыки с управлением).

**Foreground Service** — виден пользователю через постоянное уведомление в статус-баре, что даёт ему приоритет перед системой (значительно снижает вероятность уничтожения). Обязателен для операций, которые пользователь должен видеть: трекинг геолокации, воспроизведение медиа, запись экрана.

**Жизненный цикл:**

```
startService()  →  onCreate()  →  onStartCommand()  →  ... →  onDestroy()
bindService()   →  onCreate()  →  onBind()  →  ... →  onUnbind()  →  onDestroy()
```

`onStartCommand()` возвращает флаг поведения при перезапуске: `START_STICKY` (перезапустить без Intent), `START_NOT_STICKY` (не перезапускать), `START_REDELIVER_INTENT` (перезапустить с последним Intent).

**Ограничения фоновых сервисов** — начиная с Android 8.0 система убивает Started Service, запущенный из фона (приложение не на переднем плане). Для гарантированного выполнения задач используют `WorkManager` (для отложенных задач) или Foreground Service (для активных).

---

## BroadcastReceiver

BroadcastReceiver — компонент, реагирующий на широковещательные сообщения (broadcast) от системы или других приложений. Является реакцией на событие: изменение состояния сети, уровень заряда батареи, установка приложения, входящий звонок, кастомные события внутри приложения.

**Регистрация** бывает двух видов:

**Статическая** — объявляется в `AndroidManifest.xml`. Позволяет получать события даже когда приложение не запущено. Начиная с Android 8.0 большинство системных broadcast запрещены для статической регистрации (исключения: `BOOT_COMPLETED`, `SMS_RECEIVED` и др.).

```xml
<receiver android:name=".MyReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
    </intent-filter>
</receiver>
```

**Динамическая** — регистрируется в коде через `registerReceiver()` и обязательно отменяется через `unregisterReceiver()` (обычно в `onStop`/`onDestroy`). Работает только пока компонент жив. Подходит для событий, актуальных только на конкретном экране.

```kotlin
val receiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) { }
}
registerReceiver(receiver, IntentFilter(Intent.ACTION_BATTERY_LOW))
```

**`onReceive()`** — единственный метод для переопределения. Выполняется в главном потоке, должен завершиться максимально быстро (не более 10 секунд, иначе ANR). Тяжёлую работу нужно передавать в Service или WorkManager через `goAsync()` (возвращает `PendingResult`, позволяя завершить обработку асинхронно).

**Безопасность:** для ограничения доступа к кастомным broadcast используют `android:permission` в манифесте или `LocalBroadcastManager` (только внутри процесса приложения, устарел) / `EventBus`-подход через `LiveData` или `Flow` для внутренней коммуникации.

**Ordered Broadcast** — позволяет передавать сообщение последовательно по цепочке получателей по приоритету (`android:priority`); каждый может модифицировать данные или прервать цепочку через `abortBroadcast()`.

---

## ContentProvider

ContentProvider — компонент, предоставляющий структурированный доступ к данным приложения другим приложениям или компонентам через унифицированный интерфейс. Это единственный правильный способ делиться данными между приложениями в Android. Объявляется в `AndroidManifest.xml` с указанием `authorities` — уникального имени, по которому его находят.

**URI** — основной способ адресации данных: `content://authorities/table/id`. Приложение-клиент обращается к ContentProvider именно через URI, не зная, как данные хранятся внутри (SQLite, файл, сеть — всё абстрагировано).

**Основные методы для переопределения:**

- `query(uri, projection, selection, selectionArgs, sortOrder)` — выборка данных, возвращает `Cursor`
- `insert(uri, values)` — вставка записи, возвращает URI новой записи
- `update(uri, values, selection, selectionArgs)` — обновление записей
- `delete(uri, selection, selectionArgs)` — удаление записей
- `getType(uri)` — MIME-тип данных по URI
- `onCreate()` — инициализация провайдера (вызывается при старте приложения, до `Application.onCreate`)

**ContentResolver** — клиентская сторона: компоненты не обращаются к ContentProvider напрямую, а используют `context.contentResolver.query(...)`. Система сама находит нужный провайдер по authority URI и маршрутизирует запрос.

```kotlin
val cursor = contentResolver.query(
    ContactsContract.Contacts.CONTENT_URI,
    arrayOf(ContactsContract.Contacts.DISPLAY_NAME),
    null, null, null
)
```

**UriMatcher** — вспомогательный класс для сопоставления входящего URI с конкретным типом данных внутри провайдера (список или конкретная запись по id):

```kotlin
val matcher = UriMatcher(UriMatcher.NO_MATCH).apply {
    addURI("com.example", "users", 1)
    addURI("com.example", "users/#", 2)
}
```

**Безопасность:** доступ к провайдеру ограничивается через `android:readPermission` / `android:writePermission` в манифесте. Для временного доступа используются `grantUriPermissions` — механизм, позволяющий дать конкретному приложению доступ к конкретному URI без постоянного разрешения (например, передача файла через `FileProvider`).

**FileProvider** — специальная реализация ContentProvider из Jetpack, позволяющая безопасно делиться файлами между приложениями через URI вместо прямого пути к файлу (`file://`). Начиная с Android 7.0 использование `file://` URI в Intent запрещено — только `content://` через FileProvider.

**Потокобезопасность:** методы ContentProvider могут вызываться из разных потоков одновременно (разные клиенты), поэтому реализация должна быть потокобезопасной — например, через синхронизированные обращения к SQLite (Room и `SQLiteDatabase` сами по себе потокобезопасны).