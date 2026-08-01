# LazyColumn и LazyRow в Jetpack Compose

## Основная идея

LazyColumn и LazyRow — аналог RecyclerView в Compose. Рендерят только те элементы которые видны на экране, переиспользуя композицию для элементов вне области видимости. LazyColumn — вертикальный список, LazyRow — горизонтальный, LazyVerticalGrid / LazyHorizontalGrid — сетки.

---

## Базовое использование

```kotlin
@Composable
fun UserList(users: List<User>) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // Один элемент
        item {
            Text("Заголовок списка", style = MaterialTheme.typography.headlineMedium)
        }

        // Список элементов
        items(users) { user ->
            UserCard(user)
        }

        // Список с индексом
        itemsIndexed(users) { index, user ->
            UserCard(user, index)
        }

        // Ещё один элемент в конце
        item {
            CircularProgressIndicator() // футер загрузки
        }
    }
}
```

---

## Ключи (key) — важнейшая оптимизация

По умолчанию Compose идентифицирует элементы по позиции. При добавлении/удалении элементов в середину списка — все последующие перекомпозируются. `key` привязывает элемент к стабильному идентификатору:

```kotlin
// ПЛОХО — без key, перекомпозиция всего списка при изменении
items(users) { user ->
    UserCard(user)
}

// ХОРОШО — Compose знает какой элемент какой
items(users, key = { user -> user.id }) { user ->
    UserCard(user)
}

// ХОРОШО — для itemsIndexed
itemsIndexed(
    items = users,
    key = { _, user -> user.id }
) { index, user ->
    UserCard(user)
}
```

**Ловушка:** key должен быть стабильным и уникальным. Нельзя использовать позицию как key — это нивелирует смысл.

---

## LazyListState — управление скроллом

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
    items(users, key = { it.id }) { UserCard(it) }
}

// Кнопка "наверх"
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}

if (showScrollToTop) {
    FloatingActionButton(onClick = {
        coroutineScope.launch {
            listState.animateScrollToItem(0) // с анимацией
            // listState.scrollToItem(0) — без анимации
        }
    }) {
        Icon(Icons.Default.KeyboardArrowUp, contentDescription = null)
    }
}

// Информация о скролле
listState.firstVisibleItemIndex        // индекс первого видимого элемента
listState.firstVisibleItemScrollOffset // смещение в пикселях
listState.isScrollInProgress           // скроллится ли прямо сейчас
```

---

## Пагинация — подгрузка при достижении конца

```kotlin
val listState = rememberLazyListState()

// Определяем когда подгружать следующую страницу
val shouldLoadMore by remember {
    derivedStateOf {
        val lastVisible = listState.layoutInfo.visibleItemsInfo.lastOrNull()
        val totalItems = listState.layoutInfo.totalItemsCount
        lastVisible != null && lastVisible.index >= totalItems - 3 // за 3 элемента до конца
    }
}

LaunchedEffect(shouldLoadMore) {
    if (shouldLoadMore) viewModel.loadNextPage()
}

LazyColumn(state = listState) {
    items(items, key = { it.id }) { ItemCard(it) }

    if (isLoading) {
        item { CircularProgressIndicator(modifier = Modifier.fillMaxWidth().wrapContentWidth()) }
    }
}
```

---

## LazyVerticalGrid

```kotlin
LazyVerticalGrid(
    columns = GridCells.Fixed(2),           // фиксированное количество колонок
    // columns = GridCells.Adaptive(150.dp) // адаптивно — минимум 150dp на колонку
    contentPadding = PaddingValues(8.dp),
    horizontalArrangement = Arrangement.spacedBy(8.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    // Элемент на всю ширину (заголовок)
    item(span = { GridItemSpan(maxLineSpan) }) {
        Text("Заголовок")
    }

    items(photos, key = { it.id }) { photo ->
        PhotoCard(photo)
    }
}
```

---

## Горизонтальный список внутри вертикального

Часто нужен горизонтальный список карточек внутри вертикального списка секций:

```kotlin
LazyColumn {
    items(sections) { section ->
        Text(section.title)

        // Горизонтальный список внутри вертикального
        LazyRow(
            horizontalArrangement = Arrangement.spacedBy(8.dp),
            contentPadding = PaddingValues(horizontal = 16.dp)
        ) {
            items(section.items, key = { it.id }) { item ->
                ItemCard(item)
            }
        }
    }
}
```

**Ловушка:** у каждого LazyRow свой независимый `LazyListState`. Если нужно сохранять позицию скролла каждой строки — хранить состояние в ViewModel и восстанавливать через `rememberLazyListState(initialFirstVisibleItemIndex = ...)`.

---

## StickyHeaders — закреплённые заголовки

```kotlin
val grouped = users.groupBy { it.department }

LazyColumn {
    grouped.forEach { (department, users) ->
        stickyHeader {
            Text(
                text = department,
                modifier = Modifier
                    .fillMaxWidth()
                    .background(MaterialTheme.colorScheme.surface)
                    .padding(horizontal = 16.dp, vertical = 8.dp)
            )
        }

        items(users, key = { it.id }) { user ->
            UserCard(user)
        }
    }
}
```

---

## contentPadding vs padding

```kotlin
// ПЛОХО — padding обрезает контент при скролле
LazyColumn(modifier = Modifier.padding(16.dp)) { }

// ХОРОШО — contentPadding добавляет отступы не обрезая скроллируемый контент
LazyColumn(contentPadding = PaddingValues(16.dp)) { }
```

---

## Частые ловушки

**1. fillMaxHeight внутри LazyColumn** — LazyColumn сам бесконечен по высоте, дочерний `fillMaxHeight()` вызовет крэш. Использовать `fillParentMaxHeight()`.

**2. Вложенные LazyColumn** — два вертикальных ленивых списка не работают вместе напрямую. Решение: объединить контент через разные типы `item { }`.

**3. Без key — анимации вставки/удаления не работают** — Compose не знает какой элемент добавился, а какой удалился.

**4. Тяжёлые вычисления в items { }** — блок выполняется при каждой рекомпозиции видимого элемента. Тяжёлую логику выносить во ViewModel.

**5. Изменяемый список без снапшота** — передача `MutableList` напрямую: Compose не узнает об изменениях. Всегда передавать неизменяемую копию или использовать `SnapshotStateList` (`mutableStateListOf`).

```kotlin
// ПЛОХО
val items = mutableListOf<Item>()
items.add(newItem) // Compose не видит изменение

// ХОРОШО
val items = mutableStateListOf<Item>()
items.add(newItem) // Compose видит и перекомпозирует
```