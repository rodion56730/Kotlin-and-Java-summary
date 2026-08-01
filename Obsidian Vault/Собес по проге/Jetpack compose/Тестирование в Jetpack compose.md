# Тестирование в Jetpack Compose

## Зависимости

```kotlin
// build.gradle
androidTestImplementation("androidx.compose.ui:ui-test-junit4")
debugImplementation("androidx.compose.ui:ui-test-manifest")

// Для unit-тестов ViewModel + Coroutines
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test")
testImplementation("app.cash.turbine:turbine:1.0.0") // тестирование Flow
```

---

## ComposeTestRule — основа тестирования

```kotlin
class MyScreenTest {

    // Запускает Activity с пустым контентом — ты сам устанавливаешь composable
    @get:Rule
    val composeTestRule = createComposeRule()

    // Запускает конкретную Activity
    @get:Rule
    val activityRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun buttonClick_updatesText() {
        // Устанавливаем контент
        composeTestRule.setContent {
            MyAppTheme {
                CounterScreen()
            }
        }

        // Находим элемент и проверяем
        composeTestRule.onNodeWithText("Count: 0").assertIsDisplayed()

        // Кликаем
        composeTestRule.onNodeWithText("Увеличить").performClick()

        // Проверяем результат
        composeTestRule.onNodeWithText("Count: 1").assertIsDisplayed()
    }
}
```

---

## Поиск узлов (Finders)

```kotlin
// По тексту
composeTestRule.onNodeWithText("Привет")
composeTestRule.onNodeWithText("Привет", ignoreCase = true)
composeTestRule.onAllNodesWithText("Элемент") // все узлы с таким текстом

// По contentDescription (иконки, изображения)
composeTestRule.onNodeWithContentDescription("Кнопка назад")

// По тегу — явная метка для тестов
Modifier.testTag("submit_button") // в коде
composeTestRule.onNodeWithTag("submit_button") // в тесте

// По семантическому свойству
composeTestRule.onNode(hasText("OK") and isEnabled())
composeTestRule.onNode(hasClickAction())

// Дочерние элементы
composeTestRule
    .onNodeWithTag("user_list")
    .onChildren()
    .assertCountEquals(5)

composeTestRule
    .onNodeWithTag("user_list")
    .onChildAt(0)
    .assertTextContains("Иван")
```

---

## Матчеры (Matchers)

```kotlin
// Комбинирование матчеров
onNode(hasText("OK") and isEnabled())
onNode(hasText("Отмена") or hasText("Cancel"))
onNode(!isEnabled()) // отрицание

// Доступные матчеры
hasText("текст")
hasContentDescription("описание")
hasTestTag("тег")
isEnabled() / isNotEnabled()
isDisplayed() / isNotDisplayed()
isSelected() / isNotSelected()
isFocused()
hasClickAction()
isRoot()
isDialog()
hasParent(matcher)
hasAnyChild(matcher)
hasAnyDescendant(matcher)
```

---

## Действия (Actions)

```kotlin
// Клик
composeTestRule.onNodeWithText("Кнопка").performClick()

// Ввод текста
composeTestRule.onNodeWithTag("email_field").performTextInput("test@example.com")

// Очистка и ввод
composeTestRule.onNodeWithTag("email_field")
    .performTextClearance()
    .performTextInput("new@example.com")

// Прокрутка
composeTestRule.onNodeWithTag("list").performScrollToIndex(10)
composeTestRule.onNodeWithTag("scroll").performScrollTo()

// Свайп
composeTestRule.onNodeWithTag("card").performTouchInput {
    swipeLeft()
    swipeRight()
    swipeUp()
    swipeDown()
}

// Жест
composeTestRule.onNodeWithTag("map").performTouchInput {
    pinch(
        start0 = center - Offset(100f, 0f),
        end0 = center - Offset(200f, 0f),
        start1 = center + Offset(100f, 0f),
        end1 = center + Offset(200f, 0f)
    )
}
```

---

## Проверки (Assertions)

```kotlin
// Отображение
composeTestRule.onNodeWithText("Загрузка").assertIsDisplayed()
composeTestRule.onNodeWithText("Ошибка").assertDoesNotExist()

// Текст
composeTestRule.onNodeWithTag("title").assertTextEquals("Профиль")
composeTestRule.onNodeWithTag("title").assertTextContains("Проф")

// Состояние
composeTestRule.onNodeWithTag("button").assertIsEnabled()
composeTestRule.onNodeWithTag("button").assertIsNotEnabled()
composeTestRule.onNodeWithTag("checkbox").assertIsOn()
composeTestRule.onNodeWithTag("checkbox").assertIsOff()
composeTestRule.onNodeWithTag("item").assertIsSelected()

// Количество
composeTestRule.onAllNodesWithTag("list_item").assertCountEquals(5)
```

---

## Тестирование асинхронного кода

```kotlin
@Test
fun loadUsers_showsLoadingThenData() {
    composeTestRule.setContent {
        UserListScreen(viewModel = fakeViewModel)
    }

    // Загрузка отображается
    composeTestRule.onNodeWithTag("loading_indicator").assertIsDisplayed()

    // Ждём исчезновения загрузки (с таймаутом)
    composeTestRule.waitUntilDoesNotExist(
        matcher = hasTestTag("loading_indicator"),
        timeoutMillis = 5000
    )

    // Данные отображаются
    composeTestRule.onNodeWithText("Иван Иванов").assertIsDisplayed()
}

// waitUntil — ждём условия
composeTestRule.waitUntil(timeoutMillis = 3000) {
    composeTestRule.onAllNodesWithTag("list_item").fetchSemanticsNodes().size == 5
}
```

---

## Тестирование ViewModel с MVI

```kotlin
class ProfileViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule() // заменяет Dispatchers.Main на TestDispatcher

    private val fakeRepository = FakeUserRepository()
    private lateinit var viewModel: ProfileViewModel

    @Before
    fun setup() {
        viewModel = ProfileViewModel(GetUserUseCase(fakeRepository))
    }

    @Test
    fun loadProfile_success_updatesState() = runTest {
        // Turbine — тестирование Flow
        viewModel.state.test {
            // Начальное состояние
            val initial = awaitItem()
            assertFalse(initial.isLoading)
            assertNull(initial.user)

            // Запускаем загрузку
            viewModel.handleIntent(ProfileIntent.LoadProfile)

            // Состояние загрузки
            val loading = awaitItem()
            assertTrue(loading.isLoading)

            // Успешная загрузка
            val success = awaitItem()
            assertFalse(success.isLoading)
            assertEquals("Иван", success.user?.name)
        }
    }

    @Test
    fun loadProfile_error_showsError() = runTest {
        fakeRepository.shouldThrowError = true

        viewModel.state.test {
            awaitItem() // начальное
            viewModel.handleIntent(ProfileIntent.LoadProfile)
            awaitItem() // loading
            val error = awaitItem()
            assertNotNull(error.error)
            assertFalse(error.isLoading)
        }
    }
}

// MainDispatcherRule для замены Main dispatcher
class MainDispatcherRule : TestWatcher() {
    private val testDispatcher = UnconfinedTestDispatcher()
    override fun starting(description: Description) { Dispatchers.setMain(testDispatcher) }
    override fun finished(description: Description) { Dispatchers.resetMain() }
}
```

---

## Семантика — основа тестируемости

Compose использует семантическое дерево для тестирования. Каждый компонент имеет семантические свойства (текст, contentDescription, role и т.д.). Кастомные компоненты нужно аннотировать явно:

```kotlin
// Добавляем семантику для кастомного компонента
Box(
    modifier = Modifier
        .testTag("rating_bar")
        .semantics {
            contentDescription = "Рейтинг: $rating из 5"
            stateDescription = if (rating > 3) "Высокий" else "Низкий"
        }
)

// Скрыть от семантики (декоративные элементы)
Icon(
    imageVector = Icons.Default.Star,
    contentDescription = null, // null = скрыто от accessibility и тестов
    modifier = Modifier.semantics { invisibleToUser() }
)
```

---

## Частые ловушки

**1. Тест флакает из-за анимаций** — по умолчанию в тестах анимации работают. Отключить:

```kotlin
composeTestRule.mainClock.autoAdvance = false
composeTestRule.mainClock.advanceTimeBy(1000) // прокрутить время вручную
```

**2. onNodeWithText не находит текст** — текст может быть разбит на несколько узлов. Использовать `useUnmergedTree = true` или матчер `hasText` с `substring = true`.

**3. assertIsDisplayed vs assertExists** — `assertExists` проверяет что узел есть в дереве (даже за экраном), `assertIsDisplayed` — что он видим пользователю.

**4. Тестирование без testTag — хрупкие тесты** — поиск по тексту ломается при изменении копирайта. Всегда добавлять `testTag` к ключевым элементам.

**5. setContent без темы** — кастомные цвета и стили не применятся без обёртки в тему:

```kotlin
composeTestRule.setContent {
    MyAppTheme { // обязательно
        MyScreen()
    }
}
```