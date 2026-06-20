# 📚 Конспект: Свойства ACID в базах данных

## 🎯 Что такое ACID?

**ACID** - это набор свойств, которые гарантируют надежность и предсказуемость транзакций в базах данных.

**Транзакция** - это группа операций, которые выполняются как единое целое.

---

## 🔥 АТОМАРНОСТЬ (Atomicity)

### Простыми словами:
**"Все или ничего"** - либо выполняются ВСЕ операции транзакции, либо НИ ОДНА.

### Аналогия: 📦 **ЗАКАЗ В ИНТЕРНЕТ-МАГАЗИНЕ**
```sql
-- Транзакция оформления заказа
START TRANSACTION;

-- 1. Создаем заказ
INSERT INTO orders (user_id, total) VALUES (1, 5000);

-- 2. Списываем деньги
UPDATE accounts SET balance = balance - 5000 WHERE user_id = 1;

-- 3. Резервируем товар  
UPDATE products SET stock = stock - 1 WHERE id = 123;

-- Если ВСЕ операции успешны:
COMMIT; -- ✅ Подтверждаем изменения

-- Если ЛЮБАЯ операция failed:
ROLLBACK; -- ❌ Откатываем ВСЕ изменения
```

### Что гарантирует:
- **При ошибке** - откатываются ВСЕ изменения транзакции
- **При успехе** - применяются ВСЕ изменения
- **Нет промежуточных состояний**

---

## 🔒 СОГЛАСОВАННОСТЬ (Consistency)

### Простыми словами:
**"Правила всегда соблюдаются"** - транзакция переводит базу из одного валидного состояния в другое.

### Аналогия: 🏦 **БАНКОВСКИЙ ПЕРЕВОД**
```sql
-- Правила согласованности:
-- 1. Баланс не может быть отрицательным
-- 2. Сумма переводов должна сохраняться

START TRANSACTION;

-- Проверяем правила ДО операции
SELECT balance FROM accounts WHERE id = 1; -- 1000₽
-- Хотим перевести 1500₽ → НЕВОЗМОЖНО! Правило нарушено

-- Валидная операция:
UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- Стало: 500₽
UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- Стало: 1500₽

-- ОБЩАЯ СУММА: 500 + 1500 = 2000₽ (сохранилась!)
COMMIT;
```

### Что гарантирует:
- **Целостность данных** - соблюдение всех constraints (UNIQUE, NOT NULL, FOREIGN KEY)
- **Бизнес-правила** - кастомные ограничения
- **Инварианты** - условия, которые всегда истинны

---

## ⚡ ИЗОЛИРОВАННОСТЬ (Isolation)

### Простыми словами:
**"Транзакции не мешают друг другу"** - параллельные транзакции не видят промежуточные состояния друг друга.

### Аналогия: 🏢 **ЛИФТ С НЕСКОЛЬКИМИ КАБИНАМИ**
```sql
-- Транзакция A: Перевод денег
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- Баланс: 400₽
-- Еще не закоммитили...

-- Транзакция B: Проверка баланса (в другом соединении)
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1; -- Увидит 500₽ (старое значение)
-- Не видит незавершенных изменений транзакции A!
COMMIT;
```

### Уровни изоляции (от слабых к сильным):

#### 1. **READ UNCOMMITTED** - Чтение незакоммиченных данных
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- Можно читать "грязные" данные других транзакций
-- ❌ Риск: Dirty Reads
```

#### 2. **READ COMMITTED** - Чтение коммиченных данных (по умолчанию в PostgreSQL)
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Видны только завершенные транзакции
-- ✅ Решает: Dirty Reads
-- ❌ Риск: Non-repeatable Reads
```

#### 3. **REPEATABLE READ** - Повторяемое чтение
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- Снимок данных на начало транзакции
-- ✅ Решает: Dirty Reads, Non-repeatable Reads
-- ❌ Риск: Phantom Reads
```

#### 4. **SERIALIZABLE** - Сериализуемый
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Полная изоляция, как последовательное выполнение
-- ✅ Решает: ВСЕ проблемы
-- ❌ Цена: Производительность
```

### Проблемы изоляции:

| Проблема | Описание | Пример |
|----------|----------|---------|
| **Dirty Read** | Чтение незакоммиченных данных | Видим откаченные изменения |
| **Non-repeatable Read** | Данные меняются при повторном чтении | Баланс изменился между SELECT |
| **Phantom Read** | Появляются новые строки | Новые заказы появились между запросами |

---

## 💪 УСТОЙЧИВОСТЬ (Durability)

### Простыми словами:
**"Раз сохранилось - навсегда"** - после подтверждения транзакции изменения сохраняются даже при сбое системы.

### Аналогия: 📝 **НОТАРИАЛЬНАЯ ЗАПИСЬ**
```sql
-- После этого сообщения...
COMMIT;

-- Произошел сбой:
-- 💥 Отключение электричества
-- 💥 Падение сервера  
-- 💥 Авария диска

-- После восстановления:
SELECT * FROM accounts WHERE id = 1;
-- ✅ Изменения СОХРАНИЛИСЬ!
```

### Как обеспечивается:

#### 1. **Write-Ahead Logging (WAL)**
```bash
# Порядок записи:
1. 📝 Изменения в WAL (журнал)
2. ✅ Подтверждение COMMIT
3. 🗃️ Постепенная запись в основные таблицы
```

#### 2. **Replication**
```bash
# Копирование данных на несколько узлов
Master: COMMIT → ✅
Slave1: ✅  
Slave2: ✅
Slave3: ✅
```

#### 3. **Backup стратегии**
```bash
# Регулярное резервное копирование
Full Backup: Еженедельно
Incremental Backup: Ежедневно
WAL Archiving: Непрерывно
```

---

## 🏗️ ПРАКТИЧЕСКИЕ ПРИМЕРЫ

### Пример 1: **Банковская система**
```sql
START TRANSACTION;

-- Atomicity: Все операции или ничего
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

-- Consistency: Балансы не отрицательные, сумма сохраняется
-- Проверка constraints автоматически

-- Isolation: Другие транзакции не видят изменения до COMMIT
-- Durability: После COMMIT - изменения навсегда

COMMIT;
```

### Пример 2: **Интернет-магазин**
```java
@Transactional // Spring обеспечивает ACID
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        // Atomicity: Spring делает rollback при исключении
        Order order = orderRepository.save(request.toOrder());
        
        // Consistency: @NotNull, @Min проверяют данные
        inventoryService.reserveItems(order.getItems());
        
        // Isolation: @Transactional(isolation = Isolation.READ_COMMITTED)
        paymentService.processPayment(order);
        
        // Durability: После метода - данные в БД навсегда
        return order;
    }
}
```

---

## ⚠️ КОМПРОМИССЫ И БАЛАНС

### Производительность vs Надежность:

| Уровень | Производительность | Надежность | Использование |
|---------|-------------------|------------|---------------|
| **Read Uncommitted** | 🚀 Высокая | ❌ Низкая | Аналитика, отчеты |
| **Read Committed** | ⚡ Хорошая | ✅ Средняя | Веб-приложения |
| **Repeatable Read** | 🐢 Средняя | 👍 Хорошая | Финансовые системы |
| **Serializable** | 🐌 Низкая | 🔒 Высокая | Банки, аукционы |

### В реальных системах:
```sql
-- Чаще всего используется:
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Для критичных операций:
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Для максимальной надежности:
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

---

## 🎯 КЛЮЧЕВЫЕ ВЫВОДЫ

### Atomicity:
- **"All or Nothing"**
- COMMIT = все изменения применяются
- ROLLBACK = все изменения отменяются

### Consistency:  
- **"Always valid state"**
- Ограничения всегда соблюдаются
- Бизнес-правила выполняются

### Isolation:
- **"Parallel but separate"**
- Уровни от READ UNCOMMITTED до SERIALIZABLE
- Баланс между производительностью и надежностью

### Durability:
- **"Once committed, forever stored"**
- WAL, репликация, бэкапы
- Переживает сбои системы

### Для собеседования:
**"ACID - это четыре свойства, которые гарантируют надежность транзакций: Atomicity обеспечивает 'все или ничего', Consistency сохраняет целостность данных, Isolation защищает от вмешательства параллельных транзакций, а Durability гарантирует сохранность данных после сбоев."**