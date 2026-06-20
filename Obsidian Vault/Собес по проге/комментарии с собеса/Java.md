![[Pasted image 20251009200549.png]]
# Автобоксинг и анбоксинг
## 1. Основные понятия

### Примитивные типы и их обёртки

| Примитивный тип | Класс-обёртка | Размер |
|----------------|---------------|---------|
| `byte` | `Byte` | 1 байт |
| `short` | `Short` | 2 байта |
| `int` | `Integer` | 4 байта |
| `long` | `Long` | 8 байт |
| `float` | `Float` | 4 байта |
| `double` | `Double` | 8 байт |
| `char` | `Character` | 2 байта |
| `boolean` | `Boolean` | ~1 байт |

## 2. Боксинг (Boxing)

**Боксинг** - преобразование примитивного типа в соответствующий класс-обёртку.

### Явный боксинг (до Java 5)
```java
// Ручное создание объектов
Integer intObj = new Integer(42);
Double doubleObj = new Double(3.14);
Character charObj = new Character('A');
```

### Неявный боксинг (автобоксинг)
```java
// Автоматическое преобразование (Java 5+)
Integer intObj = 42;        // авто-боксинг int → Integer
Double doubleObj = 3.14;    // авто-боксинг double → Double
Boolean boolObj = true;     // авто-боксинг boolean → Boolean
```

## 3. Анбоксинг (Unboxing)

**Анбоксинг** - преобразование класса-обёртки обратно в примитивный тип.

### Явный анбоксинг
```java
Integer intObj = Integer.valueOf(100);
int primitiveInt = intObj.intValue();    // явный анбоксинг
double primitiveDouble = doubleObj.doubleValue();
boolean primitiveBool = boolObj.booleanValue();
```

### Неявный анбоксинг (авто-анбоксинг)
```java
Integer intObj = 100;
int primitiveInt = intObj;              // авто-анбоксинг Integer → int

Double doubleObj = 2.71;
double primitiveDouble = doubleObj;     // авто-анбоксинг Double → double

Boolean boolObj = true;
boolean primitiveBool = boolObj;        // авто-анбоксинг Boolean → boolean
```

## 4. Автобоксинг в действии

### Примеры использования

```java
// 1. В выражениях
Integer a = 10;     // авто-боксинг
Integer b = 20;     // авто-боксинг
int result = a + b; // авто-анбоксинг + сложение

// 2. В методах
public static void processInteger(Integer number) {
    System.out.println(number);
}

// Вызов метода
processInteger(42); // авто-боксинг int → Integer

// 3. В коллекциях
List<Integer> numbers = new ArrayList<>();
numbers.add(1);     // авто-боксинг int → Integer
numbers.add(2);     // авто-боксинг int → Integer

int first = numbers.get(0); // авто-анбоксинг Integer → int
```

## 5. Особенности и подводные камни

### Сравнение объектов
```java
Integer a = 100;
Integer b = 100;
Integer c = 200;
Integer d = 200;

System.out.println(a == b); // true (значения из кэша)
System.out.println(c == d); // false (разные объекты)
System.out.println(a.equals(b)); // всегда true
System.out.println(c.equals(d)); // всегда true
```

**Важно:** Кэшируются значения от -128 до 127 для `Integer`, `Byte`, `Short`, `Long`, `Character` (0-127).

### Производительность
```java
// Медленно - много авто-боксинга/анбоксинга
Long sum = 0L;
for (long i = 0; i < Integer.MAX_VALUE; i++) {
    sum += i; // авто-боксинг на каждой итерации
}

// Быстро - примитивные типы
long sumFast = 0L;
for (long i = 0; i < Integer.MAX_VALUE; i++) {
    sumFast += i; // работа с примитивами
}
```

### NullPointerException
```java
Integer number = null;
int value = number; // NullPointerException при анбоксинге!
```

## 6. Практические рекомендации

### ✅ Что делать
```java
// Используйте примитивы для локальных переменных
int counter = 0;
double price = 19.99;

// Используйте обёртки в коллекциях
List<Integer> numbers = Arrays.asList(1, 2, 3);

// Используйте equals() для сравнения
Integer x = 1000;
Integer y = 1000;
if (x.equals(y)) { /* корректное сравнение */ }
```

### ❌ Чего избегать
```java
// Не создавайте обёртки через конструктор (deprecated)
Integer bad = new Integer(42); // ❌
Integer good = Integer.valueOf(42); // ✅

// Не используйте == для сравнения обёрток
Integer a = 1000;
Integer b = 1000;
if (a == b) { /* может быть false! */ } // ❌

// Не забывайте про возможные NPE
Integer possibleNull = getMaybeNull();
int value = possibleNull; // ❌ возможен NPE
```

## 7. Ключевые выводы

1. **Автобоксинг** - автоматическое преобразование примитива → обёртка
2. **Авто-анбоксинг** - автоматическое преобразование обёртки → примитив
3. **Кэширование** - значения от -128 до 127 кэшируются для `Integer`, `Byte`, `Short`, `Long`
4. **Производительность** - избегайте ненужного боксинг/анбоксинга в циклах
5. **NPE риск** - анбоксинг `null` значения вызывает исключение
6. **Сравнение** - всегда используйте `equals()` для обёрток

# Тестирование Private Methods

## 1. Подходы к тестированию private методов

### ❌ "Не тестируйте private методы напрямую" (Это не возможно)

**Философия:** Private методы - это внутренняя реализация. Тестируйте через public API.

```java
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
    
    private int multiply(int a, int b) {
        return a * b;
    }
    
    // Вместо тестирования multiply напрямую
    public int square(int x) {
        return multiply(x, x); // Тестируем через public метод
    }
}
```

```java
// Тест через public метод
@Test
void testSquare() {
    Calculator calc = new Calculator();
    int result = calc.square(5);
    assertEquals(25, result); // Фактически тестируем private multiply
}
```

## 2. Рефлексия для тестирования private методов

### Базовый подход с Reflection

```java
import java.lang.reflect.Method;

public class PrivateMethodTest {
    
    @Test
    void testPrivateMethodWithReflection() throws Exception {
        Calculator calculator = new Calculator();
        
        // Получаем private метод
        Method privateMethod = Calculator.class
            .getDeclaredMethod("multiply", int.class, int.class);
        
        // Делаем метод доступным
        privateMethod.setAccessible(true);
        
        // Вызываем метод
        int result = (int) privateMethod.invoke(calculator, 5, 3);
        
        // Проверяем результат
        assertEquals(15, result);
    }
}
```

### Утилитный класс для рефлексии

```java
import java.lang.reflect.Method;

public class ReflectionTestUtils {
    
    public static Object invokePrivateMethod(
            Object target, 
            String methodName, 
            Class<?>[] parameterTypes,
            Object... args) throws Exception {
        
        Method method = target.getClass().getDeclaredMethod(methodName, parameterTypes);
        method.setAccessible(true);
        return method.invoke(target, args);
    }
    
    // Перегруженная версия для методов без параметров
    public static Object invokePrivateMethod(
            Object target, 
            String methodName) throws Exception {
        
        return invokePrivateMethod(target, methodName, new Class<?>[]{});
    }
}
```

```java
// Использование утилитного класса
@Test
void testWithUtilityClass() throws Exception {
    Calculator calculator = new Calculator();
    
    int result = (int) ReflectionTestUtils.invokePrivateMethod(
        calculator, 
        "multiply", 
        new Class<?>[]{int.class, int.class}, 
        5, 3
    );
    
    assertEquals(15, result);
}
```

## 3. Spring Framework - ReflectionTestUtils

Если используете Spring:

```java
import org.springframework.test.util.ReflectionTestUtils;

public class SpringPrivateMethodTest {
    
    @Test
    void testWithSpringUtils() {
        Calculator calculator = new Calculator();
        
        // Вызов private метода
        int result = (int) ReflectionTestUtils.invokeMethod(
            calculator, 
            "multiply", 
            5, 3
        );
        
        assertEquals(15, result);
    }
    
    @Test
    void testPrivateFieldAccess() {
        Calculator calculator = new Calculator();
        
        // Установка значения в private поле
        ReflectionTestUtils.setField(calculator, "internalCounter", 42);
        
        // Чтение значения из private поля
        int value = (int) ReflectionTestUtils.getField(calculator, "internalCounter");
        
        assertEquals(42, value);
    }
}
```

## 4. JUnit 5 с AssertJ

```java
import org.junit.jupiter.api.Test;
import java.lang.reflect.Method;
import static org.assertj.core.api.Assertions.assertThat;

class CalculatorTest {
    
    @Test
    void testPrivateMethodWithJUnit5() throws Exception {
        // Given
        Calculator calculator = new Calculator();
        Method privateMethod = Calculator.class
            .getDeclaredMethod("validateInput", String.class);
        privateMethod.setAccessible(true);
        
        // When
        boolean result = (boolean) privateMethod.invoke(calculator, "valid input");
        
        // Then
        assertThat(result).isTrue();
    }
}
```

## 5. Практический пример

### Тестируемый класс
```java
public class UserValidator {
    private boolean isEmailValid(String email) {
        return email != null && 
               email.contains("@") && 
               email.contains(".");
    }
    
    private String normalizeUsername(String username) {
        if (username == null) return "";
        return username.trim().toLowerCase();
    }
    
    public ValidationResult validateUser(String email, String username) {
        if (!isEmailValid(email)) {
            return ValidationResult.INVALID_EMAIL;
        }
        
        String normalized = normalizeUsername(username);
        if (normalized.length() < 3) {
            return ValidationResult.USERNAME_TOO_SHORT;
        }
        
        return ValidationResult.VALID;
    }
    
    public enum ValidationResult {
        VALID, INVALID_EMAIL, USERNAME_TOO_SHORT
    }
}
```

### Тесты private методов
```java
import org.junit.jupiter.api.Test;
import java.lang.reflect.Method;
import static org.junit.jupiter.api.Assertions.*;

class UserValidatorTest {
    
    @Test
    void testIsEmailValid_ValidEmail() throws Exception {
        UserValidator validator = new UserValidator();
        Method method = UserValidator.class.getDeclaredMethod("isEmailValid", String.class);
        method.setAccessible(true);
        
        assertTrue((boolean) method.invoke(validator, "test@example.com"));
        assertFalse((boolean) method.invoke(validator, "invalid-email"));
        assertFalse((boolean) method.invoke(validator, null));
    }
    
    @Test
    void testNormalizeUsername() throws Exception {
        UserValidator validator = new UserValidator();
        Method method = UserValidator.class.getDeclaredMethod("normalizeUsername", String.class);
        method.setAccessible(true);
        
        assertEquals("john", method.invoke(validator, "JOHN"));
        assertEquals("john doe", method.invoke(validator, "  John Doe  "));
        assertEquals("", method.invoke(validator, null));
    }
    
    // Тестирование через public API
    @Test
    void testValidateUser_Integration() {
        UserValidator validator = new UserValidator();
        
        assertEquals(
            UserValidator.ValidationResult.VALID,
            validator.validateUser("test@example.com", "username")
        );
        
        assertEquals(
            UserValidator.ValidationResult.INVALID_EMAIL,
            validator.validateUser("invalid", "username")
        );
    }
}
```
## 7. Лучшие практики

### ✅ Рекомендуется
```java
// 1. Тестируйте через public методы
@Test
void testBusinessLogic() {
    Service service = new Service();
    Result result = service.process(input);
    assertExpectedBehavior(result);
}

// 2. Рефакторинг в отдельный класс
// Если private метод становится сложным
class ComplexLogic {
    public static int performCalculation(int a, int b) {
        // Бывший private метод
    }
}

// 3. Используйте package-private доступ для тестирования
class MyClass {
    int methodForTesting() { // package-private
        // Логика для тестов
    }
}
```

## 8. Альтернативы
### Использование package-private модификатора
```java
// В основном коде
class DataProcessor {
    int processData(String input) { // package-private для тестов
        // сложная логика
    }
}

// В тестах (в том же package)
class DataProcessorTest {
    @Test
    void testProcessData() {
        DataProcessor processor = new DataProcessor();
        int result = processor.processData("test");
        assertEquals(42, result);
    }
}
```

## 🎯 Итоговые рекомендации

1. **Сначала попробуйте тестировать через public API**
2. **Если необходимо - используйте рефлексию через Spring Test или утилитные классы**
3. **Рассмотрите рефакторинг сложной private логики в отдельные классы**
4. **Избегайте PowerMock для новых проектов**
5. **Помните: частые тесты private методов могут указывать на проблемы дизайна**

Тестирование private методов иногда необходимо, но всегда стоит задать вопрос: "Не пора ли пересмотреть архитектуру класса?"


Отличная самокритика! Это ценный опыт. Давайте систематизируем пробелы и заполним их. Вот подробный разбор тем, в которых были ошибки, в формате конспекта для изучения.

## ArrayList и LinkedList , HashMap и HashSet

## 1. Сложность O(1) - фундаментальное непонимание

### ✅ Правильное понимание

**O(1) - константное время**
```java
// Примеры O(1) операций:
array[5] = 10;           // доступ по индексу в массиве
hashMap.get("key");      // получение из HashMap (в среднем)
arrayList.get(100);      // доступ по индексу в ArrayList
```

**Что значит O(1):**
- Время выполнения **не зависит** от количества элементов
- Операция выполняется за **постоянное время**
- Кривая на графике: **горизонтальная линия**

```java
// Визуализация
public class ConstantTimeDemo {
    public void demonstrateO1(int[] array, int index) {
        // Неважно, массив из 10 или 10_000 элементов
        int value = array[index]; // O(1) - всегда одно действие
    }
}
```

## 2. ArrayList vs LinkedList - ключевые различия
### ✅ Сравнительная таблица

| Характеристика | ArrayList | LinkedList |
|----------------|-----------|------------|
| **Внутренняя структура** | Динамический массив | Двусвязный список |
| **Доступ по индексу** | `O(1)` - быстрый | `O(n)` - медленный |
| **Добавление в начало** | `O(n)` - медленно | `O(1)` - быстро |
| **Добавление в конец** | `O(1)` (амортизированное) | `O(1)` |
| **Вставка в середину** | `O(n)` | `O(n)` (поиск) + `O(1)` (вставка) |
| **Удаление по индексу** | `O(n)` | `O(n)` |

### 📊 Визуализация операций

```java
// ArrayList - быстрый доступ, медленная вставка
List<Integer> arrayList = new ArrayList<>();
arrayList.add(1);    // O(1)
arrayList.add(2);    // O(1)  
arrayList.add(3);    // O(1)
arrayList.get(1);    // O(1) - мгновенно!
arrayList.add(1, 5); // O(n) - нужно сдвигать элементы

// LinkedList - медленный доступ, быстрая вставка
List<Integer> linkedList = new LinkedList<>();
linkedList.add(1);    // O(1)
linkedList.add(2);    // O(1)
linkedList.add(3);    // O(1)  
linkedList.get(1);    // O(n) - нужно пройти по цепочке
linkedList.add(1, 5); // O(1) - после нахождения позиции
```

## 3. HashMap и хеширование - критически важная тема
### ✅ Реальность хеширования

**Хеш-функция:**
```java
public int hashCode() {
    // Преобразует объект в int (32 бита)
    // Возможных значений: 2³² ≈ 4 миллиарда
}
```

**Коллизии НЕИЗБЕЖНЫ:**
```java
// Разные объекты → одинаковый хеш = КОЛЛИЗИЯ
String str1 = "Aa";
String str2 = "BB";

System.out.println(str1.hashCode()); // 2112
System.out.println(str2.hashCode()); // 2112 - КОЛЛИЗИЯ!
```

**Почему коллизии неизбежны:**
- Бесконечное количество возможных объектов
- Конечное количество хешей (2³²)
- **Принцип Дирихле**: если n+1 объектов размещать в n ячейках, минимум в одной будет ≥2 объектов

### 🔧 Как HashMap решает коллизии

**Метод цепочек:**
```java
// Упрощенная структура HashMap
class HashMap<K,V> {
    Node<K,V>[] table; // массив бакетов (корзин)
    
    static class Node<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next; // ссылка на следующий при коллизии
    }
}
```

**Пример коллизии:**
```java
HashMap<String, Integer> map = new HashMap<>();
map.put("Aa", 1);   // хеш = 2112 → бакет 5
map.put("BB", 2);   // хеш = 2112 → бакет 5 (КОЛЛИЗИЯ!)

// В бакете 5 образуется цепочка:
// ["Aa"=1] → ["BB"=2]
```

## 4. Время выполнения операций в Map

### ✅ Таблица сложности для HashMap

| Операция | В среднем | В худшем случае |
|----------|-----------|-----------------|
| `put(K, V)` | `O(1)` | `O(n)` |
| `get(K)` | `O(1)` | `O(n)` |
| `containsKey(K)` | `O(1)` | `O(n)` |
| `remove(K)` | `O(1)` | `O(n)` |
| `containsValue(V)` | `O(n)` | `O(n)` |

### 📈 Почему худший случай O(n)

```java
// Плохой hashCode - все объекты в один бакет
class BadKey {
    @Override
    public int hashCode() {
        return 1; // Всегда один и тот же хеш
    }
}

HashMap<BadKey, String> badMap = new HashMap<>();
for (int i = 0; i < 1000; i++) {
    badMap.put(new BadKey(), "value" + i);
    // Все элементы в одной цепочке
    // Поиск превращается в O(n) обход списка
}
```

## 6. HashSet - внутреннее устройство

### ✅ Реализация HashSet

**HashSet = HashMap под капотом:**
```java
// Реальная реализация:
public class HashSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();
    
    public boolean add(E e) {
        return map.put(e, PRESENT) == null;
    }
}
```

**Время выполнения HashSet:**
- `add()`: O(1) в среднем
- `contains()`: O(1) в среднем  
- `remove()`: O(1) в среднем

## 7. Практические примеры для запоминания

### 🎯 Ключевые моменты ArrayList
```java
List<String> list = new ArrayList<>();

// Добавление в конец - O(1) (амортизированное)
for (int i = 0; i < 1000; i++) {
    list.add("element " + i); // Быстро!
}

// Доступ по индексу - O(1)
String element = list.get(500); // Мгновенно!

// Вставка в начало - O(n)
list.add(0, "new first"); // Медленно! Сдвигает все элементы
```

### 🎯 Ключевые моменты HashMap
```java
Map<String, Integer> map = new HashMap<>();

// put() - O(1) в среднем
map.put("key1", 1);
map.put("key2", 2);

// get() - O(1) в среднем
Integer value = map.get("key1"); // Быстро!

// Коллизия - нормальная ситуация
map.put("Aa", 10);
map.put("BB", 20); // Та же корзина, но разные ключи
```
## 🎯 Итоговые ключевые идеи

1. **O(1)** ≠ "мгновенно", а "не зависит от размера данных"
2. **Коллизии в HashMap** - норма, а не исключение
3. **Хеш** - однонаправленная функция, нельзя восстановить исходное значение
4. **ArrayList** - быстрый доступ, медленная вставка в начало
5. **LinkedList** - медленный доступ, быстрая вставка в начало
6. **HashMap.get()** в среднем O(1), но может деградировать до O(n)

Теперь у вас есть четкий план для устранения пробелов! Удачи в подготовке! 🚀