**Room** — это официальная библиотека от Google для работы с локальной базой данных SQLite. Если раньше разработчики мучились с SQL-запросами вручную, то Room делает это через аннотации, проверяет синтаксис на этапе компиляции и отлично дружит с **Coroutines** и **Flow**.

---

# Room Persistence Library

## 1. Три кита Room

1. **Entity (Сущность)**: Класс (Data Class), который описывает таблицу в базе данных.
    
2. **DAO (Data Access Object)**: Интерфейс, через который мы пишем SQL-запросы (методы доступа к данным).
    
3. **Database**: Главная точка входа, которая связывает сущности с DAO.
    

---

## 2. Entity (Таблица)

Каждое поле в классе — это колонка в таблице.

```Kotlin
@Entity(tableName = "users") // Имя таблицы в БД
data class User(
    @PrimaryKey(autoGenerate = true) val id: Int = 0, // Уникальный ID
    @ColumnInfo(name = "full_name") val name: String, // Кастомное имя колонки
    val age: Int
)
```

---

## 3. DAO (Запросы)

Здесь мы описываем, как доставать или менять данные. Благодаря `suspend` и `Flow`, Room сам уводит работу с БД в фоновый поток.

```Kotlin
@Dao
interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE) // Если такой ID есть — заменить
    suspend fun insertUser(user: User)

    @Query("SELECT * FROM users WHERE age > :minAge")
    fun getUsersOlderThan(minAge: Int): Flow<List<User>> // Flow будет сам обновлять UI при изменениях в БД

    @Delete
    suspend fun deleteUser(user: User)
}
```

---

## 4. Database (База данных)

Класс-синглтон, который создает саму базу.

```Kotlin
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

---

## 5. Настройка через Hilt (DatabaseModule)

Базу данных нужно создавать один раз на всё приложение.

```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "my_app_db"
        ).build()
    }

    @Provides
    fun provideUserDao(db: AppDatabase): UserDao = db.userDao()
}
```

---

## 6. Кэширование данных (Связка Retrofit + Room)

Это стандартная задача: загрузить из сети и сохранить в локальную базу, чтобы пользователь видел данные без интернета.

```Kotlin
class UserRepository @Inject constructor(
    private val api: NewsApi,
    private val dao: NewsDao
) {
    // 1. Подписываемся на локальную базу (Single Source of Truth)
    val newsFlow: Flow<List<Article>> = dao.getAllArticles()

    // 2. Функция для обновления данных из сети
    suspend fun refreshNews() {
        try {
            val response = api.getNews()
            dao.insertAll(response.articles) // Сохраняем в БД
        } catch (e: Exception) {
            // Ошибка сети — пользователь всё равно видит старые данные из БД
        }
    }
}
```

---

## 7. Важные фишки и аннотации

- **`@Relation`**: Для связей "один ко многим" (например, один пользователь — много постов).
    
- **`@TypeConverters`**: Room умеет хранить только примитивы. Если тебе нужно сохранить сложный объект (например, `Date` или `List`), нужно написать конвертер в строку/число.
    
- **`onConflict`**: Стратегии при совпадении ID: `REPLACE` (заменить), `IGNORE` (игнорировать), `ABORT` (ошибка).
    

---

### Ключевые моменты:

1. **Flow в DAO**: Всегда используй `Flow`, если хочешь, чтобы экран обновлялся автоматически, как только в базе что-то изменилось.
    
2. **Migrations**: Если ты изменил структуру Entity (добавил колонку), тебе нужно либо удалить БД и создать заново (на этапе разработки), либо написать миграцию (для живых приложений), иначе приложение упадет.
    
3. **Main Thread**: Room **запрещает** запросы в главном потоке. Если не использовать корутины или Flow, получишь краш.
    

---
