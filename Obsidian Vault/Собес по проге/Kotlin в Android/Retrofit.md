**Retrofit** — это стандарт де-факто для работы с сетью в Android. Это типобезопасный HTTP-клиент, который превращает ваш API в обычный интерфейс Kotlin.

В связке с **Hilt**, **Coroutines** и **Serialization**, работа с сетью превращается в чистое удовольствие.

---

# Retrofit: Работа с сетью

## 1. Основные компоненты

Для работы Retrofit нужны три вещи:

1. **Интерфейс API**: Описание эндпоинтов.
    
2. **Конвертер**: Чтобы превращать JSON в объекты Kotlin (рекомендуется **Kotlin Serialization** или **Moshi**).
    
3. **Экземпляр Retrofit**: Конфигурация клиента (base URL, логирование).
    

---

## 2. Описание API (Интерфейс)

Используем `suspend` функции, чтобы Retrofit сам управлял потоками корутин.

```Kotlin
interface NewsApi {
    @GET("v2/top-headlines")
    suspend fun getTopNews(
        @Query("country") country: String,
        @Query("apiKey") key: String
    ): NewsResponse // Возвращает готовую модель данных
}
```

---

## 3. Настройка через Hilt (NetworkModule)

Правильно создавать Retrofit как **Singleton**, чтобы не плодить лишние экземпляры. Здесь же добавим `OkHttpClient`для логирования запросов.

```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://newsapi.org/")
            .client(okHttpClient)
            // Конвертер для JSON
            .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
            .build()
    }

    @Provides
    @Singleton
    fun provideNewsApi(retrofit: Retrofit): NewsApi {
        return retrofit.create(NewsApi::class.java)
    }
}
```

---

## 4. Обработка ошибок

В сетевых запросах всегда что-то может пойти не так (нет интернета, ошибка 500). Хороший тон — использовать блок `try-catch` или обертку `Result`.

```Kotlin
class NewsRepository @Inject constructor(
    private val api: NewsApi
) {
    suspend fun fetchNews(): Result<List<Article>> {
        return try {
            val response = api.getTopNews("us", "your_key")
            Result.success(response.articles)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

---

## 5. Полная цепочка: От API до UI

Вот как это выглядит в сборе:

1. **Repository** делает `suspend` запрос.
    
2. **ViewModel** запускает корутину и кладет результат в `StateFlow`.
    
3. **Compose** отображает данные или ошибку.

```Kotlin
@HiltViewModel
class NewsViewModel @Inject constructor(
    private val repository: NewsRepository
) : ViewModel() {

    private val _articles = MutableStateFlow<List<Article>>(emptyList())
    val articles = _articles.asStateFlow()

    fun loadNews() {
        viewModelScope.launch {
            repository.fetchNews()
                .onSuccess { _articles.value = it }
                .onFailure { /* логируем ошибку */ }
        }
    }
}
```

---

## 6. Полезные аннотации Retrofit

- **`@GET`, `@POST`, `@PUT`, `@DELETE`**: HTTP-методы.
    
- **`@Path`**: Для динамических путей (например, `users/{id}/profile`).
    
- **`@Query`**: Для параметров в URL (например, `?page=1`).
    
- **`@Body`**: Для отправки объекта в теле POST-запроса.
    
- **`@Headers`**: Для передачи токенов или статических заголовков.
    

---

### Ключевые моменты:

- **Интерфейс — это карта**: Ты просто рисуешь карту запросов, а Retrofit строит по ней дороги.
    
- **OkHttp Interceptors**: Это "таможня". Здесь удобно добавлять API-ключи ко всем запросам сразу или логировать ошибки.
    
- **Coroutines**: Забудь про `Call.enqueue()`, используй `suspend` функции — это современный стандарт.
    
---

# Глубокий разбор аннотаций Retrofit

## 1. Методы запросов (HTTP Methods)

Каждая функция в интерфейсе API должна иметь аннотацию метода.

- **`@GET`** — получение данных. Параметры передаются в URL.
    
- **`@POST`** — создание/отправка данных. Обычно данные идут в теле (`@Body`).
    
- **`@PUT`** — полное обновление объекта.
    
- **`@PATCH`** — частичное обновление объекта.
    
- **`@DELETE`** — удаление.
    

---

## 2. Манипуляция URL (Пути и Запросы)

### `@Path` (Динамический путь)

Используется, когда часть адреса меняется (например, ID пользователя).

```Kotlin
@GET("users/{id}/repos")
suspend fun getRepos(@Path("id") userId: Int): List<Repo>
// Вызов getRepos(123) превратится в "users/123/repos"
```

### `@Query` (Параметры строки запроса)

Добавляет параметры после знака `?` в URL.

```Kotlin
@GET("search/users")
suspend fun search(
    @Query("q") query: String,
    @Query("page") page: Int = 1
): SearchResponse
// Вызов search("kotlin") -> "search/users?q=kotlin&page=1"
```

### `@QueryMap` (Список параметров)

Если параметров слишком много, их можно передать словарем `Map`.

```Kotlin
@GET("search")
suspend fun complexSearch(@QueryMap params: Map<String, String>): List<Item>
```

---

## 3. Работа с телом запроса (Body)

### `@Body`

Отправляет объект как тело запроса. Retrofit пропустит его через конвертер (например, в JSON).

```Kotlin
@POST("users/new")
suspend fun createUser(@Body user: User)
```

### `@FormUrlEncoded` + `@Field`

Используется для отправки данных формы (как в HTML). В заголовке будет `application/x-www-form-urlencoded`.

```Kotlin
@FormUrlEncoded
@POST("login")
suspend fun login(
    @Field("username") user: String,
    @Field("password") pass: String
)
```

### `@Multipart` + `@Part`

Используется для загрузки файлов или очень больших кусков данных.

```Kotlin
@Multipart
@POST("user/photo")
suspend fun upload(
    @Part("description") description: RequestBody,
    @Part photo: MultipartBody.Part // Сам файл
)
```

---

## 4. Заголовки (Headers)

### `@Headers` (Статические)

Если нужно передать фиксированные заголовки.

```Kotlin
@Headers("Cache-Control: max-age=640000", "User-Agent: My-App")
@GET("data")
suspend fun getData(): Data
```

### `@Header` (Динамические)

Если заголовок меняется (например, токен авторизации).

```Kotlin
@GET("profile")
suspend fun getProfile(@Header("Authorization") token: String): Profile
```

---

## 5. Другие важные аннотации

- **`@Url`**: Позволяет передать **весь** URL целиком, игнорируя `baseUrl`. Полезно для скачивания файлов по прямым ссылкам.
    
    
    
    ```Kotlin
    @GET
    suspend fun downloadFile(@Url fileUrl: String): ResponseBody
    ```
    
- **`@Streaming`**: Для больших файлов. Заставляет Retrofit не загружать весь файл в память сразу, а отдавать его потоком (Stream).
    

---

## 6. Как это выглядит в жизни (Пример для Obsidian)



```Kotlin
interface AdvancedApi {

    // Смешанный пример: путь + запрос + заголовок
    @GET("api/{version}/users")
    suspend fun getUsers(
        @Path("version") ver: String,
        @Query("sort") sortBy: String,
        @Header("X-Device-ID") deviceId: String
    ): List<User>

    // Отправка формы (например, OAuth)
    @FormUrlEncoded
    @POST("token")
    suspend fun getToken(
        @Field("grant_type") type: String,
        @Field("code") code: String
    ): TokenResponse
}
```

---

### Тонкости и советы:

1. **`@Query` с null**: Если ты передашь `null` в качестве значения `@Query`, Retrofit просто проигнорирует этот параметр и не добавит его в URL.
    
2. **Trailing Slash**: Всегда пиши `baseUrl` с косой чертой в конце (напр. `https://api.com/`), а пути в аннотациях **БЕЗ**косой черты в начале. Это убережет от ошибок склейки адресов.
    
3. **Encapsulation**: Для токенов авторизации лучше не использовать `@Header` в каждом методе, а написать **Interceptor** в OkHttp. Это чище.
    