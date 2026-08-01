# Layouts в Jetpack Compose

## Базовые контейнеры

**Column** — размещает дочерние элементы вертикально. **Row** — горизонтально. **Box** — друг поверх друга (аналог FrameLayout).

```kotlin
Column(
    modifier = Modifier.fillMaxWidth().padding(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp), // отступ между элементами
    horizontalAlignment = Alignment.CenterHorizontally
) {
    Text("Первый")
    Text("Второй")
}

Row(
    horizontalArrangement = Arrangement.SpaceBetween,
    verticalAlignment = Alignment.CenterVertically
) {
    Icon(Icons.Default.Star, contentDescription = null)
    Text("Рейтинг")
    Text("5.0")
}

Box(modifier = Modifier.size(100.dp)) {
    Image(painter, contentDescription = null, modifier = Modifier.fillMaxSize())
    Text("Поверх", modifier = Modifier.align(Alignment.BottomCenter))
}
```

---

## Modifier — ключевая концепция

Modifier — цепочка инструкций для изменения внешнего вида и поведения компонента. **Порядок важен** — каждый модификатор применяется к результату предыдущего.

```kotlin
// РАЗНЫЙ результат:
Modifier.padding(16.dp).background(Color.Red)  // отступ снаружи фона
Modifier.background(Color.Red).padding(16.dp)  // отступ внутри фона (как margin vs padding)

// Размеры
Modifier.fillMaxSize()          // занять всё доступное
Modifier.fillMaxWidth(0.5f)     // занять 50% ширины
Modifier.size(100.dp)           // фиксированный размер
Modifier.wrapContentSize()      // по содержимому
Modifier.weight(1f)             // внутри Row/Column — занять пропорциональную долю

// Ловушка — weight работает только внутри Row/Column:
Row {
    Text("Левый", modifier = Modifier.weight(1f))  // занимает половину
    Text("Правый", modifier = Modifier.weight(1f)) // занимает половину
}
```

**Кастомный Modifier:**

```kotlin
fun Modifier.coloredBorder(color: Color, width: Dp) = this
    .border(width, color, RoundedCornerShape(8.dp))
    .padding(width) // чтобы контент не налезал на рамку
```

---

## Arrangement и Alignment

**Arrangement** — как распределить дочерние элементы вдоль главной оси:

- `Arrangement.Start / End / Center` — к краю или центру
- `Arrangement.SpaceBetween` — равные промежутки между элементами
- `Arrangement.SpaceAround` — промежутки вокруг каждого элемента
- `Arrangement.SpaceEvenly` — равные промежутки включая края
- `Arrangement.spacedBy(8.dp)` — фиксированный отступ между элементами

**Alignment** — выравнивание вдоль поперечной оси:

- В Column: `Alignment.Start`, `CenterHorizontally`, `End`
- В Row: `Alignment.Top`, `CenterVertically`, `Bottom`
- В Box: `Alignment.TopStart`, `Center`, `BottomEnd` и т.д.

---

## ConstraintLayout в Compose

Для сложных макетов где Column/Row недостаточно:

```kotlin
ConstraintLayout(modifier = Modifier.fillMaxSize()) {
    val (image, title, subtitle, button) = createRefs()

    Image(
        painter = painter,
        contentDescription = null,
        modifier = Modifier.constrainAs(image) {
            top.linkTo(parent.top, margin = 16.dp)
            centerHorizontallyTo(parent)
        }
    )

    Text(
        text = "Заголовок",
        modifier = Modifier.constrainAs(title) {
            top.linkTo(image.bottom, margin = 8.dp)
            centerHorizontallyTo(parent)
        }
    )

    Button(
        onClick = {},
        modifier = Modifier.constrainAs(button) {
            bottom.linkTo(parent.bottom, margin = 16.dp)
            centerHorizontallyTo(parent)
        }
    )
}
```

---

## Кастомный Layout

Когда стандартных контейнеров недостаточно — `Layout {}` для полного контроля над измерением и размещением:

```kotlin
@Composable
fun FlowRow(modifier: Modifier = Modifier, content: @Composable () -> Unit) {
    Layout(content = content, modifier = modifier) { measurables, constraints ->
        val placeables = measurables.map { it.measure(constraints) }

        var x = 0
        var y = 0
        var rowHeight = 0

        layout(constraints.maxWidth, constraints.maxHeight) {
            placeables.forEach { placeable ->
                if (x + placeable.width > constraints.maxWidth) {
                    x = 0
                    y += rowHeight
                    rowHeight = 0
                }
                placeable.placeRelative(x, y)
                x += placeable.width
                rowHeight = maxOf(rowHeight, placeable.height)
            }
        }
    }
}
```

---

## Intrinsic Measurements

Иногда нужно знать размер дочернего элемента до его финального измерения. `IntrinsicSize.Min/Max` позволяет это сделать:

```kotlin
// Сделать все дочерние элементы одной высоты
Row(modifier = Modifier.height(IntrinsicSize.Min)) {
    Text("Короткий")
    Divider(modifier = Modifier.fillMaxHeight().width(1.dp))
    Text("Длинный\nтекст\nна три строки")
}
```

**Ловушка:** `IntrinsicSize` вызывает двойной проход измерения — может быть дорого для больших деревьев.

---

## Spacer

Явный отступ между элементами:

```kotlin
Column {
    Text("Первый")
    Spacer(modifier = Modifier.height(16.dp))
    Text("Второй")
    Spacer(modifier = Modifier.weight(1f)) // заполнить всё доступное пространство
    Button(onClick = {}) { Text("Кнопка внизу") }
}
```

---

## Частые ловушки

**1. fillMaxSize внутри LazyColumn** — вызовет бесконечные ограничения и крэш. Использовать `fillParentMaxSize()` внутри lazy-контейнеров.

**2. Вложенные Column с одинаковыми scroll** — два вертикальных скроллируемых контейнера не работают вместе. Использовать `LazyColumn` с вложенными items.

**3. Порядок padding и clickable:**

```kotlin
// Область клика не включает padding
Modifier.padding(16.dp).clickable { }

// Область клика включает padding — правильно
Modifier.clickable { }.padding(16.dp)
```

**4. Нет `match_parent` и `wrap_content`** — в Compose используются `fillMaxSize()` и `wrapContentSize()`. По умолчанию компонент занимает минимально необходимое место.