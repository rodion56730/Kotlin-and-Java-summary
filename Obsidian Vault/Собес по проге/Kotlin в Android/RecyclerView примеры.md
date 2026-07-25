# RecyclerView: примеры и ловушки на собеседовании

## Архитектура

RecyclerView — компонент для отображения больших списков с переиспользованием View через механизм ViewHolder. Основные компоненты:

- **RecyclerView.Adapter** — поставщик данных и ViewHolder'ов
- **ViewHolder** — обёртка над View одного элемента списка, хранит ссылки на его дочерние View
- **LayoutManager** — управляет расположением элементов (`LinearLayoutManager`, `GridLayoutManager`, `StaggeredGridLayoutManager`)
- **ItemDecoration** — декорации: отступы, разделители (`DividerItemDecoration`)
- **ItemAnimator** — анимации при добавлении/удалении/изменении элементов

---

## Ключевые методы Adapter

```kotlin
override fun getItemCount(): Int       // количество элементов
override fun onCreateViewHolder(...)   // создание нового ViewHolder (inflate)
override fun onBindViewHolder(...)     // привязка данных к ViewHolder
```

**Ловушка:** тяжёлая работа в `onBindViewHolder` — inflate, создание объектов, сетевые запросы. Этот метод вызывается очень часто при скролле. Inflate делать только в `onCreateViewHolder`, в `onBindViewHolder` — только привязка данных.

---

## DiffUtil и ListAdapter

Наивное обновление через `notifyDataSetChanged()` перерисовывает весь список целиком — анимации теряются, производительность падает. Правильный подход — `DiffUtil`, который вычисляет минимальный набор изменений между старым и новым списком:

```kotlin
class MyDiffCallback : DiffUtil.ItemCallback<Item>() {
    override fun areItemsTheSame(old: Item, new: Item) = old.id == new.id
    override fun areContentsTheSame(old: Item, new: Item) = old == new
}
```

`ListAdapter` — адаптер со встроенным DiffUtil, обновляется через `submitList(list)`. Автоматически запускает diff в фоновом потоке.

**Ловушка:** передача одного и того же списка в `submitList`. DiffUtil сравнивает ссылки — если передать тот же объект с изменёнными данными, разницы не найдёт и список не обновится. Всегда передавать новую копию: `submitList(list.toList())`.

---

## getItemViewType

Позволяет иметь несколько типов ячеек в одном RecyclerView. ViewHolder'ы создаются и переиспользуются отдельно для каждого типа.

**Ловушка:** возврат произвольных чисел вместо последовательных индексов (0, 1, 2...) — RecyclerView использует viewType как индекс во внутреннем пуле, большие числа приведут к созданию избыточных пулов.

---

## RecycledViewPool

По умолчанию каждый RecyclerView имеет свой пул ViewHolder'ов. Если на экране несколько RecyclerView с одинаковыми типами ячеек (например, горизонтальные списки внутри вертикального), они могут делить пул:

```kotlin
val sharedPool = RecyclerView.RecycledViewPool()
innerRecyclerView.setRecycledViewPool(sharedPool)
```

---

## Частые ловушки на собеседовании

**1. Клики через setOnClickListener в onBindViewHolder** — создаётся новый объект лямбды при каждом bind. Правильно вешать клик в `onCreateViewHolder` и получать данные через `holder.adapterPosition` (или `bindingAdapterPosition`).

**2. `holder.adapterPosition` vs `holder.layoutPosition`** — `adapterPosition` отражает позицию в данных адаптера (актуальна для логики), `layoutPosition` — позицию как её видит LayoutManager (может быть устаревшей во время анимации). Для обработки кликов использовать `bindingAdapterPosition`.

**3. Мигание элементов при обновлении** — происходит когда у элементов нет стабильных id. Решение: переопределить `getItemId()` и выставить `setHasStableIds(true)`.

**4. Вложенный RecyclerView и скролл** — при горизонтальном RecyclerView внутри вертикального по умолчанию внутренний список сбрасывает позицию скролла при переиспользовании ViewHolder. Решение — сохранять и восстанавливать `LayoutManager.onSaveInstanceState()` / `restoreInstanceState()`.

**5. `notifyItemChanged` вызывает мигание** — потому что ItemAnimator запускает анимацию изменения. Решение: передать `payload` вторым аргументом в `notifyItemChanged(pos, payload)` и обработать частичное обновление в `onBindViewHolder(holder, position, payloads)`.

**6. ConcurrentModificationException** — изменение списка данных во время, когда DiffUtil считает diff. Всегда работать с неизменяемыми копиями списка.

---

## Оптимизация

- `setHasFixedSize(true)` — если размер RecyclerView не меняется при обновлении данных, отключает лишние measure/layout проходы
- `setItemViewCacheSize(n)` — увеличить кеш View перед отправкой в RecycledViewPool (по умолчанию 2)
- Избегать вложенных `wrap_content` по направлению скролла — вызывает многократные measure-проходы
- Prefetch — включён по умолчанию в LinearLayoutManager, заранее создаёт ViewHolder'ы для следующих элементов в свободное время между кадрами