# Canvas в Jetpack Compose

## Основы

Canvas — composable-функция для кастомной отрисовки. Предоставляет `DrawScope` — контекст с размерами области и методами рисования. Аналог `onDraw(canvas: Canvas)` в классическом View.

```kotlin
Canvas(modifier = Modifier.size(200.dp)) {
    // this = DrawScope
    // size.width, size.height — размеры в пикселях
    // center — центр области

    drawCircle(
        color = Color.Blue,
        radius = size.minDimension / 2,
        center = center
    )
}
```

---

## Основные методы DrawScope

```kotlin
Canvas(modifier = Modifier.fillMaxSize()) {

    // Прямоугольник
    drawRect(
        color = Color.Red,
        topLeft = Offset(50f, 50f),
        size = Size(200f, 100f)
    )

    // Скруглённый прямоугольник
    drawRoundRect(
        color = Color.Green,
        cornerRadius = CornerRadius(16f),
        topLeft = Offset(50f, 200f),
        size = Size(200f, 100f)
    )

    // Круг
    drawCircle(color = Color.Blue, radius = 80f, center = Offset(150f, 150f))

    // Линия
    drawLine(
        color = Color.Black,
        start = Offset(0f, 0f),
        end = Offset(size.width, size.height),
        strokeWidth = 4f,
        cap = StrokeCap.Round
    )

    // Дуга (сектор)
    drawArc(
        color = Color.Yellow,
        startAngle = -90f,  // начало сверху
        sweepAngle = 270f,  // на 270 градусов
        useCenter = true,   // соединять с центром (пирог)
        topLeft = Offset(50f, 50f),
        size = Size(200f, 200f)
    )

    // Путь (произвольная фигура)
    val path = Path().apply {
        moveTo(100f, 0f)
        lineTo(200f, 200f)
        lineTo(0f, 200f)
        close()
    }
    drawPath(path = path, color = Color.Magenta)

    // Текст — через drawContext.canvas и nativeCanvas
    drawContext.canvas.nativeCanvas.drawText(
        "Hello Canvas",
        100f, 100f,
        android.graphics.Paint().apply {
            textSize = 40f
            color = android.graphics.Color.BLACK
        }
    )
}
```

---

## Кисти (Brush) — градиенты

```kotlin
Canvas(modifier = Modifier.size(200.dp)) {

    // Линейный градиент
    drawRect(
        brush = Brush.linearGradient(
            colors = listOf(Color.Blue, Color.Purple, Color.Red),
            start = Offset(0f, 0f),
            end = Offset(size.width, size.height)
        ),
        size = size
    )

    // Радиальный градиент
    drawCircle(
        brush = Brush.radialGradient(
            colors = listOf(Color.White, Color.Blue),
            center = center,
            radius = size.minDimension / 2
        ),
        radius = size.minDimension / 2
    )

    // Sweeping градиент (конический)
    drawCircle(
        brush = Brush.sweepGradient(
            colors = listOf(Color.Red, Color.Yellow, Color.Green, Color.Red),
            center = center
        ),
        radius = size.minDimension / 2
    )
}
```

---

## Трансформации

```kotlin
Canvas(modifier = Modifier.size(200.dp)) {

    // Поворот вокруг точки
    rotate(degrees = 45f, pivot = center) {
        drawRect(color = Color.Blue, size = size)
    }

    // Масштабирование
    scale(scaleX = 0.5f, scaleY = 0.5f, pivot = center) {
        drawCircle(color = Color.Red, radius = size.minDimension / 2)
    }

    // Сдвиг
    translate(left = 50f, top = 50f) {
        drawCircle(color = Color.Green, radius = 50f)
    }

    // withTransform — несколько трансформаций за раз
    withTransform({
        translate(left = 100f, top = 100f)
        rotate(degrees = 30f)
        scale(0.8f)
    }) {
        drawRect(color = Color.Magenta, size = Size(100f, 100f))
    }
}
```

---

## BlendMode и clip

```kotlin
Canvas(modifier = Modifier.size(200.dp)) {

    // Обрезка по форме
    clipRect(left = 50f, top = 50f, right = 150f, bottom = 150f) {
        drawCircle(color = Color.Blue, radius = size.minDimension / 2)
    }

    // Обрезка по пути
    clipPath(Path().apply {
        addRoundRect(RoundRect(Rect(Offset.Zero, size), CornerRadius(20f)))
    }) {
        drawRect(brush = Brush.linearGradient(listOf(Color.Red, Color.Blue)), size = size)
    }
}
```

---

## Практический пример — кастомный прогресс-бар

```kotlin
@Composable
fun CircularProgressBar(
    progress: Float, // 0f..1f
    modifier: Modifier = Modifier,
    strokeWidth: Dp = 12.dp,
    backgroundColor: Color = Color.LightGray,
    progressColor: Color = Color.Blue
) {
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(1000),
        label = "progress"
    )

    Canvas(modifier = modifier.size(100.dp)) {
        val strokeWidthPx = strokeWidth.toPx()
        val radius = (size.minDimension - strokeWidthPx) / 2
        val style = Stroke(width = strokeWidthPx, cap = StrokeCap.Round)

        // Фон
        drawCircle(
            color = backgroundColor,
            radius = radius,
            style = style
        )

        // Прогресс
        drawArc(
            color = progressColor,
            startAngle = -90f,
            sweepAngle = 360f * animatedProgress,
            useCenter = false,
            style = style,
            topLeft = Offset(strokeWidthPx / 2, strokeWidthPx / 2),
            size = Size(radius * 2, radius * 2)
        )
    }
}
```

---

## drawWithContent и Modifier.drawBehind / drawWithCache

```kotlin
// drawBehind — рисуем позади контента
Text(
    text = "Выделенный текст",
    modifier = Modifier.drawBehind {
        drawRoundRect(
            color = Color.Yellow,
            cornerRadius = CornerRadius(8f),
            size = Size(size.width + 16f, size.height + 8f),
            topLeft = Offset(-8f, -4f)
        )
    }
)

// drawWithCache — кешируем объекты для оптимизации (Path, Paint не пересоздаются)
Modifier.drawWithCache {
    val path = Path().apply { /* сложный путь */ }
    onDrawBehind {
        drawPath(path, Color.Blue) // path не пересоздаётся при рекомпозиции
    }
}
```

---

## graphicsLayer — GPU-ускорение

`graphicsLayer` выполняет трансформации на GPU, не вызывая рекомпозицию:

```kotlin
// Трансформации без рекомпозиции (только перерисовка слоя)
val rotation by animateFloatAsState(targetValue = if (isRotated) 180f else 0f, label = "rot")

Box(
    modifier = Modifier.graphicsLayer(
        rotationZ = rotation,
        scaleX = scale,
        scaleY = scale,
        alpha = alpha,
        translationX = offsetX,
        translationY = offsetY,
        shadowElevation = 8f,
        shape = RoundedCornerShape(16.dp),
        clip = true
    )
)
```

---

## Частые ловушки

**1. Единицы измерения — всегда пиксели** — в DrawScope все значения в пикселях, не в dp. Конвертировать через `dp.toPx()` внутри DrawScope.

**2. Пересоздание объектов на каждом кадре** — Path, Paint, Shader — тяжёлые объекты. Создавать через `remember` или `drawWithCache`:

```kotlin
// ПЛОХО — новый Path на каждый кадр анимации
Canvas(...) { val path = Path().apply { /* ... */ }; drawPath(path, Color.Blue) }

// ХОРОШО
val path = remember { Path().apply { /* ... */ } }
Canvas(...) { drawPath(path, Color.Blue) }
```

**3. Canvas не реагирует на клики** — нужно добавить `Modifier.pointerInput` отдельно.

**4. Текст через nativeCanvas** — рисование текста в DrawScope требует обращения к `nativeCanvas` и `android.graphics.Paint`. Проще использовать `Text()` поверх Canvas через `Box { Canvas(...); Text(...) }`.