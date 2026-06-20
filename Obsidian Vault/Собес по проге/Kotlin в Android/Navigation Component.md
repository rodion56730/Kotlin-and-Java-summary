**Navigation Component** — это часть Jetpack, которая управляет переходами между экранами. Вместо ручного вызова `FragmentTransaction` или `startActivity`, ты описываешь граф переходов, а библиотека берёт на себя Back Stack, передачу аргументов и анимации.

---

# Navigation Component

## 1. Три кита Navigation

1. **NavGraph** — XML-файл (или код), который описывает все экраны и переходы между ними.
2. **NavHost** — контейнер в разметке, который показывает текущий экран из графа.
3. **NavController** — объект, через который ты командуешь: "перейди туда", "вернись назад".

```
NavGraph (карта дорог)
    ↓
NavHost (экран навигатора)
    ↓
NavController (водитель)
```

---

## 2. Настройка: NavHost в XML

```xml
<!-- activity_main.xml -->
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/nav_host_fragment"
    app:defaultNavHost="true"
    app:navGraph="@navigation/nav_graph" />
```

`defaultNavHost="true"` — значит этот NavHost будет перехватывать нажатие кнопки "Назад".

---

## 3. NavGraph: описание экранов

```xml
<!-- res/navigation/nav_graph.xml -->
<navigation
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">

    <fragment android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment">
        <!-- Переход на Detail с аргументом -->
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment" />
    </fragment>

    <fragment android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment">
        <!-- Аргумент, который принимает этот экран -->
        <argument
            android:name="itemId"
            app:argType="integer" />
    </fragment>
</navigation>
```

---

## 4. Навигация из кода

```kotlin
// Получить NavController (внутри Fragment)
val navController = findNavController()

// Перейти на Detail, передав аргумент
navController.navigate(
    HomeFragmentDirections.actionHomeToDetail(itemId = 42)
)

// Вернуться назад
navController.popBackStack()

// Вернуться на конкретный экран и очистить стек до него
navController.popBackStack(R.id.homeFragment, inclusive = false)
```

> **`Directions`** — это автогенерируемые классы от Safe Args. Они дают типобезопасную передачу аргументов: компилятор проверит типы вместо тебя.

---

## 5. Safe Args (Типобезопасная передача данных)

Без Safe Args аргументы передаются через `Bundle` вручную — легко ошибиться с ключом или типом. Safe Args генерирует классы автоматически.

**Подключение в `build.gradle`:**

```kotlin
plugins {
    id("androidx.navigation.safeargs.kotlin")
}
```

**Отправка:**

```kotlin
// HomeFragment → DetailFragment
val action = HomeFragmentDirections.actionHomeToDetail(itemId = 42)
findNavController().navigate(action)
```

**Получение:**

```kotlin
// DetailFragment
val args: DetailFragmentArgs by navArgs()
val itemId = args.itemId // Int, не nullable, тип гарантирован компилятором
```

---

## 6. Deep Links

Navigation позволяет открыть конкретный экран по URI — из уведомления, из браузера, из другого приложения.

```xml
<fragment android:id="@+id/detailFragment" ...>
    <deepLink app:uri="myapp://detail/{itemId}" />
</fragment>
```

```kotlin
// В уведомлении:
val deepLinkIntent = NavDeepLinkBuilder(context)
    .setGraph(R.navigation.nav_graph)
    .setDestination(R.id.detailFragment)
    .setArguments(bundleOf("itemId" to 42))
    .createPendingIntent()
```

---

## 7. Navigation в Jetpack Compose

В Compose нет XML-графа — граф описывается кодом.

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onItemClick = { id ->
                    navController.navigate("detail/$id")
                }
            )
        }
        composable(
            route = "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.IntType })
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getInt("itemId") ?: 0
            DetailScreen(itemId = itemId)
        }
    }
}
```

> Передача NavController напрямую в composable — антипаттерн. Правильнее передавать лямбды: `onItemClick: (Int) -> Unit`. Так экран не знает о навигации и проще тестируется.

---

## 8. Back Stack и результат обратно

Иногда нужно передать данные с экрана назад (например, пользователь выбрал что-то в диалоге).

```kotlin
// Экран B: кладём результат перед закрытием
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set("selected_item", "Kotlin")
navController.popBackStack()

// Экран A: подписываемся на результат
navController.currentBackStackEntry
    ?.savedStateHandle
    ?.getLiveData<String>("selected_item")
    ?.observe(viewLifecycleOwner) { result ->
        println("Выбрано: $result")
    }
```

---

## Сводная таблица

|**Компонент**|**Что делает**|
|---|---|
|`NavGraph`|Карта всех экранов и переходов|
|`NavHost`|Контейнер, показывающий текущий экран|
|`NavController`|Управляет переходами|
|`Safe Args`|Типобезопасная передача аргументов|
|`Directions`|Автогенерируемые классы для переходов|
|`Deep Link`|Открытие конкретного экрана по URI|
|`SavedStateHandle`|Передача результата назад по стеку|

---

## Ключевые фразы для собеседования

- **"NavController управляет Back Stack автоматически"** — не нужно вручную добавлять транзакции в стек.
- **"Safe Args вместо Bundle вручную"** — типобезопасность на этапе компиляции.
- **"Не передавай NavController в Composable — передавай лямбды"** — принцип разделения ответственности.
- **"Deep Link позволяет открыть любой экран из уведомления или браузера"** — без написания дополнительного кода в Activity.
- **"`popBackStack(destination, inclusive)`** — вернуться на конкретный экран, очистив стек до него.