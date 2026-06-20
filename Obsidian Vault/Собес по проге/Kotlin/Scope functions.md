
Это одна из самых частых тем на собесах по Kotlin. Главная сложность — их много и они похожи.

---

## 1. Что такое scope functions

Это функции из стандартной библиотеки Kotlin, которые выполняют блок кода **в контексте объекта**. Их 5: `let`, `run`, `with`, `apply`, `also`.

Все они делают одно и то же концептуально — дают доступ к объекту внутри блока. Разница в двух вещах:

- как объект доступен внутри блока — через `this` или `it`
- что возвращается — сам объект или результат блока---

## 2. let — null-check и трансформация

Объект доступен как `it`, возвращает результат блока.

```kotlin
// Главный кейс — безопасная работа с nullable
val user: User? = getUser()

user?.let {
    // it = user, и он точно не null здесь
    println(it.name)
    sendEmail(it.email)
}
```

```kotlin
// Трансформация — как map но для одного объекта
val name: String? = user?.let {
    "${it.firstName} ${it.lastName}"
}
```

```kotlin
// Реальный Android пример — работа с nullable view
binding.errorText?.let {
    it.text = "Что-то пошло не так"
    it.isVisible = true
}
```

Мнемоника: **let = дай мне объект как `it`, верни что скажу**.

---

## 3. apply — конфигурация объекта

Объект доступен как `this` (можно не писать), возвращает **сам объект**.

```kotlin
// Без apply
val intent = Intent()
intent.action = "android.intent.action.VIEW"
intent.data = Uri.parse("https://example.com")
intent.flags = Intent.FLAG_ACTIVITY_NEW_TASK
startActivity(intent)

// С apply — чище
val intent = Intent().apply {
    action = "android.intent.action.VIEW"
    data = Uri.parse("https://example.com")
    flags = Intent.FLAG_ACTIVITY_NEW_TASK
}
startActivity(intent)
```

```kotlin
// Реальный Android — настройка RecyclerView
recyclerView.apply {
    layoutManager = LinearLayoutManager(context)
    adapter = userAdapter
    addItemDecoration(DividerItemDecoration(context, VERTICAL))
    setHasFixedSize(true)
}
```

Мнемоника: **apply = настрой меня и верни меня**.

---

## 4. also — побочные эффекты в цепочке

Как `apply`, но объект доступен как `it`. Возвращает сам объект.

```kotlin
// Логгирование в цепочке без её разрыва
val user = repository.getUser()
    .also { log("Загружен пользователь: ${it.name}") }
    .also { analytics.track("user_loaded") }

// user здесь — это User, не результат блока
```

```kotlin
// Реальный кейс — дебаг Flow
flow
    .also { println("Flow создан") }
    .map { it.toUiModel() }
    .also { println("После map") }
    .collect { ... }
```

Разница `also` vs `apply`: когда хочешь обратиться к объекту по имени (`it`) чтобы не спутать с `this` внешнего класса.

---

## 5. run — инициализация с результатом

Объект доступен как `this`, возвращает результат блока. Гибрид `apply` + `let`.

```kotlin
val result = user.run {
    // this = user
    "$name, возраст: $age"  // возвращается эта строка
}
```

```kotlin
// Реальный кейс — вычисление с несколькими свойствами объекта
val isValid = form.run {
    email.isNotEmpty() && password.length >= 8 && username.isNotEmpty()
}
```

```kotlin
// run без объекта — просто блок кода с результатом
val token = run {
    val prefs = getSharedPreferences("auth", MODE_PRIVATE)
    prefs.getString("token", null) ?: generateToken()
}
```

---

## 6. with — группировка вызовов

Технически не extension функция, принимает объект как аргумент. Возвращает результат блока.

```kotlin
// Синтаксис отличается — объект передаётся аргументом
with(binding) {
    userName.text = user.name
    userEmail.text = user.email
    userAvatar.load(user.avatarUrl)
}
```

```kotlin
// Реальный кейс — заполнение ViewBinding
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    val user = users[position]
    with(holder.binding) {
        nameText.text = user.name
        emailText.text = user.email
        root.setOnClickListener { onUserClick(user) }
    }
}
```

`with` vs `apply`: `apply` возвращает объект (удобен при инициализации), `with` возвращает результат блока (удобен когда объект уже создан).

---

## 7. Цепочки и комбинирование

```kotlin
// Реальный production пример
val user = User()
    .apply {
        name = "Алексей"
        email = "alex@example.com"
    }
    .also {
        analytics.track("user_created", it.id)
    }

// null-safe цепочка
getUser()
    ?.let { mapToUiModel(it) }
    ?.also { cache.put(it) }
    ?: UiModel.empty()
```

---

## 8. Мнемоника для запоминания

```
         │  возвращает объект  │  возвращает результат
─────────┼─────────────────────┼──────────────────────
  it     │      also           │       let
  this   │      apply          │    run / with
```

Ещё проще:

```
apply  → настроить объект (конфиг)
also   → подсмотреть / залоггировать (side effect)
let    → трансформировать или null-check
run    → вычислить что-то используя объект
with   → сгруппировать вызовы на готовом объекте
```

---

## 9. Частые ошибки

```kotlin
// Ошибка — используют apply там где нужен let
val name = user.apply {
    // apply возвращает user, а не строку!
    name.uppercase()  // это выражение просто игнорируется
}
// name здесь типа User, а не String

// Правильно
val name = user.let {
    it.name.uppercase()
}
```

```kotlin
// Ошибка — this внутри apply конфликтует с внешним this
class MyFragment : Fragment() {
    fun setup() {
        textView.apply {
            text = "Hello"
            setOnClickListener {
                // this здесь — textView, не Fragment!
                // нельзя написать просто viewModel — нужно this@MyFragment.viewModel
                this@MyFragment.viewModel.onClick()
            }
        }
    }
}
```

---

## 10. Что спрашивают на собесах

**Чем `let` отличается от `apply`?** `let` — объект как `it`, возвращает результат блока. `apply` — объект как `this`, возвращает сам объект. `apply` для конфигурации, `let` для трансформации и null-check.

**Чем `also` отличается от `apply`?** Оба возвращают объект. Разница только в том, как объект доступен: `apply` через `this`, `also` через `it`. `also` удобен когда нужно явное имя чтобы не путать с `this` внешнего класса.

**Когда использовать `with` вместо `run`?** Когда объект уже существует и нужно сгруппировать несколько вызовов на нём. `run` — extension функция, `with` — обычная функция с аргументом. Разница скорее стилистическая.