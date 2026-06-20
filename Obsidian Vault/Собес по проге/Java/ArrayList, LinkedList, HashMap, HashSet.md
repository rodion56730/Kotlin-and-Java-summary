
## 1. ArrayList - динамический массив

### 🏗️ Внутреннее устройство

```java
public class ArrayList<E> {
    private static final int DEFAULT_CAPACITY = 10;
    private Object[] elementData;  // Внутренний массив
    private int size;              // Текущее количество элементов
}
```

### 📈 Механизм роста

```java
// Изначально: capacity = 10
ArrayList<Integer> list = new ArrayList<>();

// При добавлении 11-го элемента:
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // Увеличение в 1.5 раза
    elementData = Arrays.copyOf(elementData, newCapacity);
}

// Пример роста:
list.add(1);  // capacity = 10
// ... добавляем 10 элементов
list.add(11); // capacity = 15 (10 * 1.5)
list.add(16); // capacity = 22 (15 * 1.5)
```

### ⚡ Время выполнения операций

```java
ArrayList<String> list = new ArrayList<>();

// O(1) - константное время
list.add("element");     // Добавление в конец (амортизированное)
list.get(5);             // Доступ по индексу
list.size();             // Получение размера

// O(n) - линейное время
list.add(0, "first");    // Вставка в начало
list.remove(0);          // Удаление из начала
list.contains("value");  // Поиск элемента
list.remove("element");  // Удаление по значению
```

### 🎯 Практический пример

```java
public class ArrayListDemo {
    public static void main(String[] args) {
        ArrayList<Integer> numbers = new ArrayList<>();
        
        // Быстрые операции
        for (int i = 0; i < 1000000; i++) {
            numbers.add(i); // O(1) - добавление в конец
        }
        
        // Быстрый доступ
        int value = numbers.get(500000); // O(1) - мгновенный доступ
        
        // Медленные операции
        numbers.add(0, -1); // O(n) - нужно сдвинуть 1 млн элементов!
        numbers.remove(0);   // O(n) - снова сдвиг всех элементов
    }
}
```

## 2. LinkedList - двусвязный список

### 🏗️ Внутренняя структура

```java
public class LinkedList<E> {
    Node<E> first;  // Первый элемент
    Node<E> last;   // Последний элемент
    int size = 0;
    
    private static class Node<E> {
        E item;         // Данные
        Node<E> next;   // Следующий элемент
        Node<E> prev;   // Предыдущий элемент
        
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
}
```

### 🔗 Визуализация связей

```
[prev|A|next] <-> [prev|B|next] <-> [prev|C|next]
```

### ⚡ Время выполнения операций

```java
LinkedList<String> list = new LinkedList<>();

// O(1) - константное время
list.add("element");     // Добавление в конец
list.addFirst("first");  // Добавление в начало  
list.addLast("last");    // Добавление в конец
list.removeFirst();      // Удаление из начала
list.removeLast();       // Удаление из конца

// O(n) - линейное время
list.get(5);             // Доступ по индексу
list.add(3, "middle");   // Вставка по индексу
list.remove(3);          // Удаление по индексу
list.contains("value");  // Поиск элемента
```

### 🎯 Практический пример

```java
public class LinkedListDemo {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>();
        
        // Быстрые операции с началом и концом
        list.addFirst("first");  // O(1)
        list.addLast("last");    // O(1)
        list.removeFirst();      // O(1)
        
        // Медленный доступ по индексу
        for (int i = 0; i < 100000; i++) {
            list.add("element" + i);
        }
        
        // Медленно - нужно пройти по цепочке
        String element = list.get(50000); // O(n) - проходит 50000 узлов!
    }
}
```

## 3. HashMap - хеш-таблица

### 🏗️ Внутреннее устройство (Java 8+)

```java
public class HashMap<K,V> {
    Node<K,V>[] table;     // Массив бакетов (корзин)
    int size;              // Количество элементов
    float loadFactor = 0.75; // Коэффициент загрузки
    int threshold;         // Порог расширения (capacity * loadFactor)
    
    static class Node<K,V> {
        final int hash;    // Хеш ключа
        final K key;       // Ключ
        V value;           // Значение
        Node<K,V> next;    // Следующий узел в цепочке
    }
    
    // Java 8+: при длине цепочки > 8 превращается в дерево
    static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // Родительский узел
        TreeNode<K,V> left;    // Левый потомок
        TreeNode<K,V> right;   // Правый потомок
    }
}
```

### 🔄 Процесс работы HashMap

#### 1. Вычисление индекса бакета
```java
// Псевдокод вычисления индекса
int hash = key.hashCode();
int index = (table.length - 1) & hash; // Побитовое И для получения индекса

// Пример:
String key = "hello";
int hash = key.hashCode();          // 99162322
int index = (16 - 1) & hash;        // 15 & 99162322 = 2
// Элемент попадет в бакет №2
```

#### 2. Разрешение коллизий
```java
// Метод цепочек (chaining)
// Бакет 0: [key1=value1] → [key2=value2] → [key3=value3]

// При достижении порога цепочка превращается в дерево
// Бакет 1: TreeNode{keyA=valueA, left=keyB, right=keyC}
```

### 📈 Механизм расширения (rehashing)

```java
HashMap<String, Integer> map = new HashMap<>(); // capacity = 16

// При добавлении 12-го элемента (16 * 0.75 = 12)
// Происходит расширение:
void resize() {
    int newCapacity = oldCapacity << 1; // Удвоение размера (32)
    // Все элементы перераспределяются по новым бакетам
    // index = (newCapacity - 1) & hash
}
```

### ⚡ Время выполнения операций

```java
HashMap<String, Integer> map = new HashMap<>();

// O(1) в среднем случае
map.put("key", 1);        // Вставка
map.get("key");           // Получение
map.containsKey("key");   // Проверка ключа
map.remove("key");        // Удаление

// O(n) в худшем случае (все элементы в одном бакете)
map.get("bad_key");       // Деградация до O(n)

// O(n) всегда
map.containsValue(1);     // Поиск по значению
```

### 🎯 Практический пример

```java
public class HashMapDemo {
    public static void main(String[] args) {
        HashMap<Student, Integer> students = new HashMap<>();
        
        Student s1 = new Student("Alice", 1);
        Student s2 = new Student("Bob", 2);
        
        // Быстрые операции
        students.put(s1, 95);
        students.put(s2, 87);
        
        Integer score = students.get(s1); // O(1) - быстрый поиск
        
        // Коллизия
        Student s3 = new Student("Aa", 3);
        Student s4 = new Student("BB", 4); // Такие же хеши как "Aa"
        
        students.put(s3, 90);
        students.put(s4, 92); // Оба попадут в один бакет
    }
}

class Student {
    private String name;
    private int id;
    
    @Override
    public int hashCode() {
        return Objects.hash(name, id);
    }
    
    @Override 
    public boolean equals(Object o) {
        // правильная реализация
    }
}
```

## 4. HashSet - множество на основе HashMap

### 🏗️ Внутренняя реализация

```java
public class HashSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();
    
    public HashSet() {
        map = new HashMap<>();
    }
    
    public boolean add(E e) {
        return map.put(e, PRESENT) == null;
    }
    
    public boolean contains(Object o) {
        return map.containsKey(o);
    }
    
    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }
}
```

### 🔄 Как это работает

```java
HashSet<String> set = new HashSet<>();

// Добавление элемента
set.add("apple"); 
// Внутри: map.put("apple", PRESENT)

// Проверка наличия
set.contains("apple");
// Внутри: map.containsKey("apple")

// Удаление
set.remove("apple");
// Внутри: map.remove("apple")
```

### ⚡ Время выполнения операций

```java
HashSet<Integer> set = new HashSet<>();

// O(1) в среднем случае
set.add(1);              // Добавление
set.contains(1);         // Проверка наличия
set.remove(1);           // Удаление

// O(n) в худшем случае
set.contains(bad_value); // Деградация до O(n)
```

### 🎯 Практический пример

```java
public class HashSetDemo {
    public static void main(String[] args) {
        HashSet<String> uniqueWords = new HashSet<>();
        
        // Добавление уникальных элементов
        String[] words = {"apple", "banana", "apple", "orange", "banana"};
        
        for (String word : words) {
            uniqueWords.add(word); // Дубликаты игнорируются
        }
        
        System.out.println(uniqueWords); // [apple, banana, orange]
        System.out.println(uniqueWords.size()); // 3
        
        // Быстрый поиск
        boolean hasApple = uniqueWords.contains("apple"); // O(1)
        boolean hasGrape = uniqueWords.contains("grape"); // O(1)
    }
}
```

## 📊 Сравнительная таблица коллекций

| Коллекция | Внутренняя структура | Доступ по индексу | Добавление | Удаление | Поиск |
|-----------|---------------------|-------------------|------------|----------|--------|
| **ArrayList** | Динамический массив | O(1) | O(1) конец<br>O(n) начало | O(n) | O(n) |
| **LinkedList** | Двусвязный список | O(n) | O(1) | O(1) начало/конец<br>O(n) по индексу | O(n) |
| **HashMap** | Массив + цепочки/деревья | - | O(1) | O(1) | O(1) ключ<br>O(n) значение |
| **HashSet** | HashMap (ключи) | - | O(1) | O(1) | O(1) |

## 🎯 Рекомендации по выбору коллекции

### ✅ Когда использовать ArrayList:
```java
// Частый доступ по индексу
List<String> users = new ArrayList<>();
String user = users.get(150); // Быстро!

// Добавление только в конец
for (int i = 0; i < 10000; i++) {
    users.add("user" + i); // Эффективно
}
```

### ✅ Когда использовать LinkedList:
```java
// Частые операции с началом/концом
LinkedList<Process> queue = new LinkedList<>();
queue.addFirst(process);  // Быстро!
queue.removeLast();       // Быстро!

// Частые вставки/удаления в середине (если есть итератор)
ListIterator<String> iter = list.listIterator();
while (iter.hasNext()) {
    if (condition) {
        iter.add("new element"); // O(1) при использовании итератора
    }
}
```

### ✅ Когда использовать HashMap:
```java
// Быстрый поиск по ключу
HashMap<String, User> userCache = new HashMap<>();
User user = userCache.get("user123"); // Мгновенно!

// Хранение пар ключ-значение
HashMap<Integer, String> errorCodes = new HashMap<>();
errorCodes.put(404, "Not Found");
errorCodes.put(500, "Internal Error");
```

### ✅ Когда использовать HashSet:
```java
// Проверка уникальности
HashSet<String> visitedUrls = new HashSet<>();
if (!visitedUrls.contains(url)) { // Быстрая проверка
    visitedUrls.add(url);
    // Обработка URL
}

// Удаление дубликатов
List<String> withDuplicates = Arrays.asList("A", "B", "A", "C");
HashSet<String> unique = new HashSet<>(withDuplicates); // [A, B, C]
```

## 🔧 Важные особенности

### Итерация по коллекциям

```java
// ArrayList - быстрая итерация
for (int i = 0; i < arrayList.size(); i++) {
    String element = arrayList.get(i); // O(1) на каждой итерации
}

// LinkedList - медленная итерация по индексу
for (int i = 0; i < linkedList.size(); i++) {
    String element = linkedList.get(i); // O(n) на каждой итерации!
}

// LinkedList - быстрая итерация через итератор
for (String element : linkedList) {
    // O(1) переход к следующему элементу
}

// HashMap - итерация по ключам/значениям
for (String key : hashMap.keySet()) { }       // O(n)
for (Integer value : hashMap.values()) { }    // O(n)
for (Map.Entry<String, Integer> entry : hashMap.entrySet()) { } // O(n)
```

Теперь у вас есть полное понимание этих коллекций! 🚀
### ArrayList
ArrayList - класс, реализующий интерфейс [[List]] с помощью [[Массивы|массива]].
Так как размер массива ограничен, и его нельзя расширять, отдельно в памяти хранится текущий размер, отдельно вместимость.
При создании задается некоторая достаточно большая вместимость.
Если место заканчивается, создается новый массив удвоенной вместимости, все данные переносятся в него.

### LinkedList
LinkedList - класс, реализующий List с помощью двусвязного списка.
В каждой ячейке кроме собственно значения хранится ссылка на предыдущую и следующую ячейку (в первой ячейке ссылка не предыдущую [[null]], аналогично в последней ячейке ссылка на следующую).
Экземпляр также хранит ссылки на начало и конец списка

### Сравнение
|Параметр|ArrayList|LinkedList|
|:----------:|:---------:|:---------:|
|Доступ по индексу|O(1)|O(n)|
|Вставка в конец|O(1)/O(n)|O(1)|
|Вставка в начало|O(n)|O(1)|
|Вставка в произвольное место|O(n)|O(1)/O(n)|
|Удаление из конца|O(1)|O(1)|
|Удаление из начала|O(n)|O(1)|
|Удаление из произвольного места|O(n)|O(1)/O(n)|
|Перебор элементов|O(n)|O(n)|

Вставка и удаление по произвольному индексу в LinkedList выполняются быстро, если итератор уже расположен на необходимом элементе
