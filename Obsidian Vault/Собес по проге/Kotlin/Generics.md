Generics в Kotlin во многом похожи на Java, но решают главную боль: громоздкий синтаксис вайлдкардов (`? extends T` и `? super T`). В Kotlin это заменено на концепцию **Declaration-site variance** (объявление вариантности прямо в классе).

---

# Generics в Kotlin: Вариантность и Reified

Обобщения (Generics) позволяют классам и методам работать с различными типами данных, сохраняя при этом типобезопасность.

## 1. Базовый синтаксис

Классы и функции объявляются с использованием угловых скобок `<T>`.

```Kotlin
class Box<T>(val item: T)

val intBox = Box(10)      // Тип Int выведен автоматически
val stringBox = Box("Hi") // Тип String
```

---

## 2. Вариантность: `out` и `in` (Ключевое отличие от Java)

В Java мы пишем `List<? extends Number>`. В Kotlin мы используем ключевые слова `out` и `in`.

### Ковариантность (`out`) — "Только чтение"

Используется, когда класс только **отдает** данные типа `T`.

- **Аналог Java:** `? extends T`.
    
- **Правило:** Мы можем присвоить `Producer<Dog>` переменной `Producer<Animal>`.
    

```Kotlin
interface Producer<out T> {
    fun produce(): T // T может быть только результатом
}
```

### Контравариантность (`in`) — "Только запись"

Используется, когда класс только **принимает** данные типа `T`.

- **Аналог Java:** `? super T`.
    
- **Правило:** Мы можем присвоить `Consumer<Animal>` переменной `Consumer<Dog>`.


```Kotlin
interface Consumer<in T> {
    fun consume(item: T) // T может быть только аргументом
}
```

---

## 3. Ограничения типов (Generic Constraints)

Чтобы ограничить типы, которые можно передать в Generic, используется двоеточие (аналог `extends` в Java). По умолчанию лимит — `Any?`.


```Kotlin
// Только подтипы Number могут быть переданы
fun <T : Number> sort(list: List<T>) { ... }
```

---

## 4. Reified Type Parameters (Магия Inline)

В Java и Kotlin типы дженериков "стираются" в рантайме. Вы не можете проверить `if (T is String)`. Но в Kotlin есть ключевое слово `reified`, которое работает вместе с `inline` функциями.

**Это позволяет получить доступ к типу внутри функции.**


```Kotlin
inline fun <reified T> isType(value: Any): Boolean {
    return value is T // В Java так сделать нельзя!
}

fun main() {
    println(isType<String>("Hello")) // true
    println(isType<Int>("Hello"))    // false
}
```

---

## 5. Star-projections (`*`)

Используется, когда вы ничего не знаете о типе, но хотите использовать его безопасно.

- **Аналог Java:** `List<?>`.
    
- `List<*>` — это список чего-то, но мы можем только читать из него объекты типа `Any?`.
    

---

## 6. Пример кода

```Kotlin
// 1. Ковариантный интерфейс (только отдает)
interface Source<out T> {
    fun nextT(): T
}

fun demo(strs: Source<String>) {
    val objects: Source<Any> = strs // ОК, так как T помечен out
}

// 2. Функция с Reified (самый частый кейс — логика типов)
inline fun <reified T> Any.asTypeOrNull(): T? {
    return if (this is T) this else null
}

fun main() {
    val anyList: Any = listOf(1, 2, 3)
    val result = anyList.asTypeOrNull<List<Int>>()
    println(result) // [1, 2, 3]
}
```

---

### Сводная таблица для конспекта

|**Понятие**|**Kotlin**|**Java (аналог)**|**Суть**|
|---|---|---|---|
|**Ковариантность**|`<out T>`|`<? extends T>`|Можем только читать `T`|
|**Контравариантность**|`<in T>`|`<? super T>`|Можем только записывать `T`|
|**Инвариантность**|`<T>`|`<T>`|И читать, и писать (строгий тип)|
|**Стирание типов**|`*`|`?`|Неизвестный тип|
|**Доступ к типу**|`reified`|Нет аналога|Тип доступен в рантайме (только в `inline`)|

---

