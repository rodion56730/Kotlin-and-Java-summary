# Handler и Looper

**Looper** — механизм, превращающий обычный поток в поток с очередью сообщений (Message Queue). По умолчанию только главный поток (Main Thread) имеет Looper — он создаётся автоматически при старте приложения. Для фонового потока Looper нужно создавать вручную через `Looper.prepare()` и запускать через `Looper.loop()`.

**Handler** — инструмент для отправки и обработки сообщений (`Message`) и задач (`Runnable`) в очередь конкретного Looper'а. Handler привязывается к Looper'у потока, в котором создан. Главная задача — безопасно переключаться между потоками: выполнить задачу в фоне и вернуть результат в главный поток.

```kotlin
// Отправить задачу в главный поток из любого другого:
val mainHandler = Handler(Looper.getMainLooper())
mainHandler.post {
    textView.text = "Готово" // безопасно обновляем UI
}

// С задержкой:
mainHandler.postDelayed({ doSomething() }, 2000L)
```

**Message** — объект с данными, который Handler кладёт в очередь: имеет `what` (int-код), `arg1`, `arg2`, `obj`. Обрабатывается в `handleMessage(msg: Message)`.

**HandlerThread** — удобная обёртка: поток со встроенным Looper, готовый к использованию:

```kotlin
val handlerThread = HandlerThread("WorkerThread").also { it.start() }
val workerHandler = Handler(handlerThread.looper)
workerHandler.post { /* тяжёлая работа в фоне */ }
```

**Утечки памяти** — частая ловушка: если Handler объявлен как нестатический внутренний класс (или лямбда), он хранит неявную ссылку на внешний класс (Activity). Если в очереди есть отложенные сообщения (`postDelayed`), Activity не уберётся GC. Решение — использовать `WeakReference` на Activity или отменять все сообщения в `onDestroy` через `handler.removeCallbacksAndMessages(null)`.

В современном Android Handler/Looper чаще заменяют Coroutines (`Dispatchers.Main`) или RxJava (`AndroidSchedulers.mainThread()`), но понимание механизма важно, так как он лежит в основе всей системы обработки событий Android.

---
