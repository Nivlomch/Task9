---
title: "Практическая работа #9: Асинхронное программирование в Dart"
discipline: "Разработка мобильных приложений"
tags:
  [
    dart,
    async,
    future,
    stream,
    stream-controller,
    isolates,
    event-loop,
    flutter,
  ]
---

# Практическая работа: Асинхронное программирование в Dart (Event Loop, Future, Stream, Isolates)

> [!NOTE]
> **Цель работы:** Глубоко освоить неблокирующую модель выполнения программ в Dart. Изучить архитектуру цикла событий (Event Loop), очереди микротасок и событий, работу с отложенными результатами (`Future`, `async/await`), реактивную обработку непрерывных потоков данных (`Stream`, `StreamController`, `async*`), а также параллельные вычисления в изолированных потоках памяти (`Isolates`).

---

## 📋 Содержание

1. [Модель параллелизма Dart: однопоточность и Event Loop](#1-модель-параллелизма-dart-однопоточность-и-event-loop)
2. [Тема 1. Отложенные вычисления: Future API и синтаксис async/await](#тема-1-отложенные-вычисления-future-api-и-синтаксис-asyncawait)
3. [Тема 2. Непрерывные потоки: Stream API и генераторы async\*](#тема-2-непрерывные-потоки-stream-api-и-генераторы-async)
4. [Тема 3. Реактивное управление: StreamController и StreamTransformer](#тема-3-реактивное-управление-streamcontroller-и-streamtransformer)
5. [Тема 4. Истинная многопоточность: Изоляты (Isolates) и Isolate.run](#тема-4-истинная-многопоточность-изоляты-isolates-и-isolaterun)
6. [50 Смешанных заданий по асинхронной разработке (Mobile Async Focus)](#50-смешанных-заданий-по-асинхронной-разработке)

---

## 1. Модель параллелизма Dart: однопоточность и Event Loop

По умолчанию код Dart выполняется в **одном потоке** (Main Isolate). Это исключает гонки данных (Race Conditions) и необходимость в блокирующих мьютексах, но требует неблокирующего подхода при операциях ввода-вывода (сеть, файлы, таймеры).

### Архитектура цикла событий (Event Loop)

```
                            СТАРТ ПРОГРАММЫ (main())
                                       │
                                       ▼
                     ┌───────────────────────────────────┐
                     │ Очередь микрозадач (Microtask)    │ ◄── Высший приоритет
                     │ (Внутренние системные колбэки)    │
                     └─────────────────┬─────────────────┘
                                       │ Пуста?
                                       ▼ Да
                     ┌───────────────────────────────────┐
                     │ Очередь событий (Event Queue)     │ ◄── Стандартный приоритет
                     │ (I/O, Таймеры, Тапы по экрану)    │
                     └─────────────────┬─────────────────┘
                                       │
                                       ▼
                             Исполнение задачи
                                       │
                                       └──────► Цикл повторяется
```

### Приоритет очередей

1. **Синхронный код:** Выполняется немедленно от первой до последней строчки.
2. **Microtask Queue:** Высокоприоритетные короткие системные задачи (`scheduleMicrotask()`). Очередь событий не начнет обрабатываться, пока очередь микрозадач не опустеет.
3. **Event Queue:** События пользовательского ввода (тапы, жесты), срабатывание таймеров (`Timer`), ответы сетевых сокетов и завершение чтения файлов.

---

## Тема 1. Отложенные вычисления: Future API и синтаксис async/await

`Future<T>` представляет собой контейнер для значения типа `T`, которое станет доступно через некоторое время, либо завершится ошибкой.

### Состояния Future

- **Uncompleted (В процессе):** Асинхронная операция еще выполняется.
- **Completed with Data (Успех):** Операция завершилась, вернув результат.
- **Completed with Error (Ошибка):** Во время выполнения выброшено исключение.

### Пример кода (Future, цепочки и параллельное ожидание)

```dart
// Имитация сетевого запроса к каталогу
Future<List<String>> fetchProductCatalog() async {
  await Future.delayed(const Duration(seconds: 2)); // Неблокирующая задержка

  bool hasNetworkError = false;
  if (hasNetworkError) {
    throw Exception("Ошибка 503: Сервер каталога временно недоступен");
  }

  return ["Смартфон Galaxy S26", "Планшет Tab Ultra", "Наушники Buds Pro"];
}

// Имитация загрузки профиля пользователя
Future<String> fetchUserProfile() async {
  await Future.delayed(const Duration(seconds: 1));
  return "Алексей Смирнов";
}

void main() async {
  print("1. Инициализация экрана...");

  try {
    // 1. Последовательное ожидание через async/await
    String user = await fetchUserProfile();
    print("2. Пользователь загружен: $user");

    // 2. Параллельный запуск нескольких Future через Future.wait()
    print("3. Старт параллельной загрузки дополнительных данных...");
    var results = await Future.wait([
      fetchProductCatalog(),
      Future.delayed(const Duration(milliseconds: 500), () => "Курсы валют OK"),
    ]);

    List<String> products = results[0] as List<String>;
    print("4. Товары получены: $products");

  } catch (error) {
    print("Перехвачена ошибка: $error");
  } finally {
    print("5. Завершение работы, скрытие ProgressBar");
  }

  // 3. Таймаут выполнения операции
  try {
    await fetchProductCatalog().timeout(
      const Duration(milliseconds: 500),
      onTimeout: () => throw Exception("Таймаут соединения! Сервер не ответил вовремя."),
    );
  } catch (e) {
    print("Контроль таймаута: $e");
  }
}
```

### Задания для закрепления

1. **Симулятор авторизации:** Напишите функцию `Future<bool> login(String user, String pass)`, которая через 1.5 секунды возвращает `true`, если пароль равен `"1234"`, и выбрасывает `AuthException` в противном случае.
2. **Параллельное чтение файлов конфигурации:** Запустите параллельно три вызова `Future` с разным временем задержки с помощью `Future.wait()` и объедините их результаты в одну карту.
3. **Гонка асинхронных задач (Future.any):** Запустите запрос к двум зеркалам одного сервера (задержка 800 мс и 1200 мс) и получите ответ от самого быстрого с помощью `Future.any()`.
4. **Повторные попытки с задержкой (Retry Pattern):** Напишите функцию `retry(Future Function() task, int maxAttempts)`, которая при падении задачи повторяет вызов до 3 раз с паузой в 1 секунду.
5. **Цепочка промисов через then/catchError:** Реализуйте аналогичную логику авторизации без использования ключевых слов `async/await`, используя только методы `.then()`, `.catchError()` и `.whenComplete()`.

---

## Решение Тема 1\*
```dart
// Тема 1. Отложенные вычисления: Future API и async/await

// Задание 1
class AuthException implements Exception {}

Future<bool> login(String user, String pass) async {
  await Future.delayed(const Duration(milliseconds: 1500));
  if (pass == "1234") return true;
  throw AuthException();
}

// Задание 2
Future<Map<String, String>> readConfigs() async {
  var results = await Future.wait([
    Future.delayed(const Duration(seconds: 1), () => "config1"),
    Future.delayed(const Duration(seconds: 2), () => "config2"),
    Future.delayed(const Duration(seconds: 3), () => "config3"),
  ]);

  return {
    "file1": results[0],
    "file2": results[1],
    "file3": results[2],
  };
}

// Задание 3
Future<String> fastestServer() {
  return Future.any([
    Future.delayed(const Duration(milliseconds: 800), () => "Сервер 1"),
    Future.delayed(const Duration(milliseconds: 1200), () => "Сервер 2"),
  ]);
}

// Задание 4
Future retry(Future Function() task, int maxAttempts) async {
  for (int i = 0; i < maxAttempts && i < 3; i++) {
    try {
      return await task();
    } catch (e) {
      if (i == maxAttempts - 1 || i == 2) rethrow;
      await Future.delayed(const Duration(seconds: 1));
    }
  }
}

// Задание 5
Future<bool> loginThen(String user, String pass) {
  return Future.delayed(const Duration(milliseconds: 1500))
      .then((_) {
        if (pass == "1234") return true;
        throw AuthException();
      })
      .catchError((e) {
        print(e);
        return false;
      })
      .whenComplete(() {
        print("Авторизация завершена");
      });
}
```
---

## Тема 2. Непрерывные потоки: Stream API и генераторы async\*

Если `Future` возвращает **однократное** значение, то `Stream<T>` поставляет **последовательность** асинхронных событий во времени (клики, показания датчиков GPS, сообщения сокета).

### Два типа потоков

1. **Single-Subscription Stream (Одиночный поток):** Может иметь только одного слушателя за все время жизни (например, чтение файла по кускам). Повторный вызов `.listen()` приведет к исключению `StateError`.
2. **Broadcast Stream (Широковещательный поток):** Позволяет подключать множество независимых подписчиков одновременно (события UI, шина аналитики). Создается через `stream.asBroadcastStream()`.

### Пример кода (Генератор async\* и обработка потока)

```dart
// Асинхронный генератор: возвращает непрерывный поток данных
Stream<int> countdownTimer(int fromSeconds) async* {
  for (int i = fromSeconds; i >= 0; i--) {
    await Future.delayed(const Duration(seconds: 1));
    yield i; // Отправка значения в поток
  }
}

void main() async {
  print("Старт таймера:");

  Stream<int> timerStream = countdownTimer(5);

  // Функциональные операторы над потоком (подобно коллекциям)
  StreamSubscription<String> subscription = timerStream
      .where((seconds) => seconds % 2 == 0) // Фильтруем только четные секунды
      .map((seconds) => "Осталось четных секунд: $seconds")
      .listen(
        (data) => print("[DATA] $data"),
        onError: (err) => print("[ERROR] $err"),
        onDone: () => print("[DONE] Отсчет завершен!"),
        cancelOnError: false,
      );

  // Отмена подписки через 3.5 секунды для предотвращения утечки ресурсов
  await Future.delayed(const Duration(milliseconds: 3500));
  print("Принудительная отмена подписки слушателем!");
  await subscription.cancel();
}
```

### Задания для закрепления

1. **Поток тиков секундомера:** Создайте функцию `Stream<int> stopWatch() async*`, непрерывно генерирующую инкрементный счетчик каждую секунду.
2. **Поток GPS-координат (Имитация):** Сгенерируйте поток кортежей `(double lat, double lon)`, слегка изменяющих координаты каждые 500 мс.
3. **Широковещательный стрим котировок:** Создайте однократный поток цен, преобразуйте его в Broadcast через `.asBroadcastStream()` и подключите двух разных слушателей: UI-виджет и логгер аналитики.
4. **Агрегация потока (fold/reduce):** Посчитайте сумму чисел, поступающих из потока из 10 элементов, с помощью метода `stream.fold()`.
5. **Делегирование генератора через yield\*:** Напишите функцию, которая объединяет два потока последовательно с помощью синтаксиса `yield* subStream`.

---

## Решение Тема 2\*
```dart
// Тема 2. Непрерывные потоки: Stream API и генераторы async*

// Задание 1
Stream<int> stopWatch() async* {
  int i = 0;
  while (true) {
    await Future.delayed(const Duration(seconds: 1));
    yield i++;
  }
}

// Задание 2
Stream<(double, double)> gps() async* {
  double lat = 55.75, lon = 37.61;
  while (true) {
    await Future.delayed(const Duration(milliseconds: 500));
    lat += 0.001;
    lon += 0.001;
    yield (lat, lon);
  }
}

// Задание 3
void broadcastPrices() {
  Stream<int> prices = Stream.fromIterable([100, 200, 300, 400])
      .asBroadcastStream();

  prices.listen((price) => print("UI: $price"));
  prices.listen((price) => print("Лог: $price"));
}

// Задание 4
Future<int> sumStream() {
  return Stream.fromIterable([1, 2, 3, 4, 5, 6, 7, 8, 9 ,10])
      .fold(0, (sum, number) => sum + number);
}

// Задание 5
Stream<int> combineStreams(Stream<int> stream1, Stream<int> stream2) async* {
  yield* stream1;
  yield* stream2;
}
```

---

## Тема 3. Реактивное управление: StreamController и StreamTransformer

`StreamController<T>` — это программный пульт управления потоком, разделенный на две части:

- **`Sink` (Входной порт):** Место, куда события отправляются (`sink.add()`, `sink.addError()`).
- **`Stream` (Выходной порт):** Поток, на который подписываются клиенты для чтения данных.

```
                           STREAM CONTROLLER
        ┌─────────────────────────────────────────────────────┐
        │                                                     │
Events ─┼──► [ Sink (add / addError) ]                        │
        │                  │                                  │
        │                  ▼                                  │
        │         Внутренний буфер                            │
        │                  │                                  │
        │                  ▼                                  │
        │         [ Stream (listen) ] ──► Subscribers (UI)    │
        │                                                     │
        └─────────────────────────────────────────────────────┘
```

### Пример кода (Простейший реактивный BLoC счетчика без сторонних библиотек)

```dart
import 'dart:async';

class CounterBloc {
  int _counter = 0;

  // Контроллер с поддержкой нескольких слушателей (Broadcast)
  final _stateController = StreamController<int>.broadcast();

  // Публичный поток только для чтения UI-компонентами
  Stream<int> get counterStream => _stateController.stream;

  void increment() {
    _counter++;
    _stateController.sink.add(_counter); // Отправка нового стейта
  }

  void reset() {
    _counter = 0;
    _stateController.sink.add(_counter);
  }

  // Обязательное закрытие контроллера при уничтожении экрана
  void dispose() {
    _stateController.close();
  }
}

void main() async {
  final bloc = CounterBloc();

  final sub1 = bloc.counterStream.listen((val) => print("UI Экран получил: $val"));
  final sub2 = bloc.counterStream.listen((val) => print("Аналитика зафиксировала: $val"));

  bloc.increment(); // 1
  bloc.increment(); // 2
  bloc.reset();     // 0

  await Future.delayed(const Duration(milliseconds: 100));
  await sub1.cancel();
  await sub2.cancel();
  bloc.dispose();
}
```

### Задания для закрепления

1. **Кастомный StreamTransformer:** Реализуйте трансформер строк через `StreamTransformer.fromHandlers()`, переводящий все входящие сообщения в верхний регистр.
2. **Контроллер поискового ввода:** Реализуйте класс `SearchControllerWrapper`, принимающий посимвольный ввод пользователя в Sink и отдающий очищенную строку в Stream.
3. **Обработка ошибок в контроллере:** Отправьте ошибку в контроллер с помощью `sink.addError(Exception("Сбой датчика"))` и перехватите её в секции `onError` слушателя.
4. **Контроль состояния закрытия:** Напишите метод, проверяющий свойство `controller.isClosed` перед попыткой добавить новое событие, предотвращая падение с ошибкой.
5. **Счетчик активных подписчиков:** Сконфигурируйте `StreamController(onListen: ..., onCancel: ...)` для отслеживания момента появления и ухода подписчиков.



---

## Решение Тема 3\*
```dart
// Тема 3. Реактивное управление: StreamController и StreamTransformer

import 'dart:async';

// Задание 1
final upperTransformer = StreamTransformer<String, String>.fromHandlers(
  handleData: (data, sink) => sink.add(data.toUpperCase()),
);

// Задание 2
class SearchControllerWrapper {
  final _controller = StreamController<String>();

  Stream<String> get stream => _controller.stream;

  Sink<String> get sink => _controller.sink;

  void dispose() {
    _controller.close();
  }
}

// Задание 3
void errorController() {
  final controller = StreamController<String>();

  controller.stream.listen(
    (data) => print(data),
    onError: (error) => print("Ошибка: $error"),
  );

  controller.sink.addError(Exception("Сбой датчика"));
  controller.close();
}

// Задание 4
void safeAdd(StreamController<int> controller, int value) {
  if (!controller.isClosed) {
    controller.sink.add(value);
  }
}

// Задание 5
int activeSubscribers = 0;

final controller = StreamController<int>.broadcast(
  onListen: () => activeSubscribers++,
  onCancel: () => activeSubscribers--,
);
```
---

## Тема 4. Истинная многопоточность: Изоляты (Isolates) и Isolate.run

Если задача требует непрерывных тяжелых математических расчетов (например, парсинг 50-мегабайтного JSON, фильтрация графических пикселей, шифрование больших баз данных), она заблокирует Event Loop главного потока, вызвав зависание мобильного экрана (Jank).

Для тяжелых фоновых вычислений Dart использует **Изоляты** (Isolates).

### Отличие изолятов от традиционных потоков Java/C++

- Изоляты **НЕ имеют общей разделяемой памяти** (Shared Memory отсутствует).
- Нет блокировок, взаимных дедлоков и мьютексов.
- Общение между изолятами происходит исключительно через передачу сообщений по системным портам (`SendPort` и `ReceivePort`), либо через метод `Isolate.run()` (начиная с Dart 2.19+).

```
   MAIN ISOLATE (UI & Event Loop)             BACKGROUND WORKER ISOLATE
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│ Своя изолированная память       │       │ Своя изолированная память       │
│                                 │       │                                 │
│ [SendPort] ─── Message (Copy) ──┼──────►│ [ReceivePort]                   │
│                                 │       │                                 │
│                                 │       │ Тяжелый расчет:                 │
│                                 │       │ парсинг JSON / сжатие фото      │
│                                 │       │                                 │
│ [ReceivePort] ◄── Result ───────┼───────┤ [SendPort]                      │
└─────────────────────────────────┘       └─────────────────────────────────┘
```

### Пример кода: Isolate.run() против низкоуровневых портов

```dart
import 'dart:isolate';

// Тяжелая функция верхнего уровня (Top-level или static)
int heavyCalculation(int iterations) {
  int sum = 0;
  for (int i = 0; i < iterations; i++) {
    sum += (i % 3);
  }
  return sum;
}

void main() async {
  print("1. Главный поток свободен, UI отрисовывается со скоростью 120 FPS");

  // Способ 1: Современный компактный Isolate.run() (Dart 2.19 / 3+)
  print("2. Запуск тяжелого расчета в параллельном изоляте...");
  int result = await Isolate.run(() => heavyCalculation(500000000));
  print("3. Результат из Isolate.run: $result");

  // Способ 2: Низкоуровневые порты для длительного двустороннего обмена
  final receivePort = ReceivePort();
  await Isolate.spawn(_workerEntryPoint, receivePort.sendPort);

  // Ожидаем ответ от запущенного воркера
  final workerMessage = await receivePort.first;
  print("4. Сообщение от постоянного воркера: $workerMessage");
}

void _workerEntryPoint(SendPort sendPort) {
  // Выполнение фоновой задачи
  int calculatedValue = heavyCalculation(100000);
  sendPort.send("Готово! Вычислено: $calculatedValue");
}
```

### Задания для закрепления

1. **Параллельный парсер JSON:** Напишите функцию, принимающую строку из 10 000 фиктивных записей и десериализующую ее в объекты через `Isolate.run()`.
2. **Поиск простых чисел в фоне:** Запустите вычисление всех простых чисел до 1 000 000 в фоновом изоляте, параллельно выводя в консоль главного потока точки каждую секунду.
3. **Двусторонний обмен через порты:** Создайте изолят-калькулятор, который слушает команды по `SendPort`, выполняет операции сложения и отправляет ответ обратно.
4. **Замер падения FPS:** Напишите скрипт, запускающий цикл на 2 млрд итераций сначала синхронно в `main()`, а затем в `Isolate.run()`, сравнив отзывчивость таймеров.
5. **Принудительное уничтожение изолята:** Запустите бесконечный фоновый изолят через `Isolate.spawn` и принудительно завершите его работу методом `isolate.kill(priority: Isolate.immediate)`.

---

## Решение Тема 4\*
```dart
// Тема 4. Истинная многопоточность: Изоляты (Isolates) и Isolate.run

import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

// Задание 1
class Product {
  final int id;
  final String name;

  Product(this.id, this.name);

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(json["id"], json["name"]);
  }
}

Future<List<Product>> parseJson(String data) {
  return Isolate.run(() {
    return (jsonDecode(data) as List)
        .map((e) => Product.fromJson(e))
        .toList();
  });
}

Future<void> task1() async {
  String data = jsonEncode(
    List.generate(10000, (i) => {"id": i, "name": "Product $i"}),
  );

  List<Product> products = await parseJson(data);
  print(products.length);
}

// Задание 2
List<int> findPrimes(int limit) {
  List<int> primes = [];

  for (int i = 2; i <= limit; i++) {
    bool prime = true;
    for (int j = 2; j * j <= i; j++) {
      if (i % j == 0) {
        prime = false;
        break;
      }
    }
    if (prime) primes.add(i);
  }

  return primes;
}

Future<void> task2() async {
  Timer timer = Timer.periodic(
    const Duration(seconds: 1),
    (_) => print("."),
  );

  List<int> primes = await Isolate.run(() => findPrimes(1000000));

  timer.cancel();
  print("Найдено: ${primes.length}");
}

// Задание 3
void calculatorWorker(SendPort port) {
  ReceivePort receivePort = ReceivePort();
  port.send(receivePort.sendPort);

  receivePort.listen((message) {
    List data = message;
    SendPort reply = data[2];
    reply.send(data[0] + data[1]);
  });
}

Future<void> task3() async {
  ReceivePort receivePort = ReceivePort();
  await Isolate.spawn(calculatorWorker, receivePort.sendPort);

  SendPort worker = await receivePort.first as SendPort;

  ReceivePort reply = ReceivePort();
  worker.send([5, 10, reply.sendPort]);

  print(await reply.first);

  receivePort.close();
  reply.close();
}

// Задание 4
int heavyCalculation(int iterations) {
  int sum = 0;
  for (int i = 0; i < iterations; i++) {
    sum += i % 3;
  }
  return sum;
}

Future<void> task4() async {
  Timer timer = Timer.periodic(
    const Duration(seconds: 1),
    (_) => print("Таймер работает"),
  );

  heavyCalculation(2000000000);
  await Future.delayed(const Duration(seconds: 1));

  print("Синхронно завершено");

  await Isolate.run(() => heavyCalculation(2000000000));

  print("В изоляте завершено");
  timer.cancel();
}

// Задание 5
void infiniteWorker(SendPort port) {
  while (true) {
    // Бесконечная работа
  }
}

Future<void> task5() async {
  ReceivePort receivePort = ReceivePort();
  Isolate isolate = await Isolate.spawn(
    infiniteWorker,
    receivePort.sendPort,
  );

  await Future.delayed(const Duration(milliseconds: 100));
  isolate.kill(priority: Isolate.immediate);
  receivePort.close();
}
```
---

## 50 Смешанных заданий по асинхронной разработке

Задания охватывают реальные сценарии клиентской мобильной разработки (REST API, WebSocket, фоновые очереди, кэширование, датчики, архитектура MVI/BLoC).

### Блок 1: Сетевые запросы, таймеры и Future

1. **Имитатор HTTP GET запроса:** Напишите функцию `fetchData(String url)`, возвращающую JSON через 1200 мс с обработкой статусов 200, 404 и 500.
2. **Экспоненциальная задержка (Exponential Backoff):** Реализуйте повтор сетевого запроса с прогрессирующей паузой: 1с, 2с, 4с, 8с при возникновении `SocketException`.
3. **Параллельная загрузка медиафайлов:** Скачайте метаданные для 5 картинок одновременно через `Future.wait()`, ограничив параллелизм до 2 одновременных вызовов.
4. **Валидатор доступности сервера (Ping):** Отправьте пинг-запросы к трем зеркалам и верните URL первого ответившего сервера (`Future.any`).
5. **Прерываемый запрос по таймауту:** Оберните вызов длительного сохранения профиля в таймаут 3 секунды с аварийным прерыванием операции.
6. **Очередь последовательного выполнения (Async Queue):** Спроектируйте класс, выполняющий поступающие асинхронные задачи строго по очереди друг за другом (FIFO).
7. **Асинхронный замер времени выполнения:** Напишите утилиту `benchmark(Future Function() task)`, замеряющую точное время выполнения асинхронного блока в миллисекундах (`Stopwatch`).
8. **Ленивый асинхронный синглтон:** Реализуйте инициализацию конфигурации приложения `AppConfig.getInstance()`, выполняющую чтение данных из асинхронного источника ровно один раз.
9. **Параллельное слияние локального и сетевого кэша:** Запустите чтение локальной базы SQLite и сетевой вызов одновременно, вернув локальные данные, если сеть не успела ответить за 300 мс.
10. **Асинхронный дебаунсер (Debounce):** Напишите функцию, откладывающую выполнение сетевого поиска на 400 мс и сбрасывающую таймер при каждом новом нажатии клавиши.

## Решение Блок 1\*
```dart
// Блок 1. Сетевые запросы, таймеры и Future

import 'dart:async';
import 'dart:io';

// Задание 1
Future<String> fetchData(String url) async {
  await Future.delayed(const Duration(milliseconds: 1200));

  int status = 200;

  if (url.contains("404")) status = 404;
  if (url.contains("500")) status = 500;

  switch (status) {
    case 200:
      return '{"status":200,"data":"OK"}';
    case 404:
      throw Exception("404 Not Found");
    case 500:
      throw Exception("500 Server Error");
    default:
      throw Exception("Неизвестная ошибка");
  }
}

// Задание 2
Future retryRequest(Future Function() request) async {
  for (int i = 0; i < 4; i++) {
    try {
      return await request();
    } on SocketException {
      await Future.delayed(Duration(seconds: 1 << i));
    }
  }
  throw SocketException("Все попытки завершились ошибкой");
}

// Задание 3
Future<List<String>> loadImages() async {
  List<String> urls = List.generate(5, (i) => "image$i");
  List<String> results = [];
  int index = 0;

  Future<void> worker() async {
    while (index < urls.length) {
      String url = urls[index++];
      await Future.delayed(const Duration(seconds: 1));
      results.add("Metadata: $url");
    }
  }

  await Future.wait([worker(), worker()]);
  return results;
}

// Задание 4
Future<String> ping() {
  return Future.any([
    Future.delayed(const Duration(milliseconds: 800), () => "mirror1"),
    Future.delayed(const Duration(milliseconds: 1200), () => "mirror2"),
    Future.delayed(const Duration(milliseconds: 1500), () => "mirror3"),
  ]);
}

// Задание 5
Future<void> saveProfile() async {
  await Future.delayed(const Duration(seconds: 5));
}

Future<void> saveWithTimeout() async {
  await saveProfile().timeout(
    const Duration(seconds: 3),
    onTimeout: () => throw TimeoutException("Сохранение прервано"),
  );
}

// Задание 6
class AsyncQueue {
  Future<void> _queue = Future.value();

  void add(Future Function() task) {
    _queue = _queue.then((_) => task()).catchError((e) {});
  }
}

// Задание 7
Future<int> benchmark(Future Function() task) async {
  Stopwatch stopwatch = Stopwatch()..start();
  await task();
  stopwatch.stop();
  return stopwatch.elapsedMilliseconds;
}

// Задание 8
class AppConfig {
  static Future<AppConfig>? _instance;

  AppConfig();

  static Future<AppConfig> getInstance() {
    return _instance ??= _load();
  }

  static Future<AppConfig> _load() async {
    await Future.delayed(const Duration(seconds: 1));
    return AppConfig();
  }
}

// Задание 9
Future<String> getCache() async {
  Future<String> local = Future.delayed(
    const Duration(milliseconds: 100),
    () => "Локальные данные",
  );

  Future<String> network = Future.delayed(
    const Duration(seconds: 1),
    () => "Сетевые данные",
  );

  return Future.any([
    local,
    network.timeout(
      const Duration(milliseconds: 300),
      onTimeout: () => local,
    ),
  ]);
}

// Задание 10
Timer? debounceTimer;

void debounce(String query) {
  debounceTimer?.cancel();

  debounceTimer = Timer(const Duration(milliseconds: 400), () {
    fetchData(query).then(print).catchError(print);
  });
}
```
---


### Блок 2: Потоковые данные, датчики и Stream API

11. **Слушатель котировок криптовалют:** Смоделируйте WebSocket-поток, передающий случайную цену биткоина раз в 200 мс, и отфильтруйте всплески более 5%.
12. **Стриминг прогресса скачивания файла:** Напишите асинхронный генератор `async*`, передающий процент загрузки от 0% до 100% с шагом в 10% каждые 300 мс.
13. **Объединение двух потоков (Merge):** Создайте функцию, объединяющую события из потока геолокации и потока компаса в единый поток навигации.
14. **Дросселирование потока (Throttle):** Реализуйте оператор, пропускающий не более одного клика по кнопке в секунду из непрерывного потока нажатий.
15. **Поток шагов акселерометра:** Сгенерируйте поток показаний ускорения по трем осям, подсчитывающий число шагов при превышении порогового значения.
16. **Автоматическое переподключение потока (Reconnection):** Напишите слушатель стрима чата, который при возникновении ошибки автоматически пересоздает подписку через 2 секунды.
17. **Буферизация потока пачками (Buffer / Chunk):** Накапливайте события кликов в список и отдавайте их пачками по 5 штук или раз в 3 секунды.
18. **Поток обратного отсчета для OTP-кода:** Создайте стрим таймера на 60 секунд для экрана ввода SMS с уведомлением о завершении срока жизни кода.
19. **Потоковый калькулятор скользящего среднего:** Спроектируйте стрим, принимающий числа и отдающий скользящее среднее значение последних 5 измерений.
20. **Преобразование Stream в Future:** Извлеките последнее значение из конечного потока с помощью метода `stream.last`.

## Решение Блок 2\*
```dart
// Блок 2. Потоковые данные, датчики и Stream API

import 'dart:async';
import 'dart:math';

// Задание 11
Stream<double> cryptoPrices() async* {
  Random random = Random();
  double price = 50000;

  while (true) {
    await Future.delayed(const Duration(milliseconds: 200));
    price += random.nextDouble() * 1000 - 500;
    yield price;
  }
}

Stream<double> filterSpikes(Stream<double> stream) {
  double previous = 50000;

  return stream.where((price) {
    double change = (price - previous).abs() / previous * 100;
    previous = price;
    return change > 5;
  });
}

// Задание 12
Stream<int> downloadProgress() async* {
  for (int i = 0; i <= 100; i += 10) {
    await Future.delayed(const Duration(milliseconds: 300));
    yield i;
  }
}

// Задание 13
Stream<dynamic> mergeStreams(
  Stream<dynamic> location,
  Stream<dynamic> compass,
) {
  StreamController<dynamic> controller = StreamController();

  location.listen(controller.add);
  compass.listen(controller.add);

  return controller.stream;
}

// Задание 14
Stream<T> throttle<T>(Stream<T> stream) {
  DateTime last = DateTime(2000);

  return stream.where((event) {
    DateTime now = DateTime.now();
    if (now.difference(last).inSeconds >= 1) {
      last = now;
      return true;
    }
    return false;
  });
}

// Задание 15
Stream<(double, double, double)> accelerometer() async* {
  Random random = Random();

  while (true) {
    await Future.delayed(const Duration(milliseconds: 200));
    yield (
      random.nextDouble() * 20,
      random.nextDouble() * 20,
      random.nextDouble() * 20,
    );
  }
}

Stream<int> countSteps(Stream<(double, double, double)> stream) async* {
  int steps = 0;

  await for (var data in stream) {
    if (data.$1.abs() + data.$2.abs() + data.$3.abs() > 25) {
      yield ++steps;
    }
  }
}

// Задание 16
void reconnect(Stream Function() createStream) {
  createStream().listen(
    print,
    onError: (e) {
      Future.delayed(const Duration(seconds: 2), () {
        reconnect(createStream);
      });
    },
  );
}

// Задание 17
Stream<List<T>> buffer<T>(Stream<T> stream) {
  StreamController<List<T>> controller = StreamController();
  List<T> items = [];

  Timer timer = Timer.periodic(const Duration(seconds: 3), (_) {
    if (items.isNotEmpty) {
      controller.add(List.from(items));
      items.clear();
    }
  });

  stream.listen(
    (data) {
      items.add(data);
      if (items.length == 5) {
        controller.add(List.from(items));
        items.clear();
      }
    },
    onDone: () {
      if (items.isNotEmpty) controller.add(items);
      timer.cancel();
      controller.close();
    },
  );

  return controller.stream;
}

// Задание 18
Stream<int> otpTimer() async* {
  for (int i = 60; i >= 0; i--) {
    await Future.delayed(const Duration(seconds: 1));
    yield i;
  }
  print("Срок действия кода истек");
}

// Задание 19
Stream<double> movingAverage(Stream<double> stream) async* {
  List<double> values = [];

  await for (double value in stream) {
    values.add(value);
    if (values.length > 5) values.removeAt(0);
    yield values.reduce((a, b) => a + b) / values.length;
  }
}

// Задание 20
Future<int> getLast(Stream<int> stream) {
  return stream.last;
}
```
---

### Блок 3: Реактивные шины событий и StreamController

21. **Шина глобальных событий (EventBus):** Создайте синглтон `EventBus` на базе широковещательного `StreamController` для обмена событиями между несвязанными экранами.
22. **Реактивная корзина интернет-магазина:** Реализуйте класс `CartStore`, транслирующий в стрим текущее количество товаров и итоговую сумму при вызовах `add()` и `remove()`.
23. **Счетчик непрочитанных сообщений:** Спроектируйте контроллер бейджа уведомлений, увеличивающий счетчик при новых событиях и сбрасывающий его в ноль при прочтении.
24. **Управление состоянием аудиоплеера:** Реализуйте `AudioPlayerBloc` со стримом состояний: `PlayerStopped`, `PlayerPlaying(track)`, `PlayerPaused`.
25. **Контроллер бесконечной ленты (Pagination Stream):** Спроектируйте стрим, подгружающий следующую страницу новостей при передаче события `LoadNextPage`.
26. **Реактивный переключатель темной темы:** Создайте менеджер темы приложения, отдающий `ThemeMode.dark` или `ThemeMode.light` через `BehaviorSubject`-подобный контроллер.
27. **Слушатель сетевого статуса (Connectivity Stream):** Контроллер, транслирующий изменения подключения (`WiFi`, `Cellular`, `None`) во все подписанные модули.
28. **Защита контроллера от утечек памяти (Auto-Dispose):** Напишите класс с механизмом проверки наличия подписчиков и автоматическим вызовом `close()` при обнулении слушателей.
29. **Валидация формы регистрации на стримах:** Создайте два потока для полей Email и Пароль, объедините их через `Rx.combineLatest`-подобную логику и отдавайте флаг активности кнопки `isSubmitEnabled`.
30. **Потоковый логгер крашей:** Направляйте все неотловленные ошибки приложения через `StreamController` в фоновый модуль отправки аналитики.

## Решение Блок 3\*
```dart
// Блок 3. Реактивные шины событий и StreamController

import 'dart:async';
import 'package:flutter/material.dart';

// Задание 21
class EventBus {
  EventBus._();

  static final EventBus instance = EventBus._();
  final _controller = StreamController<dynamic>.broadcast();

  Stream<dynamic> get stream => _controller.stream;

  void emit(dynamic event) => _controller.add(event);
}

// Задание 22
class CartStore {
  final _controller = StreamController<(int, double)>.broadcast();
  int count = 0;
  double total = 0;

  Stream<(int, double)> get stream => _controller.stream;

  void add(double price) {
    count++;
    total += price;
    _controller.add((count, total));
  }

  void remove(double price) {
    if (count > 0) {
      count--;
      total -= price;
      _controller.add((count, total));
    }
  }
}

// Задание 23
class NotificationCounter {
  final _controller = StreamController<int>.broadcast();
  int count = 0;

  Stream<int> get stream => _controller.stream;

  void increment() => _controller.add(++count);

  void reset() {
    count = 0;
    _controller.add(count);
  }
}

// Задание 24
abstract class PlayerState {}

class PlayerStopped extends PlayerState {}

class PlayerPlaying extends PlayerState {
  String track;
  PlayerPlaying(this.track);
}

class PlayerPaused extends PlayerState {}

class AudioPlayerBloc {
  final _controller = StreamController<PlayerState>.broadcast();

  Stream<PlayerState> get stream => _controller.stream;

  void play(String track) => _controller.add(PlayerPlaying(track));
  void pause() => _controller.add(PlayerPaused());
  void stop() => _controller.add(PlayerStopped());
}

// Задание 25
class LoadNextPage {}

class PaginationStream {
  final _controller = StreamController<LoadNextPage>();
  int page = 0;

  Stream<List<String>> get stream async* {
    await for (var event in _controller.stream) {
      page++;
      yield List.generate(10, (i) => "Новость ${page * 10 + i}");
    }
  }

  void loadNext() => _controller.add(LoadNextPage());
}

// Задание 26
class ThemeManager {
  final _controller = StreamController<ThemeMode>.broadcast();
  ThemeMode mode = ThemeMode.light;

  Stream<ThemeMode> get stream => _controller.stream;

  void toggle() {
    mode = mode == ThemeMode.light ? ThemeMode.dark : ThemeMode.light;
    _controller.add(mode);
  }
}

// Задание 27
enum Connectivity { wifi, cellular, none }

class ConnectivityController {
  final _controller = StreamController<Connectivity>.broadcast();

  Stream<Connectivity> get stream => _controller.stream;

  void update(Connectivity value) => _controller.add(value);
}

// Задание 28
class AutoDisposeController<T> {
  late final StreamController<T> controller;

  AutoDisposeController() {
    controller = StreamController<T>.broadcast(
      onCancel: () => controller.close(),
    );
  }

  Stream<T> get stream => controller.stream;
  void add(T value) {
    if (!controller.isClosed) controller.add(value);
  }
}

// Задание 29
class FormValidator {
  final email = StreamController<String>.broadcast();
  final password = StreamController<String>.broadcast();
  final _result = StreamController<bool>.broadcast();

  String _email = "";
  String _password = "";

  Stream<bool> get isSubmitEnabled => _result.stream;

  FormValidator() {
    email.stream.listen((value) {
      _email = value;
      _check();
    });

    password.stream.listen((value) {
      _password = value;
      _check();
    });
  }

  void _check() {
    _result.add(_email.contains("@") && _password.length >= 8);
  }
}

// Задание 30
class CrashLogger {
  final _controller = StreamController<Object>.broadcast();

  Stream<Object> get stream => _controller.stream;

  void log(Object error) => _controller.add(error);
}
```
---

### Блок 4: Изоляты и ресурсоемкие параллельные задачи

31. **Фоновая генерация миниатюр (Thumbnail Generator):** Обработайте массив байтов изображения в `Isolate.run()`, изменив его размер и вернув сжатый массив.
32. **Шифрование базы заметок:** Выполните шифрование большого текстового файла по алгоритму XOR / AES в параллельном изоляте без просадки FPS.
33. **Полнотекстовый поиск по 100 000 документов:** Передайте поисковый запрос и список документов в фоновый изолят, вернув 10 наиболее релевантных совпадений.
34. **Парсинг тяжелого GeoJSON карты города:** Распарсите координаты полигонов районов города в фоновом потоке памяти и верните готовые структуры данных.
35. **Сжатие логов в ZIP-архив:** Смоделируйте упаковку текстовых журналов в сжатый архив через изолят перед отправкой в техподдержку.
36. **Двусторонний пул воркеров (Worker Pool):** Создайте менеджер из двух постоянно работающих изолятов, распределяющий между ними математические задачи.
37. **Генерация PDF-отчета:** Сформируйте текстовую структуру финансового отчета за год в параллельном изоляте.
38. **Сортировка массива из 1 000 000 чисел:** Сравните время сортировки массива в главном потоке и в отдельном изоляте.
39. **Конвертер аудио-семплов:** Смоделируйте преобразование массива сырых аудио-семплов в изоляте.
40. **Безопасная передача больших объектов (TransferableTypedData):** Передайте большой бинарный буфер между изолятами без затратного копирования памяти.

## Решение Блок 4\*
```dart
// Блок 4 — Изоляты и ресурсоемкие параллельные задачи (31–40)
import 'dart:isolate';
import 'dart:typed_data';
import 'dart:convert';
import 'dart:async';

// 31
Future<Uint8List> generateThumbnail(Uint8List image) async {
  return Isolate.run(() {
    Uint8List resized = Uint8List.fromList(image.take(image.length ~/ 4).toList());
    return Uint8List.fromList(ZLibEncoder().convert(resized));
  });
}

// 32
Future<String> encrypt(String text, int key) async {
  return Isolate.run(() {
    return String.fromCharCodes(text.codeUnits.map((c) => c ^ key));
  });
}

// 33
Future<List<String>> search(List<String> documents, String query) async {
  return Isolate.run(() {
    return documents.where((doc) => doc.contains(query)).take(10).toList();
  });
}

// 34
Future<List<dynamic>> parseGeoJSON(String json) async {
  return Isolate.run(() {
    var data = jsonDecode(json);
    return data['features']
        .map((feature) => feature['geometry']['coordinates'])
        .toList();
  });
}

// 35
Future<List<int>> compressLogs(String logs) async {
  return Isolate.run(() {
    return ZLibEncoder().convert(utf8.encode(logs));
  });
}

// 36
void worker(SendPort mainPort) {
  ReceivePort workerPort = ReceivePort();
  mainPort.send(workerPort.sendPort);

  workerPort.listen((message) {
    int number = message[0];
    SendPort reply = message[1];
    reply.send(number * number);
  });
}

class WorkerPool {
  List<Isolate> isolates = [];
  List<SendPort> ports = [];
  int current = 0;

  Future<void> start() async {
    for (int i = 0; i < 2; i++) {
      ReceivePort port = ReceivePort();
      isolates.add(await Isolate.spawn(worker, port.sendPort));
      ports.add(await port.first);
      port.close();
    }
  }

  Future<int> execute(int number) async {
    ReceivePort result = ReceivePort();
    ports[current].send([number, result.sendPort]);
    current = (current + 1) % 2;
    return await result.first;
  }

  void dispose() {
    for (var isolate in isolates) {
      isolate.kill();
    }
  }
}

// 37
Future<String> generateReport() async {
  return Isolate.run(() {
    return '''
Финансовый отчет за год
Доход: 1000000
Расходы: 650000
Прибыль: 350000
''';
  });
}

// 38
Future<void> sortNumbers() async {
  List<int> numbers = List.generate(1000000, (i) => 1000000 - i);

  Stopwatch stopwatch = Stopwatch()..start();
  numbers.sort();
  print('Главный поток: ${stopwatch.elapsedMilliseconds} мс');

  stopwatch.reset();
  await Isolate.run(() {
    List<int> copy = List.generate(1000000, (i) => 1000000 - i);
    copy.sort();
  });
  print('Изолят: ${stopwatch.elapsedMilliseconds} мс');
}

// 39
Future<List<double>> convertAudio(List<int> samples) async {
  return Isolate.run(() {
    return samples.map((sample) => sample / 32768.0).toList();
  });
}

// 40
Future<int> processData(TransferableTypedData data) async {
  return Isolate.run(() {
    Uint8List bytes = data.materialize().asUint8List();
    return bytes.length;
  });
}
```

### Блок 5: Сложные асинхронные архитектуры и отказоустойчивость

41. **Отмена сетевой операции через CancelToken:** Реализуйте класс токена отмены, выбрасывающий `CancellationException` при прерывании пользователем ухода со страницы.
42. **Репликация базы данных с разрешением конфликтов:** Синхронизируйте две асинхронные очереди изменений с временными метками (Timestamps).
43. **Асинхронная очередь сохранения офлайн-заказов:** Сохраняйте оформленные без интернета заказы в локальную очередь и автоматически отправляйте их по одному при восстановлении сети.
44. **Паттерн Circuit Breaker:** Реализуйте защиту от падения сервера: если 5 запросов подряд завершились ошибкой, переведите сервис в режим «Разрыв цепи» на 30 секунд.
45. **Асинхронный семафор (Rate Limiter):** Ограничьте число одновременных сетевых обращений к API до 3 параллельных потоков.
46. **Гонка состояний при быстром вводе (Race Condition Fix):** Защитите поисковую строку: гарантируйте, что старый медленный ответ сервера не перезапишет новый быстрый ответ.
47. **Слушатель жизненного цикла приложения:** Свяжите события сворачивания приложения с мгновенной приостановкой всех активных `StreamSubscription`.
48. **Конечный автомат асинхронной синхронизации:** Реализуйте переключение стейтов `Idle` -> `Syncing(progress)` -> `Success` -> `Error` с защитой от повторного запуска во время синхронизации.
49. **Многоуровневый кэш (Memory -> Disk -> Network):** Спроектируйте функцию, последовательно проверяющую быструю оперативную память, затем локальный диск и только потом идущую в сеть.
50. **Итоговый реактивный модуль финансового терминала (Production Architecture):**
    - Подключите непрерывный поток биржевых котировок через WebSocket-имитатор.
    - Передайте расчет скользящей волатильности в параллельный изолят (`Isolate.run`).
    - Ограничьте частоту обновления графического экрана до 60 кадров/сек с помощью троттлинга.
    - Реализуйте автоматическое восстановление соединения при обрыве и безопасный вызов `dispose()` для всех контроллеров и подписок.

---

## Решение Блок 5\*
```dart
// Блок 5 — Сложные асинхронные архитектуры и отказоустойчивость (41–50)
import 'dart:async';
import 'dart:isolate';

// 41
class CancellationException implements Exception {}

class CancelToken {
  bool cancelled = false;
  Completer<void> completer = Completer<void>();

  void cancel() {
    cancelled = true;
    completer.complete();
  }

  Future<T> run<T>(Future<T> operation) {
    return Future.any([
      operation,
      completer.future.then((_) => throw CancellationException()),
    ]);
  }
}

// 42
class Change {
  String value;
  DateTime timestamp;

  Change(this.value, this.timestamp);
}

Future<List<Change>> replicate(List<Change> a, List<Change> b) async {
  await Future.wait([
    Future.delayed(Duration(milliseconds: 100)),
    Future.delayed(Duration(milliseconds: 100)),
  ]);

  Map<String, Change> result = {};

  for (var change in [...a, ...b]) {
    if (!result.containsKey(change.value) ||
        change.timestamp.isAfter(result[change.value]!.timestamp)) {
      result[change.value] = change;
    }
  }

  return result.values.toList();
}

// 43
class OrderQueue {
  List<String> orders = [];
  bool online = false;
  bool sending = false;

  void add(String order) {
    orders.add(order);
    send();
  }

  Future<void> send() async {
    if (!online || sending) return;

    sending = true;

    while (online && orders.isNotEmpty) {
      String order = orders.removeAt(0);
      await Future.delayed(Duration(seconds: 1));
    }

    sending = false;
  }
}

// 44
class CircuitBreaker {
  int errors = 0;
  bool open = false;

  Future<void> request(Future<void> Function() task) async {
    if (open) throw Exception('Разрыв цепи');

    try {
      await task();
      errors = 0;
    } catch (e) {
      errors++;

      if (errors >= 5) {
        open = true;
        Timer(Duration(seconds: 30), () {
          open = false;
          errors = 0;
        });
      }

      rethrow;
    }
  }
}

// 45
class Semaphore {
  int active = 0;
  List<Completer<void>> queue = [];

  Future<void> acquire() async {
    if (active >= 3) {
      Completer<void> c = Completer<void>();
      queue.add(c);
      await c.future;
    }
    active++;
  }

  void release() {
    active--;
    if (queue.isNotEmpty) queue.removeAt(0).complete();
  }

  Future<T> run<T>(Future<T> Function() task) async {
    await acquire();
    try {
      return await task();
    } finally {
      release();
    }
  }
}

// 46
class Search {
  int current = 0;

  Future<String?> search(String query) async {
    int id = ++current;
    String result = await Future.delayed(
      Duration(milliseconds: 500),
      () => query,
    );
    return id == current ? result : null;
  }
}

// 47
class Lifecycle {
  List<StreamSubscription> subscriptions = [];

  void add(StreamSubscription subscription) {
    subscriptions.add(subscription);
  }

  Future<void> pause() async {
    for (var sub in subscriptions) {
      sub.pause();
    }
  }

  Future<void> resume() async {
    for (var sub in subscriptions) {
      sub.resume();
    }
  }

  Future<void> dispose() async {
    for (var sub in subscriptions) {
      await sub.cancel();
    }
  }
}

// 48
enum SyncState { idle, syncing, success, error }

class SyncManager {
  SyncState state = SyncState.idle;
  double progress = 0;

  Future<void> sync() async {
    if (state == SyncState.syncing) return;

    state = SyncState.syncing;

    try {
      for (int i = 1; i <= 10; i++) {
        await Future.delayed(Duration(milliseconds: 200));
        progress = i / 10;
      }
      state = SyncState.success;
    } catch (e) {
      state = SyncState.error;
    }
  }
}

// 49
class Cache {
  Map<String, String> memory = {};
  Map<String, String> disk = {};

  Future<String> get(
    String key,
    Future<String> Function() network,
  ) async {
    if (memory.containsKey(key)) return memory[key]!;
    if (disk.containsKey(key)) {
      memory[key] = disk[key]!;
      return disk[key]!;
    }

    String value = await network();
    disk[key] = value;
    memory[key] = value;
    return value;
  }
}

// 50
class FinancialTerminal {
  StreamController<double> controller = StreamController<double>();
  StreamSubscription<double>? subscription;
  bool disposed = false;

  Stream<double> connect() async* {
    while (!disposed) {
      try {
        for (int i = 0; i < 20; i++) {
          yield 100 + i.toDouble();
          await Future.delayed(Duration(milliseconds: 10));
        }
        throw Exception('Соединение потеряно');
      } catch (e) {
        await Future.delayed(Duration(seconds: 1));
      }
    }
  }

  Future<double> volatility(List<double> prices) async {
    return Isolate.run(() {
      double sum = 0;
      for (int i = 1; i < prices.length; i++) {
        sum += (prices[i] - prices[i - 1]).abs();
      }
      return sum / (prices.length - 1);
    });
  }

  Future<void> start() async {
    List<double> prices = [];
    Stopwatch stopwatch = Stopwatch()..start();

    subscription = connect().listen((price) async {
      prices.add(price);
      if (prices.length > 20) prices.removeAt(0);

      if (prices.length >= 2 && stopwatch.elapsedMilliseconds >= 16) {
        stopwatch.reset();
        controller.add(await volatility(prices));
      }
    });
  }

  Future<void> dispose() async {
    disposed = true;
    await subscription?.cancel();
    await controller.close();
  }
}
```
