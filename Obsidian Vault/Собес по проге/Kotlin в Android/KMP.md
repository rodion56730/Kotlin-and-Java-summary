
---

## Kotlin Multiplatform (KMP)

---

## Основная идея

KMP позволяет писать общий Kotlin-код, который компилируется под разные платформы: Android, iOS, Web, Desktop. Не нужно писать одну и ту же бизнес-логику дважды на Kotlin и Swift — пишется один раз в `commonMain`.

```
commonMain (общий код)
    ├── androidMain (Android-специфичный)
    ├── iosMain (iOS-специфичный)
    └── desktopMain (Desktop-специфичный)
```

---

## Что можно шарить

- Бизнес-логика, UseCase
- Модели данных
- Сетевой слой (Ktor)
- Локальная БД (SQLDelight)
- Валидация
- Утилиты, форматирование дат

**Что остаётся платформенным:**

- UI (Compose на Android/Desktop, SwiftUI на iOS)
- Работа с камерой, геолокацией, Bluetooth
- Push-уведомления
- Платёжные системы

---

## expect / actual — ключевой механизм

Когда нужна платформенная реализация — объявляется `expect` в общем коде и `actual` на каждой платформе:

```kotlin
// commonMain — объявляем контракт
expect class PlatformLogger() {
    fun log(message: String)
}

expect fun getCurrentTimeMillis(): Long
```

```kotlin
// androidMain — Android реализация
actual class PlatformLogger actual constructor() {
    actual fun log(message: String) {
        Log.d("APP", message)
    }
}

actual fun getCurrentTimeMillis(): Long = System.currentTimeMillis()
```

```kotlin
// iosMain — iOS реализация
actual class PlatformLogger actual constructor() {
    actual fun log(message: String) {
        println(message) // NSLog через println в Kotlin/Native
    }
}

actual fun getCurrentTimeMillis(): Long =
    NSDate().timeIntervalSince1970.toLong() * 1000
```

---

## Ktor — сетевой слой в KMP

Кроссплатформенный HTTP-клиент, работает в commonMain:

```kotlin
// commonMain
class UserRemoteDataSource(private val client: HttpClient) {

    suspend fun getUser(id: String): UserDto {
        return client.get("https://api.example.com/users/$id").body()
    }
}

// Создание клиента (commonMain)
fun createHttpClient() = HttpClient {
    install(ContentNegotiation) {
        json(Json { ignoreUnknownKeys = true })
    }
    install(Logging) {
        level = LogLevel.BODY
    }
}
```

Под капотом Ktor использует разные движки на каждой платформе: OkHttp на Android, NSURLSession на iOS.

---

## SQLDelight — база данных в KMP

Кроссплатформенная БД, генерирует типобезопасный Kotlin-код из SQL:

```sql
-- commonMain/sqldelight/User.sq
CREATE TABLE User (
    id TEXT NOT NULL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL
);

getUser:
SELECT * FROM User WHERE id = ?;

insertUser:
INSERT INTO User VALUES (?, ?, ?);
```

```kotlin
// Использование в commonMain
class UserLocalDataSource(private val db: AppDatabase) {
    fun getUser(id: String): User? =
        db.userQueries.getUser(id).executeAsOneOrNull()

    fun insertUser(user: User) =
        db.userQueries.insertUser(user.id, user.name, user.email)
}
```

---

## Структура KMP проекта

```
shared/
├── commonMain/kotlin/
│   ├── domain/
│   │   ├── model/User.kt
│   │   ├── repository/UserRepository.kt
│   │   └── usecase/GetUserUseCase.kt
│   └── data/
│       ├── remote/UserRemoteDataSource.kt
│       └── local/UserLocalDataSource.kt
├── androidMain/kotlin/
│   └── di/AndroidModule.kt
└── iosMain/kotlin/
    └── di/IosModule.kt

androidApp/   ← Android UI (Compose)
iosApp/       ← iOS UI (SwiftUI)
```

---

## Корутины в KMP

В commonMain корутины работают как обычно. Проблема возникает при вызове suspend-функций из Swift — Swift не понимает корутины напрямую.

**Решение — `kotlinx-coroutines-core` с `@ObjCName` и обёртки:**

```kotlin
// Обёртка для iOS — превращает suspend в callback
class UserViewModelIos(
    private val getUserUseCase: GetUserUseCase
) {
    fun getUser(id: String, onSuccess: (User) -> Unit, onError: (String) -> Unit) {
        MainScope().launch {
            try {
                val user = getUserUseCase(id)
                onSuccess(user)
            } catch (e: Exception) {
                onError(e.message ?: "Error")
            }
        }
    }
}
```

Более современный подход — **KMP-NativeCoroutines** или **SKIE** (библиотека от Touchlab) — автоматически превращают suspend-функции в async/await для Swift.

---

## Частые ловушки

**1. Слишком много в commonMain**

```kotlin
// ПЛОХО — пытаться шарить UI логику специфичную для платформы
// commonMain не место для Android-специфичных паттернов
```

**2. Утечки памяти на iOS**

Kotlin/Native (iOS) использует другой garbage collector — **ARC совместимый**. Циклические ссылки между Kotlin-объектами могут приводить к утечкам. Избегать циклических зависимостей в общем коде.

**3. Многопоточность на iOS**

До Kotlin 1.9 в Kotlin/Native объекты нельзя было передавать между потоками без `freeze()`. Сейчас это ограничение снято, но старый код с `freeze()` ещё встречается.

**4. Платформенные зависимости в commonMain**

```kotlin
// ПЛОХО
import android.util.Log // не скомпилируется в iOS

// ХОРОШО — через expect/actual
expect fun platformLog(message: String)
```

---

## KMP vs Flutter vs React Native

![[Pasted image 20260728010420.png]]

---

## Вопросы на собеседовании

**«В чём отличие KMP от Flutter?»** Flutter шарит и логику и UI через собственный движок рендеринга. KMP шарит только логику, UI остаётся нативным на каждой платформе — Compose на Android, SwiftUI на iOS. KMP даёт лучший нативный UX, Flutter — больше переиспользования кода.

**«Что нельзя вынести в commonMain?»** Всё что зависит от платформенного API: UI-компоненты, работу с камерой, геолокацией, системными разрешениями. Для таких случаев используется expect/actual.

**«Как вызвать suspend-функцию из Swift?»** Нативно Swift не понимает корутины. Нужны либо обёртки с callback, либо библиотеки SKIE или KMP-NativeCoroutines которые генерируют Swift-friendly API автоматически.