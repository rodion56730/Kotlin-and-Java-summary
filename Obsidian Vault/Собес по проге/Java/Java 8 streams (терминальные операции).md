
Stream - интерфейс, API для работы со структурами данных путем упорядочивания и конвейерной обработки. Не является структурой данных, никогда не изменяет источник данных.
Может использовать данные из коллекций.
Создание:
* Stream.empty()
* Stream.Builder
* Stream.of(E...)
* Arrays.stream(E\[])
* Stream.concat(Stream\<E>...)
* метод .stream() из **Collection**

Операции делятся на промежуточные и терминальные:
* Промежуточные выполняются "лениво", т.е. только при необходимости считать и вернуть результат и при встрече с терминальной операцией.
	* filter(Predicate\<? super E>)
	* map(Function\<? super E, ? extends R>)
	* distinct() - отфильтровывает дубликаты
	* peek(Consumer\<? super E>) - выполняет действие над каждым элементом, ничего не изменяет
	* sorted(), sorted(Comparator\<? super E>) - сортирует по возрастанию/ по Comparator
* Терминальные операции завершают промежуточные операции, возвращают итоговый результат. После них с потоком работать нельзя
	* forEach(Consumer\<? super E>) - выполняет действие над каждым элементом, ничего не изменяет
	* count() - количество элементов
	* collect(Collector) - собирает элементы в коллекцию
	* min(), max()
	* findFirst() - возвращает первый элемент как **Optional**. На пустом потоке вернет empty
	* findAny() - возвращает случайный элемент потока как **Optional**. На пустом потоке вернет empty
	* allMatch(Predicate\<? super E>), anyMatch(Predicate\<? super E>), noneMatch(Predicate\<? super E>) - все/хоть один/ни один соответствуют предикату
Отличный материал для собеседования! Вот примеры кода, которые иллюстрируют все эти концепции:

## 🎯 Создание Stream

```java
import java.util.*;
import java.util.stream.*;

public class StreamExamples {
    public static void main(String[] args) {
        // Различные способы создания Stream
        System.out.println("=== Создание Stream ===");
        
        // 1. Stream.empty()
        Stream<String> emptyStream = Stream.empty();
        System.out.println("Empty stream count: " + emptyStream.count());
        
        // 2. Stream.Builder
        Stream.Builder<String> builder = Stream.builder();
        builder.add("Apple");
        builder.add("Banana");
        builder.add("Cherry");
        Stream<String> builtStream = builder.build();
        
        // 3. Stream.of(E...)
        Stream<String> fruitStream = Stream.of("Apple", "Banana", "Cherry", "Date");
        
        // 4. Arrays.stream(E[])
        String[] fruitsArray = {"Apple", "Banana", "Cherry", "Date"};
        Stream<String> arrayStream = Arrays.stream(fruitsArray);
        
        // 5. Stream.concat(Stream<E>...)
        Stream<String> stream1 = Stream.of("Apple", "Banana");
        Stream<String> stream2 = Stream.of("Cherry", "Date");
        Stream<String> concatenatedStream = Stream.concat(stream1, stream2);
        
        // 6. метод .stream() из Collection
        List<String> fruitList = Arrays.asList("Apple", "Banana", "Cherry", "Date");
        Stream<String> collectionStream = fruitList.stream();
        
        collectionStream.forEach(System.out::println);
    }
}
```

## 🔄 Промежуточные операции (lazy)

```java
public class IntermediateOperations {
    public static void main(String[] args) {
        List<String> fruits = Arrays.asList("Apple", "Banana", "Cherry", "Date", 
                                           "Apple", "Elderberry", "Fig", "Grape");
        
        System.out.println("=== Промежуточные операции ===");
        
        // filter(Predicate<? super E>) - фильтрация по условию
        Stream<String> filtered = fruits.stream()
            .filter(fruit -> fruit.length() > 4);
        
        // map(Function<? super E, ? extends R>) - преобразование элементов
        Stream<Integer> lengthStream = fruits.stream()
            .map(String::length);
        
        // distinct() - удаление дубликатов
        Stream<String> distinctFruits = fruits.stream()
            .distinct();
        
        // peek(Consumer<? super E>) - выполнение действия без изменения
        System.out.println("Peek example:");
        fruits.stream()
            .peek(fruit -> System.out.println("Before filter: " + fruit))
            .filter(fruit -> fruit.startsWith("A"))
            .peek(fruit -> System.out.println("After filter: " + fruit))
            .count(); // Терминальная операция для активации
        
        // sorted() - сортировка по естественному порядку
        Stream<String> sortedFruits = fruits.stream()
            .sorted();
        
        // sorted(Comparator<? super E>) - сортировка по компаратору
        Stream<String> lengthSorted = fruits.stream()
            .sorted(Comparator.comparingInt(String::length));
            
        // Демонстрация "ленивости" - без терминальной операции ничего не выполнится
        System.out.println("До терминальной операции:");
        Stream<String> lazyStream = fruits.stream()
            .filter(fruit -> {
                System.out.println("Filtering: " + fruit);
                return fruit.length() > 3;
            })
            .map(fruit -> {
                System.out.println("Mapping: " + fruit);
                return fruit.toUpperCase();
            });
        
        System.out.println("После терминальной операции:");
        lazyStream.forEach(System.out::println);
    }
}
```

## ⚡ Терминальные операции

```java
public class TerminalOperations {
    public static void main(String[] args) {
        List<String> fruits = Arrays.asList("Apple", "Banana", "Cherry", "Date", 
                                           "Elderberry", "Fig", "Grape");
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        System.out.println("=== Терминальные операции ===");
        
        // forEach(Consumer<? super E>) - выполнение действия для каждого элемента
        System.out.println("forEach:");
        fruits.stream()
            .filter(f -> f.length() > 4)
            .forEach(f -> System.out.println("Fruit: " + f));
        
        // count() - подсчет количества элементов
        long count = fruits.stream()
            .filter(f -> f.startsWith("A"))
            .count();
        System.out.println("Fruits starting with 'A': " + count);
        
        // collect(Collector) - сбор в коллекцию
        List<String> collectedList = fruits.stream()
            .filter(f -> f.length() <= 5)
            .map(String::toUpperCase)
            .collect(Collectors.toList());
        System.out.println("Collected list: " + collectedList);
        
        Set<String> collectedSet = fruits.stream()
            .collect(Collectors.toSet());
        System.out.println("Collected set: " + collectedSet);
        
        // min(), max() - поиск минимального/максимального элемента
        Optional<String> shortestFruit = fruits.stream()
            .min(Comparator.comparingInt(String::length));
        shortestFruit.ifPresent(f -> System.out.println("Shortest fruit: " + f));
        
        Optional<String> longestFruit = fruits.stream()
            .max(Comparator.comparingInt(String::length));
        longestFruit.ifPresent(f -> System.out.println("Longest fruit: " + f));
        
        // findFirst() - первый элемент
        Optional<String> firstFruit = fruits.stream()
            .filter(f -> f.length() > 5)
            .findFirst();
        System.out.println("First fruit with length > 5: " + 
                          firstFruit.orElse("Not found"));
        
        // findAny() - любой элемент (полезно в параллельных потоках)
        Optional<String> anyFruit = fruits.stream()
            .filter(f -> f.contains("a"))
            .findAny();
        System.out.println("Any fruit with 'a': " + anyFruit.orElse("Not found"));
        
        // allMatch, anyMatch, noneMatch - проверка условий
        boolean allHaveVowels = fruits.stream()
            .allMatch(f -> f.matches(".*[aeiou].*"));
        System.out.println("All fruits have vowels: " + allHaveVowels);
        
        boolean anyStartsWithZ = fruits.stream()
            .anyMatch(f -> f.startsWith("Z"));
        System.out.println("Any fruit starts with 'Z': " + anyStartsWithZ);
        
        boolean noneEmpty = fruits.stream()
            .noneMatch(String::isEmpty);
        System.out.println("No empty fruits: " + noneEmpty);
    }
}
```


