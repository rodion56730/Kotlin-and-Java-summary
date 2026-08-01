# Interop: Compose ↔ View System

## Зачем нужен Interop

Большинство проектов переходят на Compose постепенно. Нельзя переписать всё сразу — нужно встраивать Compose во View-экраны и наоборот. Именно для этого существует Interop API.

---

## AndroidView — View внутри Compose

Встраивает любой классический View в Compose-дерево. Нужен когда Compose-аналога нет или нет смысла переписывать сложный кастомный View.

```kotlin
@Composable
fun WebViewScreen(url: String) {
    AndroidView(
        factory = { context ->
            // Создаётся один раз — аналог onCreateView
            WebView(context).apply {
                settings.javaScriptEnabled = true
                webViewClient = WebViewClient()
            }
        },
        update = { webView ->
            // Вызывается при рекомпозиции — аналог onBindViewHolder
            webView.loadUrl(url)
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

**Типичные случаи использования:**

- `WebView` — нет Compose-аналога
- `MapView` (Google Maps) — официальный SDK ещё View-based
- `SurfaceView` / `TextureView` — для видео, камеры
- Сложные кастомные View которые нет смысла переписывать
- Сторонние библиотеки без Compose-поддержки

```kotlin
// Google Maps в Compose
@Composable
fun MapScreen() {
    val mapView = rememberMapViewWithLifecycle()

    AndroidView(
        factory = { mapView },
        update = { map ->
            map.getMapAsync { googleMap ->
                googleMap.moveCamera(CameraUpdateFactory.newLatLngZoom(LatLng(55.75, 37.61), 12f))
            }
        }
    )
}

// Привязка MapView к жизненному циклу
@Composable
fun rememberMapViewWithLifecycle(): MapView {
    val context = LocalContext.current
    val mapView = remember { MapView(context) }
    val lifecycle = LocalLifecycleOwner.current.lifecycle

    DisposableEffect(lifecycle) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_CREATE -> mapView.onCreate(Bundle())
                Lifecycle.Event.ON_START -> mapView.onStart()
                Lifecycle.Event.ON_RESUME -> mapView.onResume()
                Lifecycle.Event.ON_PAUSE -> mapView.onPause()
                Lifecycle.Event.ON_STOP -> mapView.onStop()
                Lifecycle.Event.ON_DESTROY -> mapView.onDestroy()
                else -> {}
            }
        }
        lifecycle.addObserver(observer)
        onDispose { lifecycle.removeObserver(observer) }
    }

    return mapView
}
```

---

## AndroidViewBinding — ViewBinding внутри Compose

Если нужно встроить целый XML-макет с ViewBinding:

```kotlin
@Composable
fun LegacyFormScreen() {
    AndroidViewBinding(FragmentLegacyFormBinding::inflate) { binding ->
        // binding — уже созданный ViewBinding
        binding.nameInput.setText("Иван")
        binding.submitButton.setOnClickListener {
            // обработка
        }
    }
}
```

---

## ComposeView — Compose внутри View

Встраивает Compose в классический View-экран. Используется при постепенной миграции — заменяем части XML на Compose.

**В XML-макете:**

```xml
<LinearLayout ...>
    <TextView android:id="@+id/title" ... />

    <!-- Compose-контент здесь -->
    <androidx.compose.ui.platform.ComposeView
        android:id="@+id/compose_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
</LinearLayout>
```

**В Fragment/Activity:**

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)

    binding.composeView.apply {
        // Привязываем к жизненному циклу Fragment для корректной работы
        setViewCompositionStrategy(
            ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
        )
        setContent {
            MyAppTheme {
                // Compose-контент
                UserCard(user = viewModel.user)
            }
        }
    }
}
```

---

## ViewCompositionStrategy — важная настройка

Определяет когда Compose-дерево уничтожается. Неправильная стратегия ведёт к утечкам памяти или преждевременному уничтожению.

```kotlin
// DisposeOnDetachedFromWindow (по умолчанию)
// — уничтожает Compose когда View отсоединяется от окна
// Проблема: Fragment хранит View даже в backstack — Compose может уничтожиться

// DisposeOnViewTreeLifecycleDestroyed — РЕКОМЕНДУЕТСЯ для Fragment
// — уничтожает когда уничтожается ViewTreeLifecycle (onDestroyView)
setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)

// DisposeOnLifecycleDestroyed — для Activity или кастомных случаев
setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnLifecycleDestroyed(lifecycle))
```

---

## Передача данных между View и Compose

**ViewModel — лучший способ** — и View и Compose могут наблюдать один и тот же ViewModel:

```kotlin
// Fragment (View-based)
class ProfileFragment : Fragment() {
    private val viewModel: ProfileViewModel by viewModels()

    override fun onViewCreated(...) {
        // View-часть наблюдает через LiveData
        viewModel.user.observe(viewLifecycleOwner) { user ->
            binding.nameText.text = user.name
        }

        // Compose-часть тоже наблюдает тот же ViewModel
        binding.composeView.setContent {
            val user by viewModel.user.observeAsState()
            user?.let { UserAvatar(it) }
        }
    }
}
```

---

## AbstractComposeView — кастомный View на Compose

Создать кастомный View который внутри рендерится через Compose. Полезно для переиспользования Compose-компонентов в View-мире:

```kotlin
class UserAvatarView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : AbstractComposeView(context, attrs) {

    var user by mutableStateOf<User?>(null)

    @Composable
    override fun Content() {
        MyAppTheme {
            user?.let { UserAvatar(it) }
        }
    }
}

// Использование в XML
// <com.example.UserAvatarView android:id="@+id/avatar" ... />
// binding.avatar.user = currentUser
```

---

## Частые ловушки

**1. Отсутствие темы в ComposeView** — Compose не наследует тему из Android автоматически. Всегда оборачивать в `MyAppTheme { }`:

```kotlin
composeView.setContent {
    MyAppTheme { // обязательно!
        MyComposable()
    }
}
```

**2. Неправильная ViewCompositionStrategy во Fragment** — использование стратегии по умолчанию в Fragment приводит к тому что Compose уничтожается при уходе во backstack. Всегда использовать `DisposeOnViewTreeLifecycleDestroyed`.

**3. AndroidView и рекомпозиция** — блок `update` вызывается при каждой рекомпозиции. Тяжёлые операции внутри `update` замедлят UI. Использовать `key` чтобы пересоздавать View только когда действительно нужно.

**4. Состояние View не сохраняется автоматически** — внутри AndroidView нет автоматического `rememberSaveable`. Нужно самостоятельно сохранять состояние View (например, позицию скролла):

```kotlin
var savedState by rememberSaveable { mutableStateOf<Parcelable?>(null) }

AndroidView(
    factory = { RecyclerView(it) },
    update = { recyclerView ->
        savedState?.let { recyclerView.layoutManager?.onRestoreInstanceState(it) }
    }
)
```