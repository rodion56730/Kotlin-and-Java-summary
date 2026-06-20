

## 🎯 Что такое HTTP методы?

**HTTP методы** (иногда called "HTTP verbs") определяют действие, которое нужно выполнить с ресурсом.

---

## 📋 ОСНОВНЫЕ HTTP МЕТОДЫ

### 1. **GET** - 📥 ПОЛУЧЕНИЕ данных
**Назначение:** Получить данные с сервера (только чтение)

```java
@RestController
public class UserController {
    
    @GetMapping("/users")          // Аннотация Spring
    public List<User> getUsers() {
        return userService.findAll();
    }
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

**Характеристики:**
- ✅ **Безопасный** (не изменяет данные)
- ✅ **Идемпотентный** (многократный вызов дает одинаковый результат)
- 📝 **В URL параметры**
- 🔒 **Кэшируемый**

---

### 2. **POST** - 📤 ОТПРАВКА данных
**Назначение:** Создать новый ресурс

```java
@RestController
public class UserController {
    
    @PostMapping("/users")         // Аннотация Spring
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
    
    @PostMapping("/users/{id}/activate")
    public void activateUser(@PathVariable Long id) {
        userService.activate(id);
    }
}
```

**Характеристики:**
- ❌ **Не безопасный** (изменяет данные)
- ❌ **Не идемпотентный** (каждый вызов создает новый ресурс)
- 📦 **Данные в теле запроса (body)**
- 🔒 **Не кэшируемый**

---

### 3. **PUT** - 💾 ПОЛНОЕ ОБНОВЛЕНИЕ
**Назначение:** Полностью заменить ресурс

```java
@RestController
public class UserController {
    
    @PutMapping("/users/{id}")     // Аннотация Spring
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        user.setId(id);
        return userService.update(user);
    }
}
```

**Характеристики:**
- ❌ **Не безопасный** (изменяет данные)
- ✅ **Идемпотентный** (многократный вызов дает одинаковый результат)
- 📦 **Данные в теле запроса**
- 🔒 **Не кэшируемый**

---

### 4. **PATCH** - 🎨 ЧАСТИЧНОЕ ОБНОВЛЕНИЕ
**Назначение:** Частично обновить ресурс

```java
@RestController
public class UserController {
    
    @PatchMapping("/users/{id}")   // Аннотация Spring
    public User partialUpdateUser(@PathVariable Long id, 
                                 @RequestBody Map<String, Object> updates) {
        return userService.partialUpdate(id, updates);
    }
}
```

**Характеристики:**
- ❌ **Не безопасный**
- ✅ **Идемпотентный** (в идеале)
- 📦 **Только изменяемые поля в теле**
- 🔒 **Не кэшируемый**

---

### 5. **DELETE** - 🗑️ УДАЛЕНИЕ
**Назначение:** Удалить ресурс

```java
@RestController
public class UserController {
    
    @DeleteMapping("/users/{id}")  // Аннотация Spring
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

**Характеристики:**
- ❌ **Не безопасный**
- ✅ **Идемпотентный** (после первого удаления ресурса его уже нет)
- 📝 **В URL идентификатор**
- 🔒 **Не кэшируемый**

---

## 🔧 ДОПОЛНИТЕЛЬНЫЕ МЕТОДЫ

### 6. **HEAD** - ℹ️ ЗАГОЛОВКИ
**Назначение:** Получить только заголовки ответа (без тела)

```java
// Используется для проверки существования ресурса
// или получения мета-информации
```

**Характеристики:**
- ✅ **Безопасный**
- ✅ **Идемпотентный**
- 🔒 **Кэшируемый**

---

### 7. **OPTIONS** - 📋 ВОЗМОЖНОСТИ СЕРВЕРА
**Назначение:** Получить поддерживаемые методы для ресурса

```java
// Автоматически обрабатывается Spring и браузерами (CORS)
```

**Характеристики:**
- ✅ **Безопасный**
- ✅ **Идемпотентный**

---

### 8. **TRACE** - 🔍 ОТЛАДКА
**Назначение:** Диагностика, получает обратно то, что отправил

**Характеристики:**
- ✅ **Безопасный**
- ✅ **Идемпотентный**

---

### 9. **CONNECT** - 🔗 ТУННЕЛИРОВАНИЕ
**Назначение:** Создание TCP-туннеля (для HTTPS)

---

## 🏗️ CRUD ОПЕРАЦИИ И HTTP МЕТОДЫ

| Операция | HTTP метод | Пример URL | Описание |
|----------|------------|------------|-----------|
| **Create** | `POST` | `/users` | Создать нового пользователя |
| **Read** | `GET` | `/users` | Получить всех пользователей |
| **Read** | `GET` | `/users/1` | Получить пользователя с ID=1 |
| **Update** | `PUT` | `/users/1` | Полностью обновить пользователя |
| **Update** | `PATCH` | `/users/1` | Частично обновить пользователя |
| **Delete** | `DELETE` | `/users/1` | Удалить пользователя |

---

## 💻 ПРАКТИЧЕСКИЕ ПРИМЕРЫ В SPRING

### Полный CRUD контроллер:
```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    // GET - Получить всех пользователей
    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }
    
    // GET - Получить пользователя по ID
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    // POST - Создать нового пользователя
    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.create(user);
    }
    
    // PUT - Полностью обновить пользователя
    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        user.setId(id);
        return userService.update(user);
    }
    
    // PATCH - Частично обновить пользователя
    @PatchMapping("/{id}")
    public User partialUpdate(@PathVariable Long id, 
                             @RequestBody Map<String, Object> updates) {
        return userService.partialUpdate(id, updates);
    }
    
    // DELETE - Удалить пользователя
    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

---

## ⚡ СВОЙСТВА МЕТОДОВ

### Безопасность (Safe):
- **GET, HEAD, OPTIONS, TRACE**
- Не изменяют состояние сервера
- Можно безопасно кэшировать

### Идемпотентность (Idempotent):
- **GET, PUT, DELETE, HEAD, OPTIONS, TRACE**
- Многократный вызов дает одинаковый результат
- Важно для повторных запросов при сбоях сети

### Кэшируемость (Cacheable):
- **GET, HEAD**
- Ответы можно кэшировать
- Указывается в заголовках `Cache-Control`, `Expires`

---

## 🎯 RESTful API ПРИНЦИПЫ

### Правильное использование методов:

```java
// ✅ ПРАВИЛЬНО
POST   /users          // Создать пользователя
GET    /users/1        // Получить пользователя
PUT    /users/1        // Обновить всего пользователя
PATCH  /users/1        // Обновить часть пользователя
DELETE /users/1        // Удалить пользователя

// ❌ НЕПРАВИЛЬНО (анти-паттерны)
GET    /users/create   // Использовать GET для создания
GET    /users/delete/1 // Использовать GET для удаления
POST   /users/update   // Использовать POST для обновления
```

### Специфичные действия:
```java
// Для не-CRUD операций используйте POST с понятным URL:
POST /users/1/activate     // Активировать пользователя
POST /users/1/deactivate   // Деактивировать пользователя
POST /users/1/reset-password // Сбросить пароль
```

---

## 🔧 Spring Аннотации

| HTTP метод | Spring аннотация | Альтернативная запись |
|------------|------------------|----------------------|
| **GET** | `@GetMapping` | `@RequestMapping(method = RequestMethod.GET)` |
| **POST** | `@PostMapping` | `@RequestMapping(method = RequestMethod.POST)` |
| **PUT** | `@PutMapping` | `@RequestMapping(method = RequestMethod.PUT)` |
| **PATCH** | `@PatchMapping` | `@RequestMapping(method = RequestMethod.PATCH)` |
| **DELETE** | `@DeleteMapping` | `@RequestMapping(method = RequestMethod.DELETE)` |

---

## 💡 ВЫВОД ДЛЯ СОБЕСЕДОВАНИЯ

**"Основные HTTP методы - это GET для получения данных, POST для создания, PUT для полного обновления, PATCH для частичного обновления, и DELETE для удаления. Каждый метод имеет свои характеристики безопасности и идемпотентности, что важно учитывать при проектировании REST API."**

**Ключевые моменты:**
- **GET** - безопасный, идемпотентный, для чтения
- **POST** - не безопасный, не идемпотентный, для создания
- **PUT** - не безопасный, идемпотентный, для полного обновления
- **PATCH** - не безопасный, идемпотентный, для частичного обновления
- **DELETE** - не безопасный, идемпотентный, для удаления

**На собеседовании спросят:**
- "В чем разница между PUT и PATCH?"
- "Почему GET считается безопасным методом?"
- "Когда использовать POST вместо PUT?"
- "Что такое идемпотентность и какие методы идемпотентны?"