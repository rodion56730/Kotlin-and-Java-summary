# 🌱 ПАТТЕРНЫ В SPRING - ПОНЯТНЫМ ЯЗЫКОМ С АНАЛОГИЯМИ

## 🏗️ ПОРОЖДАЮЩИЕ ПАТТЕРНЫ В SPRING

### 1. Singleton (Одиночка) - 👑 КОРОЛЬ SPRING
**Аналогия:** В королевстве может быть только один король. Все обращаются к нему.

**В Spring:** По умолчанию ВСЕ бины в Spring - синглтоны.

```java
@Service
public class UserService {
    // Этот бин создается ОДИН РАЗ на все приложение
    public User findUser(Long id) {
        return userRepository.findById(id);
    }
}

// ГДЕ ВИДИТЕ В SPRING:
// - @Service, @Component, @Repository по умолчанию
// - @Controller (но для каждого запроса новый экземпляр)
// - Configuration классы

// КАК ПРОВЕРИТЬ:
@Autowired private UserService service1;
@Autowired private UserService service2;
// service1 == service2 - будет true! Это один и тот же объект
```

**Зачем Spring это нужно:** Экономит память, переиспользует объекты.

---

### 2. Factory Method (Фабричный метод) - 🏭 ФАБРИКА SPRING
**Аналогия:** Вы говорите "хочу бин", Spring решает как его создать.

**В Spring:** Spring сам является большой фабрикой по созданию бинов.

```java
@Configuration
public class AppConfig {
    
    // Spring вызывает этот метод когда нужен DataSource
    @Bean
    public DataSource dataSource() {
        // Здесь может быть сложная логика создания
        return new DriverManagerDataSource("jdbc:mysql://localhost/mydb");
    }
}

// ИЛИ через аннотации:
@Component
public class UserServiceFactory {
    
    // Фабричный метод
    public UserService createUserService(String type) {
        if ("admin".equals(type)) {
            return new AdminUserService();
        } else {
            return new RegularUserService();
        }
    }
}

// КАК ИСПОЛЬЗУЕТСЯ:
// Spring сканирует @Bean методы и сам решает КОГДА их вызывать
// Вы просто говорите: "@Autowired DataSource dataSource"
// Spring сам вызовет фабричный метод dataSource()
```

**Зачем Spring это нужно:** Контролирует процесс создания объектов.

---

### 3. Prototype (Прототип) - 🐑 КЛОНИРОВАНИЕ В SPRING
**Аналогия:** Каждый раз когда нужно, создается новый экземпляр (как печенье из формочки).

**В Spring:** Когда нужно чтобы каждый раз создавался новый объект.

```java
@Component
@Scope("prototype") // Ключевая аннотация!
public class PaymentSession {
    private String sessionId;
    
    public PaymentSession() {
        this.sessionId = UUID.randomUUID().toString(); // У каждого свой ID
    }
}

// КАК ИСПОЛЬЗОВАТЬ:
@Service
public class PaymentService {
    
    @Autowired
    private ApplicationContext context; // Нужен для получения prototype бинов
    
    public void processPayment() {
        // Каждый раз новый объект!
        PaymentSession session1 = context.getBean(PaymentSession.class);
        PaymentSession session2 = context.getBean(PaymentSession.class);
        
        // session1 != session2 - РАЗНЫЕ объекты!
        // У каждого свой sessionId
    }
}

// ГДЕ ИСПОЛЬЗУЕТСЯ:
// - Сессии пользователей
// - Временные объекты
// - Объекты с состоянием, которое нельзя разделять
```

---

## 🏛️ СТРУКТУРНЫЕ ПАТТЕРНЫ В SPRING

### 4. Proxy (Прокси) - 🎭 АКТЕР-ДУБЛЕР В SPRING
**Аналогия:** У звезды есть дублер, который выполняет всю черновую работу.

**В Spring:** Spring создает прокси вокруг ваших бинов чтобы добавить функциональность.

```java
@Service
@Transactional // ВОТ ОН - ПРОКСИ!
public class UserService {
    
    public User saveUser(User user) {
        // Кажется, что мы просто сохраняем пользователя
        return userRepository.save(user);
    }
}

// ЧТО ПРОИСХОДИТ НА САМОМ ДЕЛЕ:
// 1. Spring создает ПРОКСИ вокруг UserService
// 2. Когда вы вызываете saveUser(), сначала вызывается прокси
// 3. Прокси открывает транзакцию
// 4. Прокси вызывает ваш настоящий метод saveUser()
// 5. Прокси коммитит или откатывает транзакцию

// ДРУГОЙ ПРИМЕР - AOP:
@Aspect
@Component
public class LoggingAspect {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void logMethodCall(JoinPoint joinPoint) {
        System.out.println("Вызывается метод: " + joinPoint.getSignature().getName());
    }
}

// Теперь при вызове ЛЮБОГО метода в service пакете
// Spring-прокси сначала вызовет logMethodCall()
```

**Зачем Spring это нужно:** Добавляет функциональность без изменения вашего кода.

---

### 5. Adapter (Адаптер) - 🇺🇸 ПЕРЕВОДЧИК В SPRING
**Аналогия:** Переводчик адаптирует русскую речь для англичанина.

**В Spring:** Spring адаптирует HTTP запросы к Java методам.

```java
@RestController
public class UserController {
    
    // Spring адаптирует HTTP GET запрос в вызов этого метода
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}

// КАК ЭТО РАБОТАЕТ:
// HTTP: GET /users/123  →  Spring: userController.getUser(123L)

// ДРУГОЙ ПРИМЕР - Security:
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(authz -> authz
            .requestMatchers("/admin/**").hasRole("ADMIN") // Адаптер правил безопасности
            .anyRequest().authenticated()
        );
        return http.build();
    }
}
// Spring адаптирует URL паттерны к правилам безопасности
```

---

## 🎭 ПОВЕДЕНЧЕСКИЕ ПАТТЕРНЫ В SPRING

### 6. Observer (Наблюдатель) - 📢 РАДИОСТАНЦИЯ В SPRING
**Аналогия:** Радиостанция вещает новость, все радиоприемники ее ловят.

**В Spring:** Система событий - когда что-то происходит, все заинтересованные получают уведомление.

```java
// 1. СОБЫТИЕ (Новость)
public class UserRegisteredEvent extends ApplicationEvent {
    private User user;
    
    public UserRegisteredEvent(Object source, User user) {
        super(source);
        this.user = user;
    }
    public User getUser() { return user; }
}

// 2. ИЗДАТЕЛЬ (Радиостанция)
@Service
public class UserService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher; // Spring дает нам "радиопередатчик"
    
    public User registerUser(User user) {
        User savedUser = userRepository.save(user);
        
        // ВЕЩАЕМ СОБЫТИЕ: "Пользователь зарегистрирован!"
        eventPublisher.publishEvent(new UserRegisteredEvent(this, savedUser));
        
        return savedUser;
    }
}

// 3. ПОДПИСЧИКИ (Радиоприемники)
@Component
public class EmailService {
    
    // "НАСТРАИВАЕМ ПРИЕМНИК" на событие UserRegisteredEvent
    @EventListener
    public void sendWelcomeEmail(UserRegisteredEvent event) {
        System.out.println("Отправляем welcome email для: " + event.getUser().getEmail());
    }
}

@Component
public class AnalyticsService {
    
    @EventListener
    public void trackRegistration(UserRegisteredEvent event) {
        System.out.println("Отслеживаем регистрацию: " + event.getUser().getId());
    }
}

// ЧТО ПРОИСХОДИТ:
// 1. UserService регистрирует пользователя
// 2. UserService "вещает" событие
// 3. EmailService и AnalyticsService АВТОМАТИЧЕСКИ получают уведомление
// 4. Оба выполняют свои действия
```

**Зачем Spring это нужно:** Развязывает компоненты, делает систему гибкой.

---

### 7. Template Method (Шаблонный метод) - 📝 БЛАНК ДОКУМЕНТА В SPRING
**Аналогия:** Бланк заявления - общая структура есть, вам нужно заполнить только свои данные.

**В Spring:** Spring предоставляет шаблон с готовой логикой, вам нужно реализовать только специфичные части.

```java
@Repository
public class UserRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate; // ШАБЛОН от Spring
    
    public User findById(Long id) {
        // Spring делает ВСЮ черновую работу:
        // - Открывает соединение с БД
        // - Создает PreparedStatement
        // - Выполняет запрос
        // - Обрабатывает исключения
        // - Закрывает соединение
        
        // Вам нужно только сказать КАК преобразовать ResultSet в User
        return jdbcTemplate.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new Object[]{id},
            (rs, rowNum) -> new User(  // ЭТУ ЧАСТЬ вы реализуете
                rs.getLong("id"),
                rs.getString("name"),
                rs.getString("email")
            )
        );
    }
}

// ДРУГИЕ ШАБЛОНЫ В SPRING:
// - RestTemplate (для HTTP запросов)
// - JmsTemplate (для JMS сообщений)
// - TransactionTemplate (для транзакций)
```

**Зачем Spring это нужно:** Упрощает рутинные операции, предотвращает ошибки.

---

### 8. Strategy (Стратегия) - 🎯 РАЗНЫЕ СПОСОБЫ ОПЛАТЫ В SPRING
**Аналогия:** Магазин принимает разные способы оплаты - карта, PayPal, наличные.

**В Spring:** Внедрение разных реализаций одного интерфейса.

```java
// СТРАТЕГИЯ (интерфейс оплаты)
public interface PaymentStrategy {
    boolean pay(BigDecimal amount);
}

// КОНКРЕТНЫЕ СТРАТЕГИИ
@Component("creditCard") // Даем имя бину
public class CreditCardPayment implements PaymentStrategy {
    public boolean pay(BigDecimal amount) {
        System.out.println("Оплата картой: " + amount);
        return true;
    }
}

@Component("paypal")
public class PayPalPayment implements PaymentStrategy {
    public boolean pay(BigDecimal amount) {
        System.out.println("Оплата через PayPal: " + amount);
        return true;
    }
}

// КОНТЕКСТ (сервис который использует стратегии)
@Service
public class PaymentService {
    
    // Spring автоматически внедряет ВСЕ реализации PaymentStrategy
    private final Map<String, PaymentStrategy> strategies;
    
    // Конструктор получает список всех стратегий
    public PaymentService(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(
                s -> s.getClass().getSimpleName(), 
                Function.identity()
            ));
    }
    
    public boolean processPayment(String paymentType, BigDecimal amount) {
        PaymentStrategy strategy = strategies.get(paymentType);
        if (strategy != null) {
            return strategy.pay(amount); // Используем выбранную стратегию
        }
        throw new IllegalArgumentException("Unknown payment type: " + paymentType);
    }
}

// ИСПОЛЬЗОВАНИЕ:
// paymentService.processPayment("CreditCardPayment", 100.00)
// paymentService.processPayment("PayPalPayment", 200.00)
```

**Зачем Spring это нужно:** Позволяет легко добавлять новые реализации.

---

## 💡 КАК ЭТО ВИДЕТЬ В РЕАЛЬНОМ КОДЕ SPRING

### Когда вы видите:
```java
@Service
@Transactional
public class MyService {
    // У вас ПРОКСИ + СИНГЛТОН
}
```

### Когда вы видите:
```java
@EventListener  
public void handleEvent(MyEvent event) {
    // У вас НАБЛЮДАТЕЛЬ
}
```

### Когда вы видите:
```java
@Bean
public DataSource dataSource() {
    // У вас ФАБРИЧНЫЙ МЕТОД
}
```

### Когда вы видите:
```java
jdbcTemplate.query("SELECT ...", rowMapper);
// У вас ШАБЛОННЫЙ МЕТОД
```

## 🎯 ВЫВОД ДЛЯ СОБЕСЕДОВАНИЯ

**Говорите так:** "Spring неявно использует паттерны чтобы упростить разработку. Например:
- **Singleton** - чтобы экономить память на бинах
- **Proxy** - чтобы добавлять транзакции и безопасность
- **Observer** - чтобы компоненты общались без прямых связей
- **Template Method** - чтобы мы не писали boilerplate код"

