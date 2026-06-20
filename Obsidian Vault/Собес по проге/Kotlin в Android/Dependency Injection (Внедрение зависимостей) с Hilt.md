Давай углубимся в **Hilt**. Чтобы конспект был по-настоящему «продвинутым», нужно разобрать не только базовые аннотации, но и то, как передавать параметры в конструкторы (Qualifier), как работать с интерфейсами (Binds) и как Hilt управляет памятью через компоненты.

---

# Глубокое погружение в Hilt

## 1. Иерархия компонентов и их жизненный цикл

Hilt не просто создает объекты, он привязывает их к «контейнерам» (Components), которые автоматически уничтожаются вместе с соответствующим объектом Android.

|**Компонент**|**Создается при...**|**Уничтожается при...**|**Scope (Аннотация)**|
|---|---|---|---|
|**SingletonComponent**|Запуске приложения|Остановке приложения|`@Singleton`|
|**ViewModelComponent**|Создании ViewModel|Очистке ViewModel|`@ViewModelScoped`|
|**ActivityComponent**|`onCreate()` Activity|`onDestroy()` Activity|`@ActivityScoped`|
|**FragmentComponent**|`onAttach()` фрагмента|`onDestroy()` фрагмента|`@FragmentScoped`|

> **Важно:** Зависимость из «верхнего» компонента (например, Singleton) можно внедрить в «нижний» (Activity), но **наоборот нельзя**.

---

## 2. Работа с интерфейсами: `@Binds` vs `@Provides`

Часто мы хотим внедрить не конкретный класс, а интерфейс. У Hilt есть два пути:

### А. @Binds (Элегантный путь)

Используется, когда у вас есть интерфейс и его реализация, и вы хотите сказать Hilt: «Когда просят `Storage`, дай `DatabaseStorage`».

- Должен быть в `abstract` модуле.
    
- Работает быстрее и генерирует меньше кода, чем `@Provides`.

```Kotlin
interface Storage { fun save() }

class DatabaseStorage @Inject constructor() : Storage {
    override fun save() { /* ... */ }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class StorageModule {
    @Binds
    abstract fun bindStorage(impl: DatabaseStorage): Storage
}
```

### Б. @Provides (Гибкий путь)

Используется, когда объект создается сложно (нужен Builder, внешняя библиотека типа Retrofit) или когда нужно создать экземпляр класса, который не помечен `@Inject`.
```Kotlin
@Provides
fun provideRetrofit(): Retrofit = Retrofit.Builder().baseUrl("...").build()
```

---

## 3. Qualifiers (Аннотации-определители)

Что делать, если у вас два разных репозитория одного типа? Например, один для работы с локальной БД, другой — с сетью. Hilt не поймет, какой выбрать, и выдаст ошибку. Для этого нужны **Qualifiers**.
```Kotlin
// 1. Создаем свои аннотации
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class RemoteDataSource

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class LocalDataSource

// 2. Помечаем ими методы в модуле
@Module
@InstallIn(SingletonComponent::class)
object DataModule {
    @RemoteDataSource
    @Provides
    fun provideRemoteData(): ApiService = ...

    @LocalDataSource
    @Provides
    fun provideLocalData(): ApiService = ...
}

// 3. Используем при внедрении
class MyRepository @Inject constructor(
    @RemoteDataSource private val remoteApi: ApiService,
    @LocalDataSource private val localApi: ApiService
)
```

---

## 4. Предопределенные Context (Qualifiers от Hilt)

В Android нам постоянно нужен `Context`. Hilt уже знает про него, достаточно использовать готовые аннотации:

- **`@ApplicationContext`**: вернет контекст всего приложения.
    
- **`@ActivityContext`**: вернет контекст текущей Activity.
```Kotlin
class AnalyticsHelper @Inject constructor(
    @ApplicationContext private val context: Context
)
```

---

## 5. EntryPoints (Точки входа)

Иногда нужно получить зависимость в классе, который **не поддерживается Hilt** (например, какой-нибудь старый ContentProvider или чисто Java-код). Для этого используются `EntryPoint`.

```Kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface MyEntryPoint {
    fun getMyService(): MyService
}

// Получение внутри неподдерживаемого класса:
val entryPoint = EntryPointAccessors.fromApplication(context, MyEntryPoint::class.java)
val service = entryPoint.getMyService()
```

---

## 6. Hilt в Jetpack Compose

В Compose не нужно вручную пробрасывать зависимости. Достаточно использовать функцию `hiltViewModel()`.

```Kotlin
@Composable
fun MyScreen(
    // Hilt сам найдет нужную ViewModel и внедрит в неё все зависимости
    viewModel: MyViewModel = hiltViewModel() 
) {
    val state by viewModel.uiState.collectAsState()
    // ...
}
```

---

### Ключевые советы для конспекта:

1. **Scope:** Не ставь `@Singleton` на всё подряд. Если объект нужен только на одном экране, используй `@ActivityScoped` или `@ViewModelScoped`.
    
2. **Modules:** Если класс твой и ты можешь добавить `@Inject` в конструктор — делай так. Используй модули только для интерфейсов или сторонних библиотек.
    
3. **Чистота:** Hilt избавляет от «Boilerplate» кода (шаблонного кода), который раньше занимал сотни строк в Dagger.
    

