
## ORM
- **ORM (Object-Relational Mapping)** — технология связывания объектов языка программирования с таблицами БД.  
- Hibernate позволяет работать с объектами (Entity), а не напрямую с SQL-запросами.

---

## Связь класса и таблицы
- `@Entity` — помечает класс как сущность для БД.  
- `@Table(name = "table_name")` — задаёт имя таблицы в БД.  
- `@Column(name = "column_name")` — указывает имя колонки.  

Пример:
```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_name")
    private String name;
}```
#### ID

```java
@Id  — обозначает первичный ключ.
@GeneratedValue — указывает стратегию генерации ID (например, автоинкремент).```

Возможные типы: Long, Integer, UUID, String.
### Основные аннотации и пояснения
```java

@Entity — сущность Hibernate, связанная с таблицей.
@Table(name = "...") — имя таблицы, если отличается от имени класса.
@Id — первичный ключ.
@GeneratedValue — стратегия генерации ключа (AUTO, IDENTITY, SEQUENCE, TABLE).
@Column — настройка колонки (имя, nullable, unique).
@OneToOne — связь один-к-одному (обычно через внешний ключ).
@OneToMany — один-ко-многим (список зависимых объектов).
@ManyToOne — многие-к-одному (многие сущности ссылаются на одну).
@ManyToMany — многие-ко-многим (через промежуточную таблицу).
@JoinColumn — указывает внешний ключ для связи.
@JoinTable — описывает промежуточную таблицу для @ManyToMany.
```
### Примеры связей

#### One-to-One
```java

@OneToOne
@JoinColumn(name = "passport_id")
private Passport passport;
//У каждого пользователя может быть только один паспорт, а у паспорта только один владелец.
```
#### One-to-Many / Many-to-One
```java

@OneToMany(mappedBy = "category", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
private List<Product> products;


@ManyToOne
@JoinColumn(name = "category_id")
private Category category;
```
**Один-ко-многим**: у категории может быть много продуктов.  
**Многие-к-одному**: каждый продукт относится к одной категории.
#### Many-to-Many
```java
@ManyToMany
@JoinTable(
  name = "student_course",
  joinColumns = @JoinColumn(name = "student_id"),
  inverseJoinColumns = @JoinColumn(name = "course_id")
)
private List<Course> courses;
```
Студент может записаться на много курсов, и курс может принадлежать многим студентам. Для этого создаётся промежуточная таблица.
### Fetch и Cascade
#### Fetch

- **FetchType.LAZY** — данные подгружаются только при обращении (экономит ресурсы).
- **FetchType.EAGER** — данные загружаются сразу вместе с объектом.

Пример:

```java
@OneToMany(mappedBy = "category", fetch = FetchType.LAZY) 
private List<Product> products;
```

При загрузке категории продукты **не загружаются сразу**, а только когда вызовем `getProducts()`.

```java
@OneToMany(mappedBy = "category", fetch = FetchType.EAGER)
private List<Product> products;
```
При загрузке категории Hibernate сразу подтянет связанные продукты.

FetchType:
	LAZY — загрузка по требованию.
	EAGER — загрузка сразу вместе с объектом.
CascadeType:
	ALL — все операции.
	PERSIST — каскадное сохранение.
	REMOVE — каскадное удаление.
	MERGE — объединение изменений.
	REFRESH — обновление.
	DETACH — отсоединение.

#### Cascade

- **CascadeType.ALL** — применять все операции к зависимым объектам.
- **CascadeType.PERSIST** — сохранять вместе с родителем.
- **CascadeType.REMOVE** — удалять вместе с родителем.
- **CascadeType.MERGE** — обновлять вместе с родителем.

Пример:
```java
@OneToMany(mappedBy = "category", cascade = CascadeType.ALL) private List<Product> products;
```
Если сохранить или удалить категорию — её продукты будут сохранены или удалены автоматически.
```java
@OneToMany(mappedBy = "category", cascade = CascadeType.PERSIST) private List<Product> products;
```

При сохранении категории Hibernate автоматически сохранит все её продукты, но удалять их не будет.
## Жизненный цикл сущности в Hibernate (очень мало вероятно , что спро)

Состояния:

1. **Transient**  
   - Объект создан `new`, но ещё не связан с БД.  
   - В БД записи нет.  
   - Hibernate не отслеживает его.  
   - Пример:
 ```java
     Product p = new Product(); // transient
     ```

2. **Persistent**  
   - Объект сохранён в сессии (`session.save(p)` или загружен через `session.get()`).  
   - Hibernate отслеживает изменения и синхронизирует их с БД при `commit()`.  
   - Пример:
 ```java
     session.save(p); // теперь persistent
     ```

3. **Detached**  
   - Сессия закрыта, но объект остался в памяти.  
   - Hibernate больше не отслеживает его.  
   - Если поменять поля — изменения **не сохранятся**.  
   - Пример:
 ```java
     session.close();
     p.setName("New name"); // detached
     ```

4. **Removed**  
   - Объект помечен на удаление (`session.delete(p)`).  
   - После коммита будет удалён из БД.  
   - Пример:
 ```java
     session.delete(p); // removed
     ```

## Пример жизненного цикла сущности в Hibernate

```java
// Создаём новый объект (еще нет в БД)
Product product = new Product();  // состояние: Transient
product.setName("Laptop");
product.setPrice(1000);

// Открываем сессию
Session session = sessionFactory.openSession();
session.beginTransaction();

// Сохраняем объект → теперь Hibernate отслеживает его
session.save(product);            // состояние: Persistent

// Меняем поле — Hibernate сам обновит его в БД при commit()
product.setPrice(1200);

// Закрываем сессию → объект отсоединяется
session.getTransaction().commit();
session.close();                  // состояние: Detached

// Изменение теперь не попадёт в БД
product.setPrice(1500);  // Hibernate не знает про это изменение

// Открываем новую сессию
session = sessionFactory.openSession();
session.beginTransaction();

// Удаляем объект
session.delete(product);          // состояние: Removed

session.getTransaction().commit();
session.close();
```


