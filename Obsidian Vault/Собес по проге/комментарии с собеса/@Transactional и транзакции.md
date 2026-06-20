# 🔄 @Transactional - Полное руководство

## 🎯 Что такое @Transactional?

**@Transactional** - это аннотация Spring, которая управляет **транзакциями** в приложении. Простыми словами - она гарантирует, что группа операций выполнится либо **все вместе**, либо **ни одна не выполнится**.
## 🎯 Что такое транзакция?

**Транзакция** - это группа операций, которые выполняются как **единое целое**. 

### 🏦 Аналогия из жизни: **БАНКОВСКИЙ ПЕРЕВОД**

```java
// БЕЗ транзакции - ОПАСНО!
public void transferMoney(Long fromAccount, Long toAccount, BigDecimal amount) {
    // 1. Снимаем деньги с первого счета
    accountService.withdraw(fromAccount, amount);  // ✅ Деньги сняты
    
    // 💥 Тут может произойти сбой: отключение электричества, исключение...
    // databaseConnection.fail(); 
    
    // 2. Кладем деньги на второй счет  
    accountService.deposit(toAccount, amount);     // ❌ Не выполнится!
    
    // РЕЗУЛЬТАТ: Деньги УТЕРЯНЫ!
}
```

```java
// С транзакцией - БЕЗОПАСНО!
@Transactional
public void transferMoney(Long fromAccount, Long toAccount, BigDecimal amount) {
    accountService.withdraw(fromAccount, amount);  // ⏳ Пока только в памяти
    accountService.deposit(toAccount, amount);     // ⏳ Пока только в памяти
    
    // Если ВСЁ успешно - Spring ВЫПОЛНИТ КОММИТ
    // Если ошибка - Spring ВЫПОЛНИТ ОТКАТ (ROLLBACK)
    // РЕЗУЛЬТАТ: Либо обе операции, либо ни одной!
}
```

---

## ⚡ Свойства транзакций (ACID)

### 1. **Atomicity (Атомарность)** - "Все или ничего"
```java
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);           // 1. Сохраняем заказ
    inventoryService.reserveItems(order);  // 2. Резервируем товары
    paymentService.processPayment(order);  // 3. Обрабатываем платеж
    
    // Если ЛЮБАЯ операция упадет - ОТКАТЯТСЯ ВСЕ!
}
```

### 2. **Consistency (Согласованность)** - "Правила соблюдаются"
```java
@Transactional
public void updateUserBalance(Long userId, BigDecimal amount) {
    User user = userRepository.findById(userId);
    user.setBalance(user.getBalance().add(amount));
    
    // Автоматически проверяются:
    // - NOT NULL constraints
    // - FOREIGN KEY constraints  
    // - UNIQUE constraints
    // - CHECK constraints (баланс >= 0)
    
    userRepository.save(user);
}
```

### 3. **Isolation (Изолированность)** - "Транзакции не мешают друг другу"
```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void financialOperation() {
    // Другие транзакции не видят незакоммиченные изменения
    // Защита от "грязного" чтения и других проблем
}
```

### 4. **Durability (Устойчивость)** - "Сохранилось - навсегда"
```java
@Transactional
public void saveImportantData(Data data) {
    dataRepository.save(data);
}

// После успешного коммита:
// 💥 Отключение электричества
// 💥 Падение сервера
// 💥 Сбой диска

// После восстановления - данные СОХРАНЯТСЯ!


## ⚡ Как работает @Transactional

### Базовый принцип:
```java
// ВАШ КОД:
@Transactional
public void businessMethod() {
    userRepository.save(user);
    orderRepository.save(order);
    inventoryService.updateStock(product, quantity);
}

// ЧТО ДЕЛАЕТ SPRING:
public void businessMethodProxy() {
    Connection connection = null;
    try {
        connection = dataSource.getConnection();
        connection.setAutoCommit(false); // ⏸️ Начинаем транзакцию
        
        // ВЫЗЫВАЕМ ВАШ МЕТОД
        userRepository.save(user);
        orderRepository.save(order);
        inventoryService.updateStock(product, quantity);
        
        connection.commit(); // ✅ ВСЕ операции успешны - сохраняем
    } catch (Exception e) {
        if (connection != null) {
            connection.rollback(); // ❌ Что-то пошло не так - откатываем ВСЁ
        }
        throw e;
    }
}
```


## 🛠️ Как работает @Transactional в Spring

### Магия под капотом:
```java
// ВАШ КОД:
@Service
public class UserService {
    
    @Transactional
    public User createUser(User user) {
        return userRepository.save(user);
    }
}

// ЧТО ДЕЛАЕТ SPRING (упрощенно):
public class UserServiceProxy extends UserService {
    
    public User createUser(User user) {
        Connection connection = null;
        try {
            // 1. Открываем транзакцию
            connection = dataSource.getConnection();
            connection.setAutoCommit(false); // ⏸️ Начинаем транзакцию
            
            // 2. Вызываем ваш метод
            User result = super.createUser(user);
            
            // 3. Если успешно - коммитим
            connection.commit(); // ✅ Сохраняем изменения
            return result;
            
        } catch (Exception e) {
            // 4. Если ошибка - откатываем
            if (connection != null) {
                connection.rollback(); // ❌ Откатываем ВСЕ изменения
            }
            throw e;
        } finally {
            // 5. Освобождаем ресурсы
            if (connection != null) {
                connection.close();
            }
        }
    }
}
```

---

## 🎛️ Параметры @Transactional

### 1. **readOnly = true** - только чтение
```java
@Transactional(readOnly = true)
public List<User> findActiveUsers() {
    return userRepository.findByActiveTrue(); // Оптимизация производительности
}
```

### 2. **propagation** - поведение транзакции
```java
// REQUIRED (по умолчанию) - использует существующую или создает новую
@Transactional(propagation = Propagation.REQUIRED)
public void outerMethod() {
    innerMethod(); // Использует ту же транзакцию
}

// REQUIRES_NEW - всегда создает новую транзакцию
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditAction(String action) {
    auditRepository.save(action); // Всегда в отдельной транзакции
}
```

### 3. **isolation** - уровень изоляции
```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void criticalFinancialOperation() {
    // Максимальная изоляция от параллельных транзакций
}
```

### 4. **timeout** - таймаут транзакции
```java
@Transactional(timeout = 30) // 30 секунд
public void generateReport() {
    // Если выполнение займет больше 30 сек - транзакция откатится
}
```

### 5. **rollbackFor** / **noRollbackFor** - правила отката
```java
// Откат только для определенных исключений
@Transactional(rollbackFor = {BusinessException.class, ValidationException.class})
public void processPayment() {
    // Откат только при указанных исключениях
}

// НЕ откатывать для определенных исключений  
@Transactional(noRollbackFor = {NotificationException.class})
public void sendNotification() {
    // Даже если упадет отправка - данные сохранятся
}
```

## 🏗️ Практические примеры

### Пример 1: **Электронная коммерция**
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final EmailService emailService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Проверяем наличие товара
        inventoryService.reserveItems(request.getItems());
        
        // 2. Создаем заказ
        Order order = orderRepository.save(createOrderFromRequest(request));
        
        // 3. Обрабатываем платеж
        paymentService.processPayment(order, request.getPaymentDetails());
        
        // 4. Обновляем инвентарь
        inventoryService.updateStock(request.getItems());
        
        // 5. Отправляем подтверждение (даже если упадет - заказ создан)
        try {
            emailService.sendOrderConfirmation(order);
        } catch (EmailException e) {
            // Логируем, но НЕ откатываем транзакцию
            log.error("Failed to send email", e);
        }
        
        return order;
    }
}
```

### Пример 2: **Банковские операции**
```java
@Service
@Transactional // Все методы класса по умолчанию в транзакции
public class BankService {
    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;
    
    @Transactional(rollbackFor = InsufficientFundsException.class)
    public void transferMoney(Long fromAccount, Long toAccount, BigDecimal amount) {
        // 1. Снимаем со счета отправителя
        Account sender = accountRepository.findById(fromAccount)
            .orElseThrow(() -> new AccountNotFoundException(fromAccount));
        
        if (sender.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException("Недостаточно средств");
        }
        sender.setBalance(sender.getBalance().subtract(amount));
        accountRepository.save(sender);
        
        // 2. Зачисляем на счет получателя
        Account receiver = accountRepository.findById(toAccount)
            .orElseThrow(() -> new AccountNotFoundException(toAccount));
        receiver.setBalance(receiver.getBalance().add(amount));
        accountRepository.save(receiver);
        
        // 3. Сохраняем запись о транзакции
        Transaction transaction = new Transaction(fromAccount, toAccount, amount);
        transactionRepository.save(transaction);
        
        // Если любая операция упадет - ВСЁ откатится!
    }
    
    @Transactional(readOnly = true)
    public BigDecimal getBalance(Long accountId) {
        return accountRepository.findBalanceById(accountId);
    }
}
```

## ⚠️ Важные особенности и подводные камни

### 1. **Self-invocation проблема**
```java
@Service
public class ProblematicService {
    
    public void outerMethod() {
        innerMethod(); // ❌ Транзакция НЕ сработает!
    }
    
    @Transactional
    public void innerMethod() {
        userRepository.save(new User()); // Сохранится БЕЗ транзакции!
    }
}

// РЕШЕНИЕ:
@Service
public class FixedService {
    
    @Autowired
    private FixedService self; // Инжектим сами себя
    
    public void outerMethod() {
        self.innerMethod(); // ✅ Теперь транзакция работает
    }
    
    @Transactional
    public void innerMethod() {
        userRepository.save(new User()); // Теперь в транзакции
    }
}
```

### 2. **Место размещения имеет значение**
```java
// ✅ ПРАВИЛЬНО - на публичных методах сервиса
@Service
public class UserService {
    
    @Transactional
    public User createUser(User user) {
        return userRepository.save(user);
    }
}

// ❌ НЕПРАВИЛЬНО - на приватных методах
@Service
public class UserService {
    
    @Transactional // 💥 НЕ РАБОТАЕТ!
    private User validateAndSave(User user) {
        // Spring не может создать прокси для приватных методов
        return userRepository.save(user);
    }
}
```

### 3. **Порядок аннотаций**
```java
@Service
@Transactional // Действует на ВСЕ методы класса
public class OrderService {
    
    @Transactional(readOnly = true) // Переопределяет для этого метода
    public Order findById(Long id) {
        return orderRepository.findById(id);
    }
    
    // Этот метод будет использовать @Transactional с параметрами по умолчанию
    public Order save(Order order) {
        return orderRepository.save(order);
    }
}
```

### 4. **Checked vs Unchecked исключения**
```java
@Transactional
public void processData() {
    // Unchecked исключения (RuntimeException) - ОТКАТ по умолчанию
    throw new RuntimeException("Ошибка"); // ✅ Вызовет откат
    
    // Checked исключения (Exception) - НЕ вызывают откат по умолчанию
    // throw new Exception("Ошибка"); // ❌ Не вызовет откат
}

// Чтобы checked исключение вызывало откат:
@Transactional(rollbackFor = Exception.class)
public void processData() {
    throw new Exception("Ошибка"); // ✅ Теперь вызовет откат
}
```

## 🔧 Настройка в различных сценариях

### JPA/Hibernate:
```java
@Service
@Transactional
public class UserService {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Transactional
    public void updateUser(Long userId, String newName) {
        User user = entityManager.find(User.class, userId);
        user.setName(newName);
        // Не нужно вызывать entityManager.merge() - изменения сохранятся автоматически
    }
}
```

### Spring Data JPA:
```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Все методы Spring Data JPA уже транзакционные!
    
    @Transactional(readOnly = true)
    @Query("SELECT u FROM User u WHERE u.email = :email")
    User findByEmail(@Param("email") String email);
    
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
    int deactivateInactiveUsers(@Param("date") LocalDateTime date);
}
```

### Программное управление транзакциями:
```java
@Service
@RequiredArgsConstructor
public class ManualTransactionService {
    
    private final TransactionTemplate transactionTemplate;
    private final PlatformTransactionManager transactionManager;
    
    public void manualTransaction() {
        // 1. Использование TransactionTemplate
        transactionTemplate.execute(status -> {
            userRepository.save(user1);
            userRepository.save(user2);
            return null;
        });
        
        // 2. Программное управление через TransactionManager
        TransactionDefinition definition = new DefaultTransactionDefinition();
        TransactionStatus status = transactionManager.getTransaction(definition);
        
        try {
            userRepository.save(user1);
            userRepository.save(user2);
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

## 🎯 Best Practices

### 1. **Размещайте на уровне сервиса**
```java
@Service
@Transactional
public class BusinessService {
    // Бизнес-логика с транзакциями
}

@Repository
public class UserRepository {
    // Только доступ к данным, без @Transactional
}
```

### 2. **Используйте readOnly где возможно**
```java
@Transactional(readOnly = true)
public List<User> findActiveUsers() {
    return userRepository.findByActiveTrue(); // Оптимизация производительности
}
```

### 3. **Будьте осторожны с длительными операциями**
```java
// ❌ ПЛОХО - долгая транзакция
@Transactional
public void generateMonthlyReport() {
    List<Data> data = dataRepository.findLastMonthData(); // Долгий запрос
    // ... сложная обработка ...
    reportRepository.save(report); // Блокировка ресурсов!
}

// ✅ ЛУЧШЕ - разбить на части
@Transactional
public void prepareReportData() {
    // Быстрые операции с БД
}

public void generateReport() {
    // Долгие вычисления без транзакции
}
```

### 4. **Правильно обрабатывайте исключения**
```java
@Transactional(rollbackFor = BusinessException.class)
public void processOrder(Order order) {
    try {
        inventoryService.reserveItems(order.getItems());
        paymentService.processPayment(order);
    } catch (PaymentException e) {
        // Логика обработки, но транзакция откатится
        throw new BusinessException("Payment failed", e);
    }
}
```

