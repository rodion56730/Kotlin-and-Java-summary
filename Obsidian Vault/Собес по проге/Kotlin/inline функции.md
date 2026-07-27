## Inline-функции в Kotlin

`inline` — модификатор функции, указывающий компилятору **вставить тело функции прямо в место вызова** вместо создания объекта функции и вызова через стек. Особенно актуален для функций с лямбда-параметрами.

---

## Зачем это нужно

В Kotlin каждая лямбда — это объект. При обычном вызове функции с лямбдой:

1. Создаётся объект лямбды в heap
2. Происходит виртуальный вызов метода `invoke()`
3. Дополнительная нагрузка на GC

```kotlin
// Обычная функция — лямбда создаёт объект при каждом вызове
fun doSomething(action: () -> Unit) {
    action()
}

// Вызов
doSomething { println("Hello") } // создаётся анонимный объект
```

С `inline` компилятор просто вставляет тело функции и тело лямбды на место вызова — никаких объектов не создаётся:

```kotlin
inline fun doSomething(action: () -> Unit) {
    action()
}

// После компиляции выглядит примерно так:
// println("Hello")  ← просто вставился код
```

---

## noinline

Если функция `inline` имеет несколько лямбда-параметров, но один из них нужно **не** инлайнить (например, сохранить в переменную или передать дальше) — помечается `noinline`:

```kotlin
inline fun doWork(
    inlinedAction: () -> Unit,
    noinline savedAction: () -> Unit  // этот останется объектом
) {
    inlinedAction()
    handler.post(savedAction) // нельзя передать инлайн-лямбду как объект
}
```

---

## crossinline

Запрещает `return` из лямбды, когда лямбда вызывается в другом контексте (например, внутри другой лямбды или Runnable). Без `crossinline` можно случайно написать `return` который выйдет из внешней функции — `crossinline` это запрещает:

```kotlin
inline fun runOnMain(crossinline action: () -> Unit) {
    Handler(Looper.getMainLooper()).post {
        action() // вызывается внутри другой лямбды
        // без crossinline компилятор не знает из чего делать return
    }
}

// Использование
runOnMain {
    println("на главном потоке")
    // return здесь запрещён — crossinline не позволит
}
```

---

## reified — главная суперсила inline

Обычно тип-параметр дженерика в рантайме **стёрт** (type erasure) — нельзя написать `T::class` или `is T`. Но с `inline` + `reified` тип становится доступен в рантайме:

```kotlin
// Без inline — не скомпилируется
fun <T> getService(): T {
    return getSystemService(T::class.java) // ошибка — T неизвестен в рантайме
}

// С inline + reified — работает
inline fun <reified T> Context.getService(): T {
    return getSystemService(T::class.java) as T
}

// Использование — чисто и без передачи класса вручную
val manager = context.getService<LocationManager>()
```

Именно так работают многие функции Gson, Retrofit, стандартной библиотеки Kotlin:

```kotlin
// Под капотом используют reified
val intent = Intent(context, MainActivity::class.java)
// можно написать так благодаря inline + reified:
inline fun <reified T : Activity> Context.startActivity() {
    startActivity(Intent(this, T::class.java))
}
context.startActivity<MainActivity>()
```

---

## Когда использовать inline

|Ситуация|Использовать inline?|
|---|---|
|Функция с лямбда-параметром вызывается часто|Да|
|Нужен `reified` тип-параметр|Обязательно|
|Функция большая и вызывается редко|Нет — раздует bytecode|
|Лямбду нужно сохранить или передать дальше|Нет (или `noinline`)|
|Лямбда вызывается в другом контексте|`crossinline`|

---

## Коротко о non-local return

Обычный `return` внутри лямбды возвращает только из лямбды. Но в `inline`-функции можно написать `return` который выйдет из **вызывающей** функции — это называется non-local return:

```kotlin
inline fun forEach(list: List<Int>, action: (Int) -> Unit) {
    for (item in list) action(item)
}

fun findFirst() {
    forEach(listOf(1, 2, 3)) { item ->
        if (item == 2) return // выходит из findFirst(), не из лямбды!
        println(item)
    }
    println("не выполнится если нашли 2")
}
```

Именно поэтому можно писать `return` внутри `forEach`, `filter`, `map` из стандартной библиотеки Kotlin — они все `inline`.