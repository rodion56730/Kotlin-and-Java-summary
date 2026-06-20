## Что такое Spring
- Фреймворк для Java, решает задачи: 
  - IoC (Inversion of Control)  
  - DI (Dependency Injection)  
  - AOP (аспектно-ориентированное программирование)  
- Управляет созданием и связыванием объектов (бинов).

---

## IoC и DI
- **IoC** — инверсия управления: объектами управляет контейнер Spring.  
- **DI** — внедрение зависимостей в объект (через конструктор, сеттер, поле).  
- Пример:
  ```java
  @Service
  public class OrderService {
      private final PaymentService payment;

      @Autowired
      public OrderService(PaymentService payment) {
          this.payment = payment; // DI через конструктор
      }
  }
  ```

## Основные аннотации Spring
- `@Component` — делает класс Spring-бином (управляется контейнером).  
- `@Service` — бин уровня сервисов (бизнес-логика).  
- `@Repository` — бин уровня DAO (работа с БД).  
- `@Controller` — контроллер для MVC (обычно веб).  
- `@RestController` — контроллер, который сразу возвращает JSON/XML.  
- `@Autowired` — внедрение зависимости (через конструктор, поле или сеттер).  
- `@Configuration` — класс с конфигурацией Spring (бин-фабрика).  
- `@Bean` — метод, создающий бин вручную.  
- `@Scope` — область видимости бина (`singleton`, `prototype` и др.).  
- `@Transactional`- это аннотация Spring, которая управляет **транзакциями** в приложении. Простыми словами - она гарантирует, что группа операций выполнится либо **все вместе**, либо **ни одна не выполнится**.

---
## BeanFactory vs ApplicationContext

- **BeanFactory** — базовый контейнер, создаёт бины только по запросу (lazy).
    
- **ApplicationContext** — расширяет BeanFactory:
    
    - Создаёт бины при старте (eager loading).
        
    - Поддерживает аннотации, события, интернационализацию.  
        В реальных проектах почти всегда используется **ApplicationContext**.

---
## Жизненный цикл бина

1. Контейнер Spring создаёт бин.  
2. Внедряются зависимости (`@Autowired`).  
3. Вызывается метод `@PostConstruct` (если есть).  
4. Бин готов к использованию.  
5. Перед уничтожением вызывается `@PreDestroy`.  

## Как можно добавить бин

1. **Через аннотации**:
    
    `@Component public class MyService {}`
    
1. **Через конфигурацию**:
```java
    @Configuration public class AppConfig {     
    @Bean     
    public MyService myService() {         
       return new MyService();     
      } 
    }
```

## Отличия @Component, @Service, @Controller, @Repository

- `@Component` — универсальный бин.
- `@Service` — бин уровня бизнес-логики.
- `@Repository` — бин уровня DAO, добавляет обработку исключений (DataAccessException).
- `@Controller` — веб-контроллер (возвращает View).
- `@RestController` — контроллер, который возвращает JSON/XML.

## ApplicationContext
- **ApplicationContext** — контейнер, в котором хранятся бины.  
- Виды:
  - `AnnotationConfigApplicationContext` — конфигурация через Java-аннотации.  
  - `ClassPathXmlApplicationContext` — конфигурация через XML.  
- Пример:
  ```java
  ApplicationContext context = 
      new AnnotationConfigApplicationContext(AppConfig.class);

  MyService service = context.getBean(MyService.class);
  ```
  
## Scope (область видимости бинов)

- `singleton` (по умолчанию) — один объект на весь контекст.
```java
		@Component
		@Scope("singleton") // можно опустить, это по умолчанию
		public class SingletonService {
		    public SingletonService() {
		        System.out.println("SingletonService created");
		    }
		}
		```
- `prototype` — новый объект при каждом запросе.
```java
@Component
@Scope("prototype")
public class PrototypeService {
    public PrototypeService() {
        System.out.println("PrototypeService created");
    }
}
```
- `request` — один бин на HTTP-запрос (в веб-приложениях).
```java
@Component
@Scope("request")
public class RequestScopedService {
    public RequestScopedService() {
        System.out.println("RequestScopedService created");
    }
}

```
- `session` — один бин на HTTP-сессию.
```java
@Component
@Scope("session")
public class SessionScopedService {
    public SessionScopedService() {
        System.out.println("SessionScopedService created");
    }
}

```

Пример:
`@Component @Scope("prototype") public class MyBean {}`

# Внедрение бинов (Dependency Injection) в Spring

Spring позволяет внедрять зависимости несколькими способами.  
Главная цель: объект получает всё, что ему нужно, **без создания объектов вручную**.  

---

## 1. Внедрение через поле (Field Injection)
- Используется аннотация `@Autowired` прямо над полем.  
- **Минусы:**
  - Трудно тестировать (нельзя легко создать объект без Spring).  
  - Зависимости создаются рефлексией → нарушается инкапсуляция.  
  - Порядок создания объектов менее предсказуем.  

```java
@Component
public class FieldInjectionExample {
    @Autowired
    private ServiceA serviceA; // внедряется автоматически

    public void doSomething() {
        serviceA.action();
    }
}
```
## 2. Внедрение через сеттер (Setter Injection)

- Внедрение через метод с `@Autowired`.
    
- **Плюсы:**
    
    - Позволяет менять зависимости после создания объекта.
    - Можно сделать необязательные зависимости (`required=false`).
- **Минусы:**
    
    - Объект может быть создан без полной зависимости → потенциально NPE.
    - Порядок вызова сеттеров важен.

```java
@Component 
public class SetterInjectionExample {     
	private ServiceA serviceA;      
	@Autowired     
	public void setServiceA(ServiceA serviceA) {         
	this.serviceA = serviceA;     
	}      
	public void doSomething() {         
	serviceA.action();     
	} 
	}
```

**Вывод:** лучше, чем field injection, но всё ещё риск частично инициализированного объекта.

---

## 3. Внедрение через конструктор (Constructor Injection)

- Внедрение через конструктор класса (с `@Autowired` можно опустить в Spring 4.3+ для одного конструктора).
    
- **Плюсы:**
    
    - Объект создаётся только с полными зависимостями → безопасно.
    - Immutable объекты.
    - Легко тестировать (без Spring).
    - Порядок вызова предсказуем.

- **Минусы:** больше кода, если много зависимостей (можно использовать Lombok `@RequiredArgsConstructor`).

```java
@Component 
public class ConstructorInjectionExample {     
	private final ServiceA serviceA;  // Spring автоматически внедрит ServiceA     
    public ConstructorInjectionExample(ServiceA serviceA) {     
      this.serviceA = serviceA;    
    }      
	public void doSomething() {         
     serviceA.action();     
    } 
}
```

**Вывод:** **предпочтительный способ** для большинства случаев.

## Spring Boot (минимум)

- **Spring Boot** — упрощает настройку Spring-приложений.
- Автоконфигурация + встроенный сервер (Tomcat/Jetty).
- Основная аннотация:
    - `@SpringBootApplication` — включает автоконфигурацию и сканирование компонентов.
- Запуск приложения:
```java
@SpringBootApplication 
public class DemoApplication {    
 public static void main(String[] args) {
   SringApplication.run(DemoApplication.class, args);     
 }
}
```


## Популярные области применения аннотаций

- **Web**:
    - `@Controller`, `@RestController`, `@GetMapping`, `@PostMapping`
- **DI**:
    - `@Autowired`, `@Qualifier` (указывает конкретный бин, если их несколько)
- **Конфигурация**:
    - `@Configuration`, `@Bean`, `@PropertySource`

---

## Циклическая зависимость

- Ситуация: A зависит от B, а B зависит от A.
- Решения:
    - Использовать **DI через конструктор + @Lazy** на одной из зависимостей.
    - Перепроектировать (часто лучше разделить логику).

Пример:

```java 
@Service public class A {     
 private final B b; 
   
 public A(@Lazy B b) { this.b = b; } 
}
  ```