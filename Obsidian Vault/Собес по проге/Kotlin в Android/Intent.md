## Intent в Android

Intent — объект-сообщение, через который компоненты Android взаимодействуют друг с другом: запускают Activity, Service, отправляют Broadcast. Это основной механизм коммуникации между компонентами как внутри одного приложения, так и между разными приложениями.

---

## Два типа Intent

**Явный (Explicit)** — явно указывает целевой компонент по классу. Используется для навигации внутри своего приложения.

```kotlin
// Запуск конкретной Activity
val intent = Intent(this, DetailActivity::class.java)
intent.putExtra("reportId", 42)
startActivity(intent)

// Запуск своего сервиса
val serviceIntent = Intent(this, SyncService::class.java)
startService(serviceIntent)
```

**Неявный (Implicit)** — описывает действие, не указывая конкретный компонент. Система сама находит подходящий компонент по Intent-фильтру. Если подходящих несколько — показывает chooser.

```kotlin
// Открыть ссылку — система выберет браузер
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))
startActivity(intent)

// Поделиться текстом
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "Привет!")
}
startActivity(Intent.createChooser(shareIntent, "Поделиться через"))

// Позвонить
val callIntent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:+79001234567"))
startActivity(callIntent)
```

---

## Структура Intent

- **Action** — что нужно сделать: `ACTION_VIEW`, `ACTION_SEND`, `ACTION_CALL`, `ACTION_MAIN`
- **Data** — URI с данными для действия: `tel:`, `mailto:`, `content://`, `https://`
- **Category** — дополнительная характеристика компонента: `CATEGORY_LAUNCHER`, `CATEGORY_BROWSABLE`
- **Type** — MIME-тип данных: `image/*`, `text/plain`, `application/pdf`
- **Extras** — пара ключ-значение для передачи дополнительных данных
- **Flags** — управление поведением стека и жизненным циклом

---

## Передача данных через Extras

Extras поддерживают примитивы, String, Serializable, Parcelable и их массивы/списки.

```kotlin
// Отправитель
intent.putExtra("userId", 123)
intent.putExtra("userName", "Иван")
intent.putExtra("user", user) // user : Parcelable

// Получатель
val userId = intent.getIntExtra("userId", 0)
val userName = intent.getStringExtra("userName")
val user = intent.getParcelableExtra<User>("user")
```

**Ограничение:** размер данных в Extras ограничен размером Binder-буфера — около **1 МБ на весь процесс**. Передавать большие объекты (Bitmap, большие списки) через Intent нельзя — будет `TransactionTooLargeException`. Вместо этого передают id, а данные загружают из БД или передают через ViewModel/SharedFlow.

---

## Intent Flags

Флаги управляют поведением Back Stack и жизненным циклом Activity.

```kotlin
// Очистить стек и сделать Activity корневой (например, после логаута)
intent.flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK

// Не создавать новый экземпляр если Activity уже на вершине стека
intent.flags = Intent.FLAG_ACTIVITY_SINGLE_TOP

// Вернуться к существующему экземпляру Activity и очистить всё что над ней
intent.flags = Intent.FLAG_ACTIVITY_CLEAR_TOP
```

---

## Intent Filter

Объявляется в манифесте — описывает какие неявные Intent'ы компонент готов обработать.

```xml
<activity android:name=".ShareActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="image/*"/>
    </intent-filter>
</activity>
```

Теперь при `ACTION_SEND` с изображением система предложит это приложение в списке.

---

## PendingIntent

Обёртка над Intent, которую можно передать другому приложению или системе для выполнения **от имени твоего приложения** в будущем. Используется в уведомлениях, виджетах, AlarmManager, WorkManager.

```kotlin
// Intent который откроет MainActivity при нажатии на уведомление
val intent = Intent(context, MainActivity::class.java)
val pendingIntent = PendingIntent.getActivity(
    context,
    requestCode,
    intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

// Использование в уведомлении
NotificationCompat.Builder(context, CHANNEL_ID)
    .setContentIntent(pendingIntent) // нажатие на уведомление
    .addAction(R.drawable.ic_reply, "Ответить", replyPendingIntent) // кнопка действия
    .build()
```

**FLAG_IMMUTABLE** — обязателен с Android 12: запрещает другим приложениям изменять Intent внутри PendingIntent. **FLAG_MUTABLE** — нужен только в специфичных случаях (например, inline-reply в уведомлениях).

---

## Получение результата от Activity

Современный способ через **ActivityResultLauncher** (старый `startActivityForResult` — deprecated):

```kotlin
// Регистрируем launcher
val launcher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == Activity.RESULT_OK) {
        val data = result.data?.getStringExtra("result")
    }
}

// Запускаем
launcher.launch(Intent(this, PickerActivity::class.java))

// В PickerActivity возвращаем результат
val resultIntent = Intent().putExtra("result", "выбранное значение")
setResult(Activity.RESULT_OK, resultIntent)
finish()
```

**Стандартные контракты** из Jetpack избавляют от написания Intent вручную:

```kotlin
// Выбор фото из галереи
val pickPhoto = registerForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri ->
    // uri выбранного фото
}
pickPhoto.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))

// Запрос разрешения
val requestPermission = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
    if (granted) { /* разрешение получено */ }
}
requestPermission.launch(Manifest.permission.CAMERA)
```

---

## Безопасность

**Проверять Intent перед обработкой** — если Activity принимает неявные Intent'ы, любое приложение может их отправить с произвольными данными:

```kotlin
override fun onCreate(...) {
    val action = intent.action
    val data = intent.data
    // всегда валидировать входящие данные
}
```

**`android:exported`** — с Android 12 обязателен для всех компонентов с Intent Filter. `true` — компонент доступен другим приложениям, `false` — только внутри своего.

**Проверять что есть обработчик** перед запуском неявного Intent, иначе `ActivityNotFoundException`:

```kotlin
if (intent.resolveActivity(packageManager) != null) {
    startActivity(intent)
}
// или через try-catch
```

---

## Частые ловушки

**1. Большие данные в Extras** — `TransactionTooLargeException`. Передавать только id, данные хранить в БД или ViewModel.

**2. Bitmap в Intent** — категорически нельзя. Передавать через URI или сохранять во временный файл через FileProvider.

**3. Неявный Intent без проверки обработчика** — `ActivityNotFoundException` на устройствах где нет нужного приложения.

**4. PendingIntent без FLAG_IMMUTABLE** — на Android 12+ крашится без этого флага.

**5. Утеря данных при пересоздании Activity** — данные из `getIntent()` остаются старыми при повороте экрана. Хранить состояние в ViewModel, а не только в Intent.