**Extension Functions (Функции-расширения)** — это одна из самых "магических" фишек Kotlin. Они позволяют добавлять новые методы в существующие классы, даже если у вас нет доступа к их исходному коду (например, к стандартным классам Java `String`, `List` или сторонним библиотекам).

При этом вы не меняете сам класс и не используете наследование.

---

# Extension Functions (Функции-расширения)

## 1. Базовый синтаксис

Чтобы создать функцию-расширение, нужно указать **имя класса** (Receiver type) перед именем функции.

```Kotlin
// Мы добавляем метод к стандартному классу String
fun String.removeFirstAndLast(): String {
    return this.substring(1, this.length - 1)
}

fun main() {
    val myString = "Hello"
    // Теперь у любой строки "появился" этот метод
    println(myString.removeFirstAndLast()) // Выведет: ell
}
```

---

## 2. Как это работает "под капотом"?

Важно понимать: Kotlin **не вносит** изменения в реальный класс.

- Это синтаксический сахар. Компилятор создает обычную статическую функцию, которая принимает объект класса как первый аргумент.
    
- В Java это выглядело бы так: `StringUtil.removeFirstAndLast(myString)`.
    
- **Следовательно:** Расширения не имеют доступа к `private` или `protected` членам класса. Только к публичным.
    

---

## 3. Статическое разрешение (Важный нюанс)

Функции-расширения выбираются **статически** на этапе компиляции, а не динамически (в отличие от обычных методов при наследовании).

```Kotlin
open class Shape
class Circle : Shape()

fun Shape.getName() = "Shape"
fun Circle.getName() = "Circle"

fun printName(s: Shape) {
    println(s.getName()) 
}

fun main() {
    printName(Circle()) // ВЫВЕДЕТ "Shape", а не "Circle"!
}
```

> **Правило:** Если в классе уже есть метод с такой же сигнатурой (имя и параметры), расширение никогда не будет вызвано. Приоритет всегда у "родного" метода.

---

## 4. Расширения для Nullable типов

Вы можете написать расширение для типов, которые могут быть `null`. Это позволяет вызывать методы через `.`вместо `?.`.

```Kotlin
fun Any?.toStringOrEmpty(): String {
    if (this == null) return "Empty"
    return toString()
}

val s: String? = null
println(s.toStringOrEmpty()) // Выведет "Empty" без NPE!
```

---

## 5. Extension Properties (Свойства-расширения)

По аналогии с функциями можно добавлять и свойства. Но так как поля в класс не добавляются, у свойств-расширений **обязательно** должен быть геттер.


```Kotlin
val String.lastChar: Char
    get() = this[length - 1]

println("Kotlin".lastChar) // 'n'
```

---

## 6. Пример кода для Obsidian

```Kotlin
// 1. Расширение для форматирования валюты
fun Double.toMoney(): String = "$${String.format("%.2f", this)}"

// 2. Расширение для списка (поменять местами элементы)
fun <T> MutableList<T>.swap(index1: Int, index2: Int) {
    val tmp = this[index1]
    this[index1] = this[index2]
    this[index2] = tmp
}

fun main() {
    val price = 19.99
    println(price.toMoney()) // "$19.99"

    val list = mutableListOf(1, 2, 3)
    list.swap(0, 2)
    println(list) // [3, 2, 1]
}
```

---

### Ключевые моменты для конспекта:

- **Краткость**: Позволяет избавиться от громоздких классов `StringUtils`, `DateUtils`.
    
- **Читаемость**: Код читается слева направо (`data.toFormat()`), а не изнутри наружу (`Utils.toFormat(data)`).
    
- **Инкапсуляция**: Не дает доступа к скрытым данным класса (только `public`).
    
- **Область видимости**: Чтобы использовать расширение в другом файле, его нужно импортировать.
    

---

