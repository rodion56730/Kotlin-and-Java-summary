# StringBuilder для собеседования - полный разбор

## 📝 Основная концепция

**StringBuilder** - это изменяемая последовательность символов, в отличие от неизменяемых String объектов.

### Ключевые характеристики:
- **Изменяемый** (mutable)
- **Не синхронизированный** (не thread-safe)
- **Высокая производительность** при множественных операциях

## 🔧 Зачем нужен? Основная проблема

```java
// ПРОБЛЕМА - медленно и ресурсоемко
String str = "";
for (int i = 0; i < 1000; i++) {
    str += i; // На каждой итерации создается новый String объект!
}
// Создается 1000 объектов String + нагрузка на GC
```

**StringBuilder решает эту проблему:**
```java
// РЕШЕНИЕ - быстро и эффективно
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i); // Работает с одним внутренним массивом
}
String result = sb.toString(); // Только один String создается
```

## ⚙️ Как работает внутренне

### Внутренняя структура:
```java
public final class StringBuilder {
    char[] value;    // Внутренний массив символов
    int count;       // Текущее количество символов
}
```

**Принцип работы:**
1. Создается начальный массив (обычно 16 символов)
2. При добавлении данных массив расширяется по формуле:
   ```java
   newCapacity = (oldCapacity * 2) + 2
   ```
3. Данные копируются в новый увеличенный массив

### Пример расширения:
```java
StringBuilder sb = new StringBuilder(); // capacity = 16
sb.append("Hello World!"); // count = 12, capacity = 16
sb.append(" This is a long text..."); // count > 16 → capacity = 34
```

## 📚 Основные методы

### Создание:
```java
StringBuilder sb1 = new StringBuilder();        // capacity = 16
StringBuilder sb2 = new StringBuilder(50);      // capacity = 50
StringBuilder sb3 = new StringBuilder("Hello"); // capacity = 16 + 5
```

### Основные операции:
```java
StringBuilder sb = new StringBuilder();

// Добавление
sb.append("Hello");              // "Hello"
sb.append(123);                  // "Hello123"
sb.append(true);                 // "Hello123true"

// Вставка
sb.insert(5, " ");              // "Hello 123true"
sb.insert(9, "ABC");            // "Hello 123ABCtrue"

// Удаление
sb.delete(9, 12);               // "Hello 123true"
sb.deleteCharAt(5);             // "Hello123true"

// Замена
sb.replace(5, 8, " World");     // "Hello Worldtrue"

// Реверс
sb.reverse();                   // "eurt dlroW olleH"
sb.reverse();                   // "Hello Worldtrue"

// Получение и установка
char ch = sb.charAt(0);         // 'H'
sb.setCharAt(0, 'h');           // "hello Worldtrue"

// Длина и capacity
int len = sb.length();          // 16
int cap = sb.capacity();        // 34 (или больше)
```

## ⚡ Производительность - сравнение

### Бенчмарк конкатенации:
```java
// String concatenation - медленно
long start = System.currentTimeMillis();
String result = "";
for (int i = 0; i < 100000; i++) {
    result += i;
}
long time1 = System.currentTimeMillis() - start;

// StringBuilder - быстро
start = System.currentTimeMillis();
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100000; i++) {
    sb.append(i);
}
String result2 = sb.toString();
long time2 = System.currentTimeMillis() - start;

System.out.println("String time: " + time1 + "ms");   // ~4000ms
System.out.println("StringBuilder time: " + time2 + "ms"); // ~10ms
```

## 🔄 StringBuffer vs StringBuilder

| Характеристика | StringBuilder | StringBuffer |
|----------------|---------------|--------------|
| **Синхронизация** | Нет (быстрее) | Да (thread-safe) |
| **Производительность** | Выше | Ниже |
| **С версии** | Java 5 | Java 1.0 |
| **Использование** | Однопоточные приложения | Многопоточные приложения |

```java
// StringBuilder - для однопоточных приложений
StringBuilder sb = new StringBuilder();

// StringBuffer - для многопоточных приложений
StringBuffer sbf = new StringBuffer();
```

## 💡 Лучшие практики

### 1. Инициализация с правильным capacity:
```java
// ПЛОХО - может быть много расширений массива
StringBuilder sb = new StringBuilder();

// ХОРОШО - минимизирует расширения массива
StringBuilder sb = new StringBuilder(estimatedSize);
```

### 2. Цепочка вызовов (method chaining):
```java
String result = new StringBuilder()
    .append("Hello ")
    .append(name)
    .append(", your age is ")
    .append(age)
    .append(" years.")
    .toString();
```

### 3. Использование в циклах:
```java
// ПРАВИЛЬНО - для множественных операций
public String buildSQL(List<String> conditions) {
    StringBuilder sql = new StringBuilder("SELECT * FROM users");
    if (!conditions.isEmpty()) {
        sql.append(" WHERE ");
        for (String condition : conditions) {
            sql.append(condition).append(" AND ");
        }
        sql.setLength(sql.length() - 5); // Удаляем последний " AND "
    }
    return sql.toString();
}
```

## 🎯 Ключевые моменты для собеседования

### Обязательно упомянуть:
1. **Изменяемость** - в отличие от String
2. **Производительность** - особенно в циклах
3. **Внутренний массив** char[] и механизм расширения
4. **Не синхронизирован** - отличие от StringBuffer
5. **Method chaining** - методы возвращают this

### Частые вопросы на собеседовании:

**Q: Когда использовать StringBuilder вместо String?**
A: При множественных операциях модификации строк, особенно в циклах.

**Q: В чем разница между StringBuilder и StringBuffer?**
A: StringBuffer синхронизирован (thread-safe), но медленнее. StringBuilder быстрее, но не thread-safe.

**Q: Как работает внутренний массив StringBuilder?**
A: Начинается с capacity 16, расширяется по формуле (n*2)+2 при необходимости.

**Q: Когда String конкатенация лучше?**
A: Для простых операций или когда компилятор может оптимизировать (например, `"a" + "b"` компилятор преобразует в StringBuilder).

## 🚀 Примеры использования

### 1. Построение сложных строк:
```java
public String buildEmail(String firstName, String lastName, String domain) {
    return new StringBuilder()
        .append(firstName.toLowerCase())
        .append(".")
        .append(lastName.toLowerCase())
        .append("@")
        .append(domain)
        .toString();
}
```

### 2. Эффективное удаление символов:
```java
public String removeSpaces(String input) {
    StringBuilder sb = new StringBuilder(input.length());
    for (char c : input.toCharArray()) {
        if (c != ' ') {
            sb.append(c);
        }
    }
    return sb.toString();
}
```

### 3. Многократные модификации:
```java
public String processText(String text) {
    StringBuilder sb = new StringBuilder(text);
    
    // Удаляем первый символ
    sb.deleteCharAt(0);
    
    // Добавляем в конец
    sb.append(" processed");
    
    // Вставляем в середину
    sb.insert(sb.length() / 2, " MIDDLE ");
    
    return sb.toString();
}
```

**Итог для собеседования:** StringBuilder - это инструмент для эффективной работы со строками при множественных операциях модификации, который избегает создания множества временных String объектов за счет использования изменяемого внутреннего массива символов.