
Начнём с главного сюрприза для тех, кто пришёл из Java.

---

## 1. Дефолтная видимость — главное отличие

```java
// Java — дефолт это package-private
class User {
    String name; // виден только внутри пакета
}
```

```kotlin
// Kotlin — дефолт это public
class User {
    val name: String = "" // виден ВЕЗДЕ
}
```

На собесе это спрашивают почти всегда. Ответ простой: в Kotlin дефолт — `public`, в Java — `package-private`.

---

## 2. Все модификаторы — сравнительная таблица---

## 3. internal — уникальность Kotlin

Это самый частый вопрос на собесе по этой теме.

```kotlin
// Модуль A (например, :feature-auth)
internal class TokenManager {
    internal fun getToken(): String = "..."
}

// Модуль B (например, :feature-profile)
// TokenManager отсюда НЕВИДИМ — другой модуль
```

Модуль в Kotlin = один Gradle-модуль (`:app`, `:core`, `:feature-login`).

Зачем нужен `internal`:

- Скрыть реализацию от других модулей
- Не засорять публичное API библиотеки
- Код виден внутри модуля для тестов и других классов, но не снаружи

В Java ничего подобного нет — там граница видимости только на уровне пакета, а не модуля.

---

## 4. private в Kotlin — шире чем в Java

В Java `private` работает только внутри класса. В Kotlin `private` можно поставить на уровне файла:

```kotlin
// файл Utils.kt

private fun helperFunction() { }  // видна только в этом файле

private val SECRET = "abc"        // тоже только в этом файле

class MyClass {
    // helperFunction() здесь доступна
}

class AnotherClass {
    // helperFunction() здесь тоже доступна — тот же файл
}
```

Это называется file-level private. В Java такого нет совсем.

---

## 5. protected — Kotlin строже Java

```java
// Java — protected виден в пакете тоже
package com.example;

class A {
    protected String name = "A";
}

// Другой класс в том же пакете — не наследник!
class B {
    void test() {
        A a = new A();
        System.out.println(a.name); // ✅ работает в Java
    }
}
```

```kotlin
// Kotlin — protected только для наследников
open class A {
    protected val name = "A"
}

class B {  // не наследник
    fun test() {
        val a = A()
        println(a.name)  // ❌ ошибка компиляции
    }
}

class C : A() {
    fun test() {
        println(name)  // ✅ наследник — видит
    }
}
```

---

## 6. Видимость в Android — реальные кейсы

```kotlin
// Repository — internal, снаружи модуля не нужен
internal class UserRepository(
    private val api: UserApi,       // private — только этот класс
    private val db: UserDao
) {
    internal suspend fun getUser(): User { ... }
}

// ViewModel — public, нужен из UI-слоя
class UserViewModel(
    private val repository: UserRepository  // internal доступен внутри модуля
) : ViewModel() {

    // _uiState — private, наружу только через uiState
    private val _uiState = MutableStateFlow(UiState())
    val uiState = _uiState.asStateFlow()  // public
}
```

Это классическая инкапсуляция StateFlow: мутабельный стейт `private`, снаружи отдаём только `asStateFlow()`.

---

## 7. Видимость для конструкторов

```kotlin
// Приватный конструктор — паттерн Singleton / Factory
class Database private constructor() {

    companion object {
        @Volatile
        private var instance: Database? = null

        fun getInstance(): Database {
            return instance ?: synchronized(this) {
                instance ?: Database().also { instance = it }
            }
        }
    }
}
```

---

## 8. Видимость для геттеров и сеттеров

В Kotlin можно сделать геттер публичным, а сеттер приватным:

```kotlin
class UserViewModel : ViewModel() {

    var isLoading: Boolean = false
        private set  // снаружи только читать, писать — только внутри класса

    fun loadUser() {
        isLoading = true  // ✅ внутри класса
        // ...
        isLoading = false
    }
}

// Снаружи:
viewModel.isLoading        // ✅ читать можно
viewModel.isLoading = true // ❌ ошибка компиляции
```

---

## 9. Что спрашивают на собесах

**Какой дефолтный модификатор в Kotlin?** `public`. В Java — `package-private`.

**Что такое `internal` и зачем он нужен?** Видимость в рамках одного Gradle-модуля. Позволяет скрыть детали реализации от других модулей, но не от своего. В Java аналога нет.

**Чем `protected` в Kotlin отличается от Java?** В Java `protected` = наследники + весь пакет. В Kotlin только наследники — пакет не видит.

**Как сделать поле только для чтения снаружи?** `private set` на свойстве:

```kotlin
var count = 0
    private set
```

---

## Шпаргалка

```
public    → все (дефолт в Kotlin)
private   → только класс / файл
protected → класс + наследники (без пакета — строже Java!)
internal  → весь Gradle-модуль (только в Kotlin)

package-private → только Java, дефолт там
```