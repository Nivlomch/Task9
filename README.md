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

## Содержание

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

```text
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

```dart
class AuthException implements Exception {
  final String message;

  AuthException(this.message);
}

Future<bool> login(String user, String pass) async {
  await Future.delayed(const Duration(milliseconds: 1500));

  if (pass == "1234") {
    return true;
  } else {
    throw AuthException("Неверный пароль");
  }
}

Future<void> readConfig(String name, int seconds) async {
  await Future.delayed(Duration(seconds: seconds));
  return Future.value();
}

Future<Map<String, String>> readAllConfigs() async {
  var results = await Future.wait([
    Future.delayed(const Duration(seconds: 1), () => "Database config"),
    Future.delayed(const Duration(seconds: 2), () => "Network config"),
    Future.delayed(const Duration(seconds: 3), () => "Server config"),
  ]);

  return {
    "database": results[0],
    "network": results[1],
    "server": results[2],
  };
}

Future<String> requestMirror1() async {
  await Future.delayed(const Duration(milliseconds: 800));
  return "Ответ от зеркала 1";
}

Future<String> requestMirror2() async {
  await Future.delayed(const Duration(milliseconds: 1200));
  return "Ответ от зеркала 2";
}

Future<String> fastestRequest() {
  return Future.any([
    requestMirror1(),
    requestMirror2(),
  ]);
}

Future<T> retry<T>(Future<T> Function() task, int maxAttempts) async {
  for (int i = 0; i < maxAttempts; i++) {
    try {
      return await task();
    } catch (e) {
      if (i == maxAttempts - 1) {
        rethrow;
      }

      await Future.delayed(const Duration(seconds: 1));
    }
  }

  throw Exception("Все попытки завершились ошибкой");
}

Future<bool> loginWithoutAsync(String user, String pass) {
  return Future.delayed(const Duration(milliseconds: 1500))
      .then((value) {
    if (pass == "1234") {
      return true;
    } else {
      throw AuthException("Неверный пароль");
    }
  }).catchError((error) {
    throw error;
  }).whenComplete(() {
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

```dart
Stream<int> stopWatch() async* {
  int seconds = 0;

  while (true) {
    await Future.delayed(const Duration(seconds: 1));
    yield seconds++;
  }
}

Stream<(double, double)> gpsCoordinates() async* {
  double lat = 55.7558;
  double lon = 37.6173;

  while (true) {
    await Future.delayed(const Duration(milliseconds: 500));

    lat += 0.0001;
    lon += 0.0001;

    yield (lat, lon);
  }
}

Stream<double> priceStream() async* {
  for (double price in [100.0, 101.5, 102.0, 99.5, 103.0]) {
    await Future.delayed(const Duration(seconds: 1));
    yield price;
  }
}

void subscribeToPrices() {
  Stream<double> broadcastStream = priceStream().asBroadcastStream();

  broadcastStream.listen(
    (price) => print("UI-виджет: $price"),
  );

  broadcastStream.listen(
    (price) => print("Логгер аналитики: $price"),
  );
}

Future<int> calculateSum() async {
  Stream<int> numbers = Stream.fromIterable(
    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
  );

  return await numbers.fold(0, (sum, number) => sum + number);
}

Stream<int> firstStream() async* {
  yield 1;
  yield 2;
  yield 3;
}

Stream<int> secondStream() async* {
  yield 4;
  yield 5;
  yield 6;
}

Stream<int> combinedStream() async* {
  yield* firstStream();
  yield* secondStream();
}
```

---

## Тема 3. Реактивное управление: StreamController и StreamTransformer

`StreamController<T>` — это программный пульт управления потоком, разделенный на две части:

- **`Sink` (Входной порт):** Место, куда события отправляются (`sink.add()`, `sink.addError()`).
- **`Stream` (Выходной порт):** Поток, на который подписываются клиенты для чтения данных.

```text
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

```dart
import 'dart:async';

final upperCaseTransformer = StreamTransformer<String, String>.fromHandlers(
  handleData: (data, sink) {
    sink.add(data.toUpperCase());
  },
);

class SearchControllerWrapper {
  final StreamController<String> _controller =
      StreamController<String>.broadcast();

  Stream<String> get stream => _controller.stream;

  void addInput(String input) {
    _controller.sink.add(input.trim());
  }

  void dispose() {
    _controller.close();
  }
}

void handleControllerError() {
  final controller = StreamController<String>();

  controller.stream.listen(
    (data) => print(data),
    onError: (error) => print("Ошибка: $error"),
  );

  controller.sink.addError(Exception("Сбой датчика"));
  controller.close();
}

void addEvent(StreamController<int> controller, int value) {
  if (!controller.isClosed) {
    controller.sink.add(value);
  }
}

void trackSubscribers() {
  final controller = StreamController<int>(
    onListen: () => print("Подписчик подключился"),
    onCancel: () => print("Подписчик отключился"),
  );

  final subscription = controller.stream.listen(
    (value) => print(value),
  );

  controller.sink.add(1);
  subscription.cancel();
  controller.close();
}
```

---

## Тема 4. Истинная многопоточность: Изоляты (Isolates) и Isolate.run

Если задача требует непрерывных тяжелых математических расчетов (например, парсинг 50-мегабайтного JSON, фильтрация графических пикселей, шифрование больших баз данных), она заблокирует Event Loop главного потока, вызвав зависание мобильного экрана (Jank).

Для тяжелых фоновых вычислений Dart использует **Изоляты** (Isolates).

### Отличие изолятов от традиционных потоков Java/C++

- Изоляты **НЕ имеют общей разделяемой памяти** (Shared Memory отсутствует).
- Нет блокировок, взаимных дедлоков и мьютексов.
- Общение между изолятами происходит исключительно через передачу сообщений по системным портам (`SendPort` и `ReceivePort`), либо через метод `Isolate.run()` (начиная с Dart 2.19+).

```text
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

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

class Record {
  final int id;
  final String name;

  Record(this.id, this.name);

  factory Record.fromJson(Map<String, dynamic> json) {
    return Record(json['id'], json['name']);
  }
}

List<Record> parseJson(String jsonString) {
  List<dynamic> data = jsonDecode(jsonString);

  return data
      .map((item) => Record.fromJson(item))
      .toList();
}

Future<void> parallelJsonParser() async {
  String jsonString = jsonEncode(
    List.generate(
      10000,
      (i) => {'id': i, 'name': 'Record $i'},
    ),
  );

  List<Record> records = await Isolate.run(
    () => parseJson(jsonString),
  );

  print(records.length);
}

List<int> findPrimes(int limit) {
  List<int> primes = [];

  for (int i = 2; i <= limit; i++) {
    bool isPrime = true;

    for (int j = 2; j * j <= i; j++) {
      if (i % j == 0) {
        isPrime = false;
        break;
      }
    }

    if (isPrime) {
      primes.add(i);
    }
  }

  return primes;
}

Future<void> searchPrimes() async {
  Timer.periodic(const Duration(seconds: 1), (timer) {
    print(".");
  });

  List<int> primes = await Isolate.run(
    () => findPrimes(1000000),
  );

  print("Количество простых чисел: ${primes.length}");
}

void calculatorWorker(SendPort mainSendPort) {
  final receivePort = ReceivePort();

  mainSendPort.send(receivePort.sendPort);

  receivePort.listen((message) {
    int a = message[0];
    int b = message[1];
    String operation = message[2];

    int result = 0;

    if (operation == "+") {
      result = a + b;
    } else if (operation == "-") {
      result = a - b;
    } else if (operation == "*") {
      result = a * b;
    }

    mainSendPort.send(result);
  });
}

Future<void> calculator() async {
  final receivePort = ReceivePort();

  await Isolate.spawn(
    calculatorWorker,
    receivePort.sendPort,
  );

  SendPort workerSendPort = await receivePort.first as SendPort;

  final resultPort = ReceivePort();

  workerSendPort.send([10, 5, "+"]);

  print(await resultPort.first);
}

int heavyCalculation() {
  int sum = 0;

  for (int i = 0; i < 2000000000; i++) {
    sum += i % 3;
  }

  return sum;
}

Future<void> measurePerformance() async {
  Timer.periodic(const Duration(seconds: 1), (timer) {
    print("Таймер работает");
  });

  Stopwatch stopwatch = Stopwatch()..start();

  int syncResult = heavyCalculation();

  stopwatch.stop();

  print("Синхронно: ${stopwatch.elapsedMilliseconds} мс");
  print(syncResult);

  stopwatch.reset();
  stopwatch.start();

  int isolateResult = await Isolate.run(
    () => heavyCalculation(),
  );

  stopwatch.stop();

  print("Изолят: ${stopwatch.elapsedMilliseconds} мс");
  print(isolateResult);
}

void infiniteWorker(SendPort sendPort) {
  while (true) {}
}

Future<void> killIsolate() async {
  final receivePort = ReceivePort();

  Isolate isolate = await Isolate.spawn(
    infiniteWorker,
    receivePort.sendPort,
  );

  await Future.delayed(const Duration(seconds: 2));

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

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

Future<String> fetchData(String url) async {
  await Future.delayed(const Duration(milliseconds: 1200));

  int statusCode = 200;

  if (url.contains("404")) {
    statusCode = 404;
  } else if (url.contains("500")) {
    statusCode = 500;
  }

  if (statusCode == 200) {
    return jsonEncode({
      "status": 200,
      "url": url,
      "data": "Данные получены",
    });
  } else if (statusCode == 404) {
    throw HttpException("Ошибка 404: Не найдено");
  } else {
    throw HttpException("Ошибка 500: Ошибка сервера");
  }
}

Future<T> fetchWithRetry<T>(Future<T> Function() request) async {
  int retryCount = 0;
  List<int> delays = [1, 2, 4, 8];

  while (true) {
    try {
      return await request();
    } on SocketException {
      if (retryCount >= delays.length) {
        rethrow;
      }

      await Future.delayed(Duration(seconds: delays[retryCount]));
      retryCount++;
    }
  }
}

Future<Map<String, String>> fetchImageMetadata() async {
  List<String> urls = List.generate(
    5,
    (index) => "image_$index.jpg",
  );

  Map<String, String> results = {};
  int index = 0;

  Future<void> worker() async {
    while (index < urls.length) {
      String url = urls[index++];

      await Future.delayed(const Duration(seconds: 1));

      results[url] = "Метаданные $url";
    }
  }

  await Future.wait([
    worker(),
    worker(),
  ]);

  return results;
}

Future<String> pingMirror(String url, int milliseconds) async {
  await Future.delayed(Duration(milliseconds: milliseconds));
  return url;
}

Future<String> findFastestServer() {
  return Future.any([
    pingMirror("https://mirror1.com", 800),
    pingMirror("https://mirror2.com", 1200),
    pingMirror("https://mirror3.com", 1000),
  ]);
}

class CancelToken {
  bool isCancelled = false;

  void cancel() {
    isCancelled = true;
  }
}

Future<void> saveProfile(CancelToken token) async {
  for (int i = 0; i < 10; i++) {
    await Future.delayed(const Duration(seconds: 1));

    if (token.isCancelled) {
      throw Exception("Операция прервана");
    }
  }
}

Future<void> saveProfileWithTimeout() async {
  final token = CancelToken();

  try {
    await Future.any([
      saveProfile(token),
      Future.delayed(const Duration(seconds: 3), () {
        token.cancel();
        throw TimeoutException("Превышено время сохранения профиля");
      }),
    ]);
  } catch (e) {
    print(e);
  }
}

class AsyncQueue {
  Future<void> _tail = Future.value();

  Future<T> add<T>(Future<T> Function() task) {
    final completer = Completer<T>();

    _tail = _tail.then((_) async {
      try {
        completer.complete(await task());
      } catch (e, stackTrace) {
        completer.completeError(e, stackTrace);
      }
    });

    return completer.future;
  }
}

Future<int> benchmark(Future Function() task) async {
  Stopwatch stopwatch = Stopwatch()..start();

  await task();

  stopwatch.stop();

  return stopwatch.elapsedMilliseconds;
}

class AppConfig {
  final Map<String, dynamic> data;

  AppConfig._(this.data);

  static Future<AppConfig>? _instance;

  static Future<AppConfig> getInstance() {
    return _instance ??= _load();
  }

  static Future<AppConfig> _load() async {
    await Future.delayed(const Duration(seconds: 1));

    return AppConfig._({
      "appName": "MyApplication",
      "version": "1.0.0",
    });
  }
}

Future<String> readLocalDatabase() async {
  await Future.delayed(const Duration(milliseconds: 200));
  return "Локальные данные";
}

Future<String> fetchNetworkData() async {
  await Future.delayed(const Duration(seconds: 1));
  return "Сетевые данные";
}

Future<String> loadCache() async {
  Future<String> localData = readLocalDatabase();

  return fetchNetworkData().timeout(
    const Duration(milliseconds: 300),
    onTimeout: () => localData,
  );
}

class Debouncer {
  final Duration delay;
  final void Function(String) onSearch;

  Timer? _timer;

  Debouncer({
    required this.delay,
    required this.onSearch,
  });

  void search(String query) {
    _timer?.cancel();

    _timer = Timer(delay, () {
      onSearch(query);
    });
  }

  void dispose() {
    _timer?.cancel();
  }
}
```

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

```dart
import 'dart:async';
import 'dart:math';

Stream<double> bitcoinPrices() async* {
  final random = Random();
  double price = 90000;

  while (true) {
    await Future.delayed(const Duration(milliseconds: 200));

    price += (random.nextDouble() - 0.5) * 1000;

    yield price;
  }
}

Stream<double> filterPriceSpikes(Stream<double> stream) async* {
  double? previousPrice;

  await for (final price in stream) {
    if (previousPrice == null ||
        ((price - previousPrice).abs() / previousPrice) <= 0.05) {
      yield price;
    }

    previousPrice = price;
  }
}

Stream<int> downloadProgress() async* {
  for (int i = 0; i <= 100; i += 10) {
    await Future.delayed(const Duration(milliseconds: 300));
    yield i;
  }
}

Stream<String> mergeNavigationStreams(
  Stream<String> locationStream,
  Stream<String> compassStream,
) {
  return StreamGroup.merge([
    locationStream,
    compassStream,
  ]);
}

Stream<T> throttle<T>(Stream<T> stream) async* {
  DateTime? lastEvent;

  await for (final event in stream) {
    DateTime now = DateTime.now();

    if (lastEvent == null ||
        now.difference(lastEvent).inMilliseconds >= 1000) {
      lastEvent = now;
      yield event;
    }
  }
}

Stream<(double, double, double)> accelerometer() async* {
  final random = Random();

  while (true) {
    await Future.delayed(const Duration(milliseconds: 200));

    yield (
      random.nextDouble() * 3,
      random.nextDouble() * 3,
      random.nextDouble() * 3,
    );
  }
}

Stream<int> countSteps(
  Stream<(double, double, double)> stream,
) async* {
  int steps = 0;

  await for (final acceleration in stream) {
    double magnitude = sqrt(
      acceleration.$1 * acceleration.$1 +
          acceleration.$2 * acceleration.$2 +
          acceleration.$3 * acceleration.$3,
    );

    if (magnitude > 2.5) {
      steps++;
      yield steps;
    }
  }
}

Stream<String> createChatStream() async* {
  await Future.delayed(const Duration(seconds: 1));
  yield "Новое сообщение";
  throw Exception("Соединение потеряно");
}

void listenToChat() {
  late StreamSubscription<String> subscription;

  void connect() {
    subscription = createChatStream().listen(
      (message) => print(message),
      onError: (error) async {
        await Future.delayed(const Duration(seconds: 2));
        connect();
      },
    );
  }

  connect();
}

Stream<List<T>> bufferClicks<T>(Stream<T> stream) async* {
  final buffer = <T>[];
  final controller = StreamController<List<T>>();

  Timer? timer;

  void flush() {
    if (buffer.isNotEmpty) {
      controller.add(List<T>.from(buffer));
      buffer.clear();
    }
  }

  timer = Timer.periodic(
    const Duration(seconds: 3),
    (_) => flush(),
  );

  final subscription = stream.listen(
    (event) {
      buffer.add(event);

      if (buffer.length >= 5) {
        flush();
      }
    },
    onError: controller.addError,
    onDone: () {
      flush();
      timer?.cancel();
      controller.close();
    },
  );

  yield* controller.stream;

  await subscription.cancel();
}

Stream<int> otpCountdown() async* {
  for (int i = 60; i >= 0; i--) {
    await Future.delayed(const Duration(seconds: 1));
    yield i;
  }

  print("Срок действия OTP-кода истек");
}

Stream<double> movingAverage(Stream<double> stream) async* {
  final values = <double>[];

  await for (final value in stream) {
    values.add(value);

    if (values.length > 5) {
      values.removeAt(0);
    }

    double average =
        values.reduce((a, b) => a + b) / values.length;

    yield average;
  }
}

Future<void> getLastValue() async {
  Stream<int> stream = Stream.fromIterable([10, 20, 30, 40, 50]);

  int lastValue = await stream.last;

  print(lastValue);
}
```

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

```dart
import 'dart:async';

class EventBus {
  EventBus._();

  static final EventBus instance = EventBus._();

  final StreamController<Object> _controller =
      StreamController<Object>.broadcast();

  Stream<Object> get stream => _controller.stream;

  void fire(Object event) {
    _controller.sink.add(event);
  }

  void dispose() {
    _controller.close();
  }
}

class CartState {
  final int itemCount;
  final double totalPrice;

  CartState(this.itemCount, this.totalPrice);
}

class CartStore {
  final StreamController<CartState> _controller =
      StreamController<CartState>.broadcast();

  final Map<String, double> _products = {};

  Stream<CartState> get stream => _controller.stream;

  void add(String name, double price) {
    _products[name] = (_products[name] ?? 0) + price;
    _update();
  }

  void remove(String name) {
    _products.remove(name);
    _update();
  }

  void _update() {
    double total = _products.values.fold(0, (a, b) => a + b);

    _controller.add(
      CartState(_products.length, total),
    );
  }

  void dispose() {
    _controller.close();
  }
}

class NotificationBadge {
  final StreamController<int> _controller =
      StreamController<int>.broadcast();

  int _count = 0;

  Stream<int> get stream => _controller.stream;

  void increment() {
    _count++;
    _controller.add(_count);
  }

  void reset() {
    _count = 0;
    _controller.add(_count);
  }

  void dispose() {
    _controller.close();
  }
}

abstract class PlayerState {}

class PlayerStopped extends PlayerState {}

class PlayerPlaying extends PlayerState {
  final String track;

  PlayerPlaying(this.track);
}

class PlayerPaused extends PlayerState {}

class AudioPlayerBloc {
  final StreamController<PlayerState> _controller =
      StreamController<PlayerState>.broadcast();

  Stream<PlayerState> get stream => _controller.stream;

  void play(String track) {
    _controller.add(PlayerPlaying(track));
  }

  void pause() {
    _controller.add(PlayerPaused());
  }

  void stop() {
    _controller.add(PlayerStopped());
  }

  void dispose() {
    _controller.close();
  }
}

class LoadNextPage {}

class PaginationController {
  final StreamController<LoadNextPage> _requestController =
      StreamController<LoadNextPage>();

  final StreamController<List<String>> _newsController =
      StreamController<List<String>>.broadcast();

  int _page = 0;

  Stream<List<String>> get stream => _newsController.stream;

  PaginationController() {
    _requestController.stream.listen((event) async {
      _page++;

      await Future.delayed(const Duration(seconds: 1));

      _newsController.add([
        "Новость $_page.1",
        "Новость $_page.2",
        "Новость $_page.3",
      ]);
    });
  }

  void loadNextPage() {
    _requestController.add(LoadNextPage());
  }

  void dispose() {
    _requestController.close();
    _newsController.close();
  }
}

enum ThemeMode {
  dark,
  light,
}

class ThemeManager {
  final StreamController<ThemeMode> _controller =
      StreamController<ThemeMode>.broadcast();

  ThemeMode _theme = ThemeMode.light;

  Stream<ThemeMode> get stream => _controller.stream;

  ThemeMode get currentTheme => _theme;

  void toggleTheme() {
    _theme = _theme == ThemeMode.light
        ? ThemeMode.dark
        : ThemeMode.light;

    _controller.add(_theme);
  }

  void dispose() {
    _controller.close();
  }
}

enum ConnectivityStatus {
  WiFi,
  Cellular,
  None,
}

class ConnectivityController {
  final StreamController<ConnectivityStatus> _controller =
      StreamController<ConnectivityStatus>.broadcast();

  Stream<ConnectivityStatus> get stream => _controller.stream;

  void updateStatus(ConnectivityStatus status) {
    _controller.add(status);
  }

  void dispose() {
    _controller.close();
  }
}

class AutoDisposeController<T> {
  late final StreamController<T> _controller;

  int _listeners = 0;

  AutoDisposeController() {
    _controller = StreamController<T>(
      onListen: () {
        _listeners++;
      },
      onCancel: () {
        _listeners--;

        if (_listeners <= 0) {
          close();
        }
      },
    );
  }

  Stream<T> get stream => _controller.stream;

  void add(T value) {
    if (!_controller.isClosed) {
      _controller.add(value);
    }
  }

  void close() {
    if (!_controller.isClosed) {
      _controller.close();
    }
  }
}

class RegistrationController {
  final StreamController<String> _emailController =
      StreamController<String>.broadcast();

  final StreamController<String> _passwordController =
      StreamController<String>.broadcast();

  final StreamController<bool> _submitController =
      StreamController<bool>.broadcast();

  String _email = "";
  String _password = "";

  Stream<bool> get isSubmitEnabled => _submitController.stream;

  RegistrationController() {
    _emailController.stream.listen((email) {
      _email = email;
      _validate();
    });

    _passwordController.stream.listen((password) {
      _password = password;
      _validate();
    });
  }

  void updateEmail(String email) {
    _emailController.add(email);
  }

  void updatePassword(String password) {
    _passwordController.add(password);
  }

  void _validate() {
    bool isValid = _email.contains("@") && _password.length >= 8;

    _submitController.add(isValid);
  }

  void dispose() {
    _emailController.close();
    _passwordController.close();
    _submitController.close();
  }
}

class CrashLogger {
  final StreamController<Object> _controller =
      StreamController<Object>.broadcast();

  Stream<Object> get stream => _controller.stream;

  CrashLogger() {
    stream.listen((error) {
      sendAnalytics(error);
    });
  }

  void logError(Object error) {
    _controller.add(error);
  }

  Future<void> sendAnalytics(Object error) async {
    await Future.delayed(const Duration(seconds: 1));
    print("Ошибка отправлена в аналитику: $error");
  }

  void dispose() {
    _controller.close();
  }
}
```

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

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';
import 'dart:math';
import 'dart:typed_data';
import 'package:archive/archive.dart';

Uint8List generateThumbnail(Uint8List imageBytes) {
  return Isolate.run(() {
    int newLength = imageBytes.length ~/ 4;

    return Uint8List.fromList(
      imageBytes.sublist(0, newLength),
    );
  }) as Uint8List;
}

Uint8List encryptNotes(String text) {
  final bytes = utf8.encode(text);

  return Uint8List.fromList(
    bytes.map((byte) => byte ^ 123).toList(),
  );
}

Future<Uint8List> encryptNotesInIsolate(String text) {
  return Isolate.run(() => encryptNotes(text));
}

class SearchResult {
  final String document;
  final int relevance;

  SearchResult(this.document, this.relevance);
}

List<SearchResult> searchDocuments(
  List<String> documents,
  String query,
) {
  List<SearchResult> results = [];

  for (String document in documents) {
    int relevance = query.allMatches(
      document.toLowerCase(),
    ).length;

    if (relevance > 0) {
      results.add(SearchResult(document, relevance));
    }
  }

  results.sort(
    (a, b) => b.relevance.compareTo(a.relevance),
  );

  return results.take(10).toList();
}

Future<List<SearchResult>> fullTextSearch(
  List<String> documents,
  String query,
) {
  return Isolate.run(
    () => searchDocuments(documents, query),
  );
}

List<List<List<double>>> parseGeoJson(String jsonString) {
  final data = jsonDecode(jsonString);

  List<List<List<double>>> polygons = [];

  for (var feature in data['features']) {
    var coordinates =
        feature['geometry']['coordinates'];

    for (var polygon in coordinates) {
      List<List<double>> points = [];

      for (var point in polygon) {
        points.add([
          point[0].toDouble(),
          point[1].toDouble(),
        ]);
      }

      polygons.add(points);
    }
  }

  return polygons;
}

Future<List<List<List<double>>>> parseGeoJsonInIsolate(
  String jsonString,
) {
  return Isolate.run(() => parseGeoJson(jsonString));
}

Future<List<int>> compressLogs(List<String> logs) {
  return Isolate.run(() {
    String text = logs.join('\n');
    List<int> bytes = utf8.encode(text);

    return GZipEncoder().encode(bytes);
  });
}

void workerEntryPoint(SendPort mainPort) {
  final receivePort = ReceivePort();

  mainPort.send(receivePort.sendPort);

  receivePort.listen((message) {
    int id = message[0];
    int number = message[1];
    SendPort replyPort = message[2];

    int result = number * number;

    replyPort.send([id, result]);
  });
}

class WorkerPool {
  final List<SendPort> _workers = [];
  final ReceivePort _receivePort = ReceivePort();

  int _currentWorker = 0;
  int _taskId = 0;

  final Map<int, Completer<int>> _tasks = {};

  Future<void> start() async {
    _receivePort.listen((message) {
      int id = message[0];
      int result = message[1];

      _tasks.remove(id)?.complete(result);
    });

    for (int i = 0; i < 2; i++) {
      final port = ReceivePort();

      await Isolate.spawn(
        workerEntryPoint,
        port.sendPort,
      );

      SendPort worker = await port.first as SendPort;

      _workers.add(worker);
    }
  }

  Future<int> execute(int number) {
    int id = _taskId++;
    final completer = Completer<int>();

    _tasks[id] = completer;

    _workers[_currentWorker].send([
      id,
      number,
      _receivePort.sendPort,
    ]);

    _currentWorker = (_currentWorker + 1) % 2;

    return completer.future;
  }

  void dispose() {
    _receivePort.close();
  }
}

String generateFinancialReport(Map<String, double> finances) {
  StringBuffer report = StringBuffer();

  report.writeln("Финансовый отчет за год");
  report.writeln("

---

---

---

---

---

---

---

---

");

  finances.forEach((month, amount) {
    report.writeln("$month: $amount");
  });

  double total = finances.values.fold(
    0,
    (sum, value) => sum + value,
  );

  report.writeln("

---

---

---

---

---

---

---

---

");
  report.writeln("Итого: $total");

  return report.toString();
}

Future<String> generatePdfReport(
  Map<String, double> finances,
) {
  return Isolate.run(
    () => generateFinancialReport(finances),
  );
}

Future<void> compareSorting() async {
  List<int> numbers = List.generate(
    1000000,
    (_) => Random().nextInt(1000000),
  );

  Stopwatch stopwatch = Stopwatch()..start();

  List<int> syncNumbers = List.from(numbers);
  syncNumbers.sort();

  stopwatch.stop();

  print(
    "Главный поток: ${stopwatch.elapsedMilliseconds} мс",
  );

  stopwatch.reset();
  stopwatch.start();

  List<int> isolateNumbers = await Isolate.run(() {
    List<int> sorted = List.from(numbers);
    sorted.sort();
    return sorted;
  });

  stopwatch.stop();

  print(
    "Изолят: ${stopwatch.elapsedMilliseconds} мс",
  );
}

Future<Float32List> convertAudioSamples(
  Float32List samples,
) {
  return Isolate.run(() {
    return Float32List.fromList(
      samples.map((sample) => sample / 32768.0).toList(),
    );
  });
}

Future<TransferableTypedData> transferLargeBuffer(
  TransferableTypedData data,
) async {
  return await Isolate.run(() {
    return data;
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

import 'dart:async';
import 'dart:convert';
import 'dart:io';

// Блок 1. Сетевые запросы, таймеры и Future

Future<String> fetchData(String url) async {
  await Future.delayed(const Duration(milliseconds: 1200));

  int statusCode = 200;

  if (url.contains("404")) {
    statusCode = 404;
  } else if (url.contains("500")) {
    statusCode = 500;
  }

  if (statusCode == 200) {
    return jsonEncode({
      "status": 200,
      "url": url,
      "data": "Данные получены",
    });
  } else if (statusCode == 404) {
    throw HttpException("Ошибка 404: Не найдено");
  } else {
    throw HttpException("Ошибка 500: Ошибка сервера");
  }
}

Future<T> fetchWithRetry<T>(Future<T> Function() request) async {
  int retryCount = 0;
  List<int> delays = [1, 2, 4, 8];

  while (true) {
    try {
      return await request();
    } on SocketException {
      if (retryCount >= delays.length) {
        rethrow;
      }

      await Future.delayed(Duration(seconds: delays[retryCount]));
      retryCount++;
    }
  }
}

Future<Map<String, String>> fetchImageMetadata() async {
  List<String> urls = List.generate(
    5,
    (index) => "image_$index.jpg",
  );

  Map<String, String> results = {};
  int index = 0;

  Future<void> worker() async {
    while (index < urls.length) {
      String url = urls[index++];

      await Future.delayed(const Duration(seconds: 1));

      results[url] = "Метаданные $url";
    }
  }

  await Future.wait([
    worker(),
    worker(),
  ]);

  return results;
}

Future<String> pingMirror(String url, int milliseconds) async {
  await Future.delayed(Duration(milliseconds: milliseconds));
  return url;
}

Future<String> findFastestServer() {
  return Future.any([
    pingMirror("https://mirror1.com", 800),
    pingMirror("https://mirror2.com", 1200),
    pingMirror("https://mirror3.com", 1000),
  ]);
}

class CancelToken {
  bool isCancelled = false;

  void cancel() {
    isCancelled = true;
  }
}

Future<void> saveProfile(CancelToken token) async {
  for (int i = 0; i < 10; i++) {
    await Future.delayed(const Duration(seconds: 1));

    if (token.isCancelled) {
      throw Exception("Операция прервана");
    }
  }
}

Future<void> saveProfileWithTimeout() async {
  final token = CancelToken();

  try {
    await Future.any([
      saveProfile(token),
      Future.delayed(const Duration(seconds: 3), () {
        token.cancel();
        throw TimeoutException("Превышено время сохранения профиля");
      }),
    ]);
  } catch (e) {
    print(e);
  }
}

class AsyncQueue {
  Future<void> _tail = Future.value();

  Future<T> add<T>(Future<T> Function() task) {
    final completer = Completer<T>();

    _tail = _tail.then((_) async {
      try {
        completer.complete(await task());
      } catch (e, stackTrace) {
        completer.completeError(e, stackTrace);
      }
    });

    return completer.future;
  }
}

Future<int> benchmark(Future Function() task) async {
  Stopwatch stopwatch = Stopwatch()..start();

  await task();

  stopwatch.stop();

  return stopwatch.elapsedMilliseconds;
}

class AppConfig {
  final Map<String, dynamic> data;

  AppConfig._(this.data);

  static Future<AppConfig>? _instance;

  static Future<AppConfig> getInstance() {
    return _instance ??= _load();
  }

  static Future<AppConfig> _load() async {
    await Future.delayed(const Duration(seconds: 1));

    return AppConfig._({
      "appName": "MyApplication",
      "version": "1.0.0",
    });
  }
}

Future<String> readLocalDatabase() async {
  await Future.delayed(const Duration(milliseconds: 200));
  return "Локальные данные";
}

Future<String> fetchNetworkData() async {
  await Future.delayed(const Duration(seconds: 1));
  return "Сетевые данные";
}

Future<String> loadCache() async {
  Future<String> localData = readLocalDatabase();

  return fetchNetworkData().timeout(
    const Duration(milliseconds: 300),
    onTimeout: () => localData,
  );
}

class Debouncer {
  final Duration delay;
  final void Function(String) onSearch;

  Timer? _timer;

  Debouncer({
    required this.delay,
    required this.onSearch,
  });

  void search(String query) {
    _timer?.cancel();

    _timer = Timer(delay, () {
      onSearch(query);
    });
  }

  void dispose() {
    _timer?.cancel();
  }
}
Блок 2: Потоковые данные, датчики и Stream API

    Слушатель котировок криптовалют: Смоделируйте WebSocket-поток, передающий случайную цену биткоина раз в 200 мс, и отфильтруйте всплески более 5%.
    Стриминг прогресса скачивания файла: Напишите асинхронный генератор async*, передающий процент загрузки от 0% до 100% с шагом в 10% каждые 300 мс.
    Объединение двух потоков (Merge): Создайте функцию, объединяющую события из потока геолокации и потока компаса в единый поток навигации.
    Дросселирование потока (Throttle): Реализуйте оператор, пропускающий не более одного клика по кнопке в секунду из непрерывного потока нажатий.
    Поток шагов акселерометра: Сгенерируйте поток показаний ускорения по трем осям, подсчитывающий число шагов при превышении порогового значения.
    Автоматическое переподключение потока (Reconnection): Напишите слушатель стрима чата, который при возникновении ошибки автоматически пересоздает подписку через 2 секунды.
    Буферизация потока пачками (Buffer / Chunk): Накапливайте события кликов в список и отдавайте их пачками по 5 штук или раз в 3 секунды.
    Поток обратного отсчета для OTP-кода: Создайте стрим таймера на 60 секунд для экрана ввода SMS с уведомлением о завершении срока жизни кода.
    Потоковый калькулятор скользящего среднего: Спроектируйте стрим, принимающий числа и отдающий скользящее среднее значение последних 5 измерений.
    Преобразование Stream в Future: Извлеките последнее значение из конечного потока с помощью метода stream.last.

import 'dart:async';
import 'dart:math';

// Блок 2. Потоковые данные, датчики и Stream API

Stream<double> bitcoinPrices() async* {
  final random = Random();
  double price = 90000;

  while (true) {
    await Future.delayed(const Duration(milliseconds: 200));

    price += (random.nextDouble() - 0.5) * 1000;

    yield price;
  }
}

Stream<double> filterPriceSpikes(Stream<double> stream) async* {
  double? previousPrice;

  await for (final price in stream) {
    if (previousPrice == null ||
        ((price - previousPrice).abs() / previousPrice) <= 0.05) {
      yield price;
    }

    previousPrice = price;
  }
}

Stream<int> downloadProgress() async* {
  for (int i = 0; i <= 100; i += 10) {
    await Future.delayed(const Duration(milliseconds: 300));
    yield i;
  }
}

Stream<String> mergeNavigationStreams(
  Stream<String> locationStream,
  Stream<String> compassStream,
) {
  return StreamGroup.merge([
    locationStream,
    compassStream,
  ]);
}

Stream<T> throttle<T>(Stream<T> stream) async* {
  DateTime? lastEvent;

  await for (final event in stream) {
    DateTime now = DateTime.now();

    if (lastEvent == null ||
        now.difference(lastEvent).inMilliseconds >= 1000) {
      lastEvent = now;
      yield event;
    }
  }
}

Stream<(double, double, double)> accelerometer() async* {
  final random = Random();

  while (true) {
    await Future.delayed(const Duration(milliseconds: 200));

    yield (
      random.nextDouble() * 3,
      random.nextDouble() * 3,
      random.nextDouble() * 3,
    );
  }
}

Stream<int> countSteps(
  Stream<(double, double, double)> stream,
) async* {
  int steps = 0;

  await for (final acceleration in stream) {
    double magnitude = sqrt(
      acceleration.$1 * acceleration.$1 +
          acceleration.$2 * acceleration.$2 +
          acceleration.$3 * acceleration.$3,
    );

    if (magnitude > 2.5) {
      steps++;
      yield steps;
    }
  }
}

Stream<String> createChatStream() async* {
  await Future.delayed(const Duration(seconds: 1));
  yield "Новое сообщение";
  throw Exception("Соединение потеряно");
}

void listenToChat() {
  late StreamSubscription<String> subscription;

  void connect() {
    subscription = createChatStream().listen(
      (message) => print(message),
      onError: (error) async {
        await Future.delayed(const Duration(seconds: 2));
        connect();
      },
    );
  }

  connect();
}

Stream<List<T>> bufferClicks<T>(Stream<T> stream) async* {
  final buffer = <T>[];
  final controller = StreamController<List<T>>();

  Timer? timer;

  void flush() {
    if (buffer.isNotEmpty) {
      controller.add(List<T>.from(buffer));
      buffer.clear();
    }
  }

  timer = Timer.periodic(
    const Duration(seconds: 3),
    (_) => flush(),
  );

  final subscription = stream.listen(
    (event) {
      buffer.add(event);

      if (buffer.length >= 5) {
        flush();
      }
    },
    onError: controller.addError,
    onDone: () {
      flush();
      timer?.cancel();
      controller.close();
    },
  );

  yield* controller.stream;

  await subscription.cancel();
}

Stream<int> otpCountdown() async* {
  for (int i = 60; i >= 0; i--) {
    await Future.delayed(const Duration(seconds: 1));
    yield i;
  }

  print("Срок действия OTP-кода истек");
}

Stream<double> movingAverage(Stream<double> stream) async* {
  final values = <double>[];

  await for (final value in stream) {
    values.add(value);

    if (values.length > 5) {
      values.removeAt(0);
    }

    double average =
        values.reduce((a, b) => a + b) / values.length;

    yield average;
  }
}

Future<void> getLastValue() async {
  Stream<int> stream = Stream.fromIterable([10, 20, 30, 40, 50]);

  int lastValue = await stream.last;

  print(lastValue);
}
лок 3: Реактивные шины событий и StreamController

    Шина глобальных событий (EventBus): Создайте синглтон EventBus на базе широковещательного StreamController для обмена событиями между несвязанными экранами.
    Реактивная корзина интернет-магазина: Реализуйте класс CartStore, транслирующий в стрим текущее количество товаров и итоговую сумму при вызовах add() и remove().
    Счетчик непрочитанных сообщений: Спроектируйте контроллер бейджа уведомлений, увеличивающий счетчик при новых событиях и сбрасывающий его в ноль при прочтении.
    Управление состоянием аудиоплеера: Реализуйте AudioPlayerBloc со стримом состояний: PlayerStopped, PlayerPlaying(track), PlayerPaused.
    Контроллер бесконечной ленты (Pagination Stream): Спроектируйте стрим, подгружающий следующую страницу новостей при передаче события LoadNextPage.
    Реактивный переключатель темной темы: Создайте менеджер темы приложения, отдающий ThemeMode.dark или ThemeMode.light через BehaviorSubject-подобный контроллер.
    Слушатель сетевого статуса (Connectivity Stream): Контроллер, транслирующий изменения подключения (WiFi, Cellular, None) во все подписанные модули.
    Защита контроллера от утечек памяти (Auto-Dispose): Напишите класс с механизмом проверки наличия подписчиков и автоматическим вызовом close() при обнулении слушателей.
    Валидация формы регистрации на стримах: Создайте два потока для полей Email и Пароль, объедините их через Rx.combineLatest-подобную логику и отдавайте флаг активности кнопки isSubmitEnabled.
    Потоковый логгер крашей: Направляйте все неотловленные ошибки приложения через StreamController в фоновый модуль отправки аналитики

import 'dart:async';

// Блок 3. Реактивные шины событий и StreamController

class EventBus {
  EventBus._();

  static final EventBus instance = EventBus._();

  final StreamController<Object> _controller =
      StreamController<Object>.broadcast();

  Stream<Object> get stream => _controller.stream;

  void fire(Object event) {
    _controller.sink.add(event);
  }

  void dispose() {
    _controller.close();
  }
}

class CartState {
  final int itemCount;
  final double totalPrice;

  CartState(this.itemCount, this.totalPrice);
}

class CartStore {
  final StreamController<CartState> _controller =
      StreamController<CartState>.broadcast();

  final Map<String, double> _products = {};

  Stream<CartState> get stream => _controller.stream;

  void add(String name, double price) {
    _products[name] = (_products[name] ?? 0) + price;
    _update();
  }

  void remove(String name) {
    _products.remove(name);
    _update();
  }

  void _update() {
    double total = _products.values.fold(0, (a, b) => a + b);

    _controller.add(
      CartState(_products.length, total),
    );
  }

  void dispose() {
    _controller.close();
  }
}

class NotificationBadge {
  final StreamController<int> _controller =
      StreamController<int>.broadcast();

  int _count = 0;

  Stream<int> get stream => _controller.stream;

  void increment() {
    _count++;
    _controller.add(_count);
  }

  void reset() {
    _count = 0;
    _controller.add(_count);
  }

  void dispose() {
    _controller.close();
  }
}

abstract class PlayerState {}

class PlayerStopped extends PlayerState {}

class PlayerPlaying extends PlayerState {
  final String track;

  PlayerPlaying(this.track);
}

class PlayerPaused extends PlayerState {}

class AudioPlayerBloc {
  final StreamController<PlayerState> _controller =
      StreamController<PlayerState>.broadcast();

  Stream<PlayerState> get stream => _controller.stream;

  void play(String track) {
    _controller.add(PlayerPlaying(track));
  }

  void pause() {
    _controller.add(PlayerPaused());
  }

  void stop() {
    _controller.add(PlayerStopped());
  }

  void dispose() {
    _controller.close();
  }
}

class LoadNextPage {}

class PaginationController {
  final StreamController<LoadNextPage> _requestController =
      StreamController<LoadNextPage>();

  final StreamController<List<String>> _newsController =
      StreamController<List<String>>.broadcast();

  int _page = 0;

  Stream<List<String>> get stream => _newsController.stream;

  PaginationController() {
    _requestController.stream.listen((event) async {
      _page++;

      await Future.delayed(const Duration(seconds: 1));

      _newsController.add([
        "Новость $_page.1",
        "Новость $_page.2",
        "Новость $_page.3",
      ]);
    });
  }

  void loadNextPage() {
    _requestController.add(LoadNextPage());
  }

  void dispose() {
    _requestController.close();
    _newsController.close();
  }
}

enum ThemeMode {
  dark,
  light,
}

class ThemeManager {
  final StreamController<ThemeMode> _controller =
      StreamController<ThemeMode>.broadcast();

  ThemeMode _theme = ThemeMode.light;

  Stream<ThemeMode> get stream => _controller.stream;

  ThemeMode get currentTheme => _theme;

  void toggleTheme() {
    _theme = _theme == ThemeMode.light
        ? ThemeMode.dark
        : ThemeMode.light;

    _controller.add(_theme);
  }

  void dispose() {
    _controller.close();
  }
}

enum ConnectivityStatus {
  WiFi,
  Cellular,
  None,
}

class ConnectivityController {
  final StreamController<ConnectivityStatus> _controller =
      StreamController<ConnectivityStatus>.broadcast();

  Stream<ConnectivityStatus> get stream => _controller.stream;

  void updateStatus(ConnectivityStatus status) {
    _controller.add(status);
  }

  void dispose() {
    _controller.close();
  }
}

class AutoDisposeController<T> {
  late final StreamController<T> _controller;

  int _listeners = 0;

  AutoDisposeController() {
    _controller = StreamController<T>(
      onListen: () {
        _listeners++;
      },
      onCancel: () {
        _listeners--;

        if (_listeners <= 0) {
          close();
        }
      },
    );
  }

  Stream<T> get stream => _controller.stream;

  void add(T value) {
    if (!_controller.isClosed) {
      _controller.add(value);
    }
  }

  void close() {
    if (!_controller.isClosed) {
      _controller.close();
    }
  }
}

class RegistrationController {
  final StreamController<String> _emailController =
      StreamController<String>.broadcast();

  final StreamController<String> _passwordController =
      StreamController<String>.broadcast();

  final StreamController<bool> _submitController =
      StreamController<bool>.broadcast();

  String _email = "";
  String _password = "";

  Stream<bool> get isSubmitEnabled => _submitController.stream;

  RegistrationController() {
    _emailController.stream.listen((email) {
      _email = email;
      _validate();
    });

    _passwordController.stream.listen((password) {
      _password = password;
      _validate();
    });
  }

  void updateEmail(String email) {
    _emailController.add(email);
  }

  void updatePassword(String password) {
    _passwordController.add(password);
  }

  void _validate() {
    bool isValid = _email.contains("@") && _password.length >= 8;

    _submitController.add(isValid);
  }

  void dispose() {
    _emailController.close();
    _passwordController.close();
    _submitController.close();
  }
}

class CrashLogger {
  final StreamController<Object> _controller =
      StreamController<Object>.broadcast();

  Stream<Object> get stream => _controller.stream;

  CrashLogger() {
    stream.listen((error) {
      sendAnalytics(error);
    });
  }

  void logError(Object error) {
    _controller.add(error);
  }

  Future<void> sendAnalytics(Object error) async {
    await Future.delayed(const Duration(seconds: 1));
    print("Ошибка отправлена в аналитику: $error");
  }

  void dispose() {
    _controller.close();
  }
}
Блок 4: Изоляты и ресурсоемкие параллельные задачи

    Фоновая генерация миниатюр (Thumbnail Generator): Обработайте массив байтов изображения в Isolate.run(), изменив его размер и вернув сжатый массив.
    Шифрование базы заметок: Выполните шифрование большого текстового файла по алгоритму XOR / AES в параллельном изоляте без просадки FPS.
    Полнотекстовый поиск по 100 000 документов: Передайте поисковый запрос и список документов в фоновый изолят, вернув 10 наиболее релевантных совпадений.
    Парсинг тяжелого GeoJSON карты города: Распарсите координаты полигонов районов города в фоновом потоке памяти и верните готовые структуры данных.
    Сжатие логов в ZIP-архив: Смоделируйте упаковку текстовых журналов в сжатый архив через изолят перед отправкой в техподдержку.
    Двусторонний пул воркеров (Worker Pool): Создайте менеджер из двух постоянно работающих изолятов, распределяющий между ними математические задачи.
    Генерация PDF-отчета: Сформируйте текстовую структуру финансового отчета за год в параллельном изоляте.
    Сортировка массива из 1 000 000 чисел: Сравните время сортировки массива в главном потоке и в отдельном изоляте.
    Конвертер аудио-семплов: Смоделируйте преобразование массива сырых аудио-семплов в изоляте.
    Безопасная передача больших объектов (TransferableTypedData): Передайте большой бинарный буфер между изолятами без затратного копирования памяти.

import 'dart:async';
import 'dart:convert';
import 'dart:isolate';
import 'dart:math';
import 'dart:typed_data';
import 'package:archive/archive.dart';

// Блок 4. Изоляты и ресурсоемкие параллельные задачи

Uint8List generateThumbnail(Uint8List imageBytes) {
  return Isolate.run(() {
    int newLength = imageBytes.length ~/ 4;

    return Uint8List.fromList(
      imageBytes.sublist(0, newLength),
    );
  }) as Uint8List;
}

Uint8List encryptNotes(String text) {
  final bytes = utf8.encode(text);

  return Uint8List.fromList(
    bytes.map((byte) => byte ^ 123).toList(),
  );
}

Future<Uint8List> encryptNotesInIsolate(String text) {
  return Isolate.run(() => encryptNotes(text));
}

class SearchResult {
  final String document;
  final int relevance;

  SearchResult(this.document, this.relevance);
}

List<SearchResult> searchDocuments(
  List<String> documents,
  String query,
) {
  List<SearchResult> results = [];

  for (String document in documents) {
    int relevance = query.allMatches(
      document.toLowerCase(),
    ).length;

    if (relevance > 0) {
      results.add(SearchResult(document, relevance));
    }
  }

  results.sort(
    (a, b) => b.relevance.compareTo(a.relevance),
  );

  return results.take(10).toList();
}

Future<List<SearchResult>> fullTextSearch(
  List<String> documents,
  String query,
) {
  return Isolate.run(
    () => searchDocuments(documents, query),
  );
}

List<List<List<double>>> parseGeoJson(String jsonString) {
  final data = jsonDecode(jsonString);

  List<List<List<double>>> polygons = [];

  for (var feature in data['features']) {
    var coordinates =
        feature['geometry']['coordinates'];

    for (var polygon in coordinates) {
      List<List<double>> points = [];

      for (var point in polygon) {
        points.add([
          point[0].toDouble(),
          point[1].toDouble(),
        ]);
      }

      polygons.add(points);
    }
  }

  return polygons;
}

Future<List<List<List<double>>>> parseGeoJsonInIsolate(
  String jsonString,
) {
  return Isolate.run(() => parseGeoJson(jsonString));
}

Future<List<int>> compressLogs(List<String> logs) {
  return Isolate.run(() {
    String text = logs.join('\n');
    List<int> bytes = utf8.encode(text);

    return GZipEncoder().encode(bytes);
  });
}

void workerEntryPoint(SendPort mainPort) {
  final receivePort = ReceivePort();

  mainPort.send(receivePort.sendPort);

  receivePort.listen((message) {
    int id = message[0];
    int number = message[1];
    SendPort replyPort = message[2];

    int result = number * number;

    replyPort.send([id, result]);
  });
}

class WorkerPool {
  final List<SendPort> _workers = [];
  final ReceivePort _receivePort = ReceivePort();

  int _currentWorker = 0;
  int _taskId = 0;

  final Map<int, Completer<int>> _tasks = {};

  Future<void> start() async {
    _receivePort.listen((message) {
      int id = message[0];
      int result = message[1];

      _tasks.remove(id)?.complete(result);
    });

    for (int i = 0; i < 2; i++) {
      final port = ReceivePort();

      await Isolate.spawn(
        workerEntryPoint,
        port.sendPort,
      );

      SendPort worker = await port.first as SendPort;

      _workers.add(worker);
    }
  }

  Future<int> execute(int number) {
    int id = _taskId++;
    final completer = Completer<int>();

    _tasks[id] = completer;

    _workers[_currentWorker].send([
      id,
      number,
      _receivePort.sendPort,
    ]);

    _currentWorker = (_currentWorker + 1) % 2;

    return completer.future;
  }

  void dispose() {
    _receivePort.close();
  }
}

String generateFinancialReport(Map<String, double> finances) {
  StringBuffer report = StringBuffer();

  report.writeln("Финансовый отчет за год");
  report.writeln("

---

---

---

---

---

---

---

---

");

  finances.forEach((month, amount) {
    report.writeln("$month: $amount");
  });

  double total = finances.values.fold(
    0,
    (sum, value) => sum + value,
  );

  report.writeln("

---

---

---

---

---

---

---

---

");
  report.writeln("Итого: $total");

  return report.toString();
}

Future<String> generatePdfReport(
  Map<String, double> finances,
) {
  return Isolate.run(
    () => generateFinancialReport(finances),
  );
}

Future<void> compareSorting() async {
  List<int> numbers = List.generate(
    1000000,
    (_) => Random().nextInt(1000000),
  );

  Stopwatch stopwatch = Stopwatch()..start();

  List<int> syncNumbers = List.from(numbers);
  syncNumbers.sort();

  stopwatch.stop();

  print(
    "Главный поток: ${stopwatch.elapsedMilliseconds} мс",
  );

  stopwatch.reset();
  stopwatch.start();

  List<int> isolateNumbers = await Isolate.run(() {
    List<int> sorted = List.from(numbers);
    sorted.sort();
    return sorted;
  });

  stopwatch.stop();

  print(
    "Изолят: ${stopwatch.elapsedMilliseconds} мс",
  );
}

Future<Float32List> convertAudioSamples(
  Float32List samples,
) {
  return Isolate.run(() {
    return Float32List.fromList(
      samples.map((sample) => sample / 32768.0).toList(),
    );
  });
}

Future<TransferableTypedData> transferLargeBuffer(
  TransferableTypedData data,
) async {
  return await Isolate.run(() {
    return data;
  });
}

Блок 5: Сложные асинхронные архитектуры и отказоустойчивость

    Отмена сетевой операции через CancelToken: Реализуйте класс токена отмены, выбрасывающий CancellationException при прерывании пользователем ухода со страницы.
    Репликация базы данных с разрешением конфликтов: Синхронизируйте две асинхронные очереди изменений с временными метками (Timestamps).
    Асинхронная очередь сохранения офлайн-заказов: Сохраняйте оформленные без интернета заказы в локальную очередь и автоматически отправляйте их по одному при восстановлении сети.
    Паттерн Circuit Breaker: Реализуйте защиту от падения сервера: если 5 запросов подряд завершились ошибкой, переведите сервис в режим «Разрыв цепи» на 30 секунд.
    Асинхронный семафор (Rate Limiter): Ограничьте число одновременных сетевых обращений к API до 3 параллельных потоков.
    Гонка состояний при быстром вводе (Race Condition Fix): Защитите поисковую строку: гарантируйте, что старый медленный ответ сервера не перезапишет новый быстрый ответ.
    Слушатель жизненного цикла приложения: Свяжите события сворачивания приложения с мгновенной приостановкой всех активных StreamSubscription.
    Конечный автомат асинхронной синхронизации: Реализуйте переключение стейтов Idle -> Syncing(progress) -> Success -> Error с защитой от повторного запуска во время синхронизации.
    Многоуровневый кэш (Memory -> Disk -> Network): Спроектируйте функцию, последовательно проверяющую быструю оперативную память, затем локальный диск и только потом идущую в сеть.
    Итоговый реактивный модуль финансового терминала (Production Architecture):
        Подключите непрерывный поток биржевых котировок через WebSocket-имитатор.
        Передайте расчет скользящей волатильности в параллельный изолят (Isolate.run).
        Ограничьте частоту обновления графического экрана до 60 кадров/сек с помощью троттлинга.
        Реализуйте автоматическое восстановление соединения при обрыве и безопасный вызов dispose() для всех контроллеров и подписок.

import 'dart:async';
import 'dart:math';
import 'dart:collection';
import 'dart:isolate';

// Блок 5. Сложные асинхронные архитектуры и отказоустойчивость

class CancellationException implements Exception {
  final String message;

  CancellationException(this.message);
}

class CancelToken {
  bool _isCancelled = false;

  bool get isCancelled => _isCancelled;

  void cancel() {
    _isCancelled = true;
  }

  void throwIfCancelled() {
    if (_isCancelled) {
      throw CancellationException("Операция отменена");
    }
  }
}

Future<String> cancellableRequest(CancelToken token) async {
  for (int i = 0; i < 10; i++) {
    await Future.delayed(const Duration(milliseconds: 500));

    token.throwIfCancelled();
  }

  return "Запрос завершен";
}

class DatabaseChange {
  final String key;
  final String value;
  final DateTime timestamp;

  DatabaseChange(this.key, this.value, this.timestamp);
}

class DatabaseReplication {
  final Map<String, DatabaseChange> databaseA = {};
  final Map<String, DatabaseChange> databaseB = {};

  Future<void> synchronize() async {
    await Future.wait([
      _replicate(databaseA, databaseB),
      _replicate(databaseB, databaseA),
    ]);
  }

  Future<void> _replicate(
    Map<String, DatabaseChange> source,
    Map<String, DatabaseChange> target,
  ) async {
    await Future.delayed(const Duration(milliseconds: 100));

    for (var entry in source.entries) {
      var existing = target[entry.key];

      if (existing == null ||
          entry.value.timestamp.isAfter(existing.timestamp)) {
        target[entry.key] = entry.value;
      }
    }
  }
}

class OfflineOrderQueue {
  final Queue<String> _orders = Queue<String>();

  bool _isOnline = false;
  bool _isSending = false;

  void addOrder(String order) {
    _orders.add(order);
    _processQueue();
  }

  void setOnline(bool online) {
    _isOnline = online;
    _processQueue();
  }

  Future<void> _processQueue() async {
    if (!_isOnline || _isSending) return;

    _isSending = true;

    try {
      while (_orders.isNotEmpty && _isOnline) {
        String order = _orders.first;

        try {
          await _sendOrder(order);
          _orders.removeFirst();
        } catch (e) {
          break;
        }
      }
    } finally {
      _isSending = false;
    }
  }

  Future<void> _sendOrder(String order) async {
    await Future.delayed(const Duration(seconds: 1));
    print("Заказ отправлен: $order");
  }
}

class CircuitBreaker {
  int _failures = 0;
  bool _isOpen = false;
  DateTime? _openedAt;

  Future<T> execute<T>(Future<T> Function() request) async {
    if (_isOpen) {
      if (DateTime.now().difference(_openedAt!) <
          const Duration(seconds: 30)) {
        throw Exception("Circuit Breaker: Разрыв цепи");
      }

      _isOpen = false;
      _failures = 0;
    }

    try {
      T result = await request();
      _failures = 0;
      return result;
    } catch (e) {
      _failures++;

      if (_failures >= 5) {
        _isOpen = true;
        _openedAt = DateTime.now();
      }

      rethrow;
    }
  }
}

class AsyncSemaphore {
  final int maxConcurrent;
  int _active = 0;

  final Queue<Completer<void>> _waiting = Queue();

  AsyncSemaphore(this.maxConcurrent);

  Future<T> execute<T>(Future<T> Function() task) async {
    if (_active >= maxConcurrent) {
      final completer = Completer<void>();
      _waiting.add(completer);
      await completer.future;
    }

    _active++;

    try {
      return await task();
    } finally {
      _active--;

      if (_waiting.isNotEmpty) {
        _waiting.removeFirst().complete();
      }
    }
  }
}

class SearchManager {
  int _requestId = 0;

  Future<void> search(
    String query,
    void Function(String) onResult,
  ) async {
    int currentId = ++_requestId;

    String result = await _serverSearch(query);

    if (currentId == _requestId) {
      onResult(result);
    }
  }

  Future<String> _serverSearch(String query) async {
    await Future.delayed(
      Duration(milliseconds: query.length * 300),
    );

    return "Результат поиска: $query";
  }
}

class AppLifecycleManager {
  final List<StreamSubscription<dynamic>> _subscriptions = [];

  void addSubscription(StreamSubscription<dynamic> subscription) {
    _subscriptions.add(subscription);
  }

  Future<void> onAppPaused() async {
    for (var subscription in _subscriptions) {
      await subscription.pause();
    }
  }

  void onAppResumed() {
    for (var subscription in _subscriptions) {
      subscription.resume();
    }
  }

  Future<void> dispose() async {
    for (var subscription in _subscriptions) {
      await subscription.cancel();
    }

    _subscriptions.clear();
  }
}

abstract class SyncState {}

class Idle extends SyncState {}

class Syncing extends SyncState {
  final double progress;

  Syncing(this.progress);
}

class Success extends SyncState {}

class Error extends SyncState {
  final Object error;

  Error(this.error);
}

class SyncManager {
  final StreamController<SyncState> _controller =
      StreamController<SyncState>.broadcast();

  bool _isSyncing = false;

  Stream<SyncState> get stream => _controller.stream;

  Future<void> synchronize() async {
    if (_isSyncing) return;

    _isSyncing = true;
    _controller.add(Syncing(0));

    try {
      for (int i = 1; i <= 10; i++) {
        await Future.delayed(const Duration(milliseconds: 300));
        _controller.add(Syncing(i / 10));
      }

      _controller.add(Success());
    } catch (e) {
      _controller.add(Error(e));
    } finally {
      _isSyncing = false;
    }
  }

  void dispose() {
    _controller.close();
  }
}

class MultiLevelCache<T> {
  final Map<String, T> memoryCache = {};

  final Future<T?> Function(String) readDisk;
  final Future<void> Function(String, T) writeDisk;
  final Future<T> Function(String) fetchNetwork;

  MultiLevelCache({
    required this.readDisk,
    required this.writeDisk,
    required this.fetchNetwork,
  });

  Future<T> get(String key) async {
    if (memoryCache.containsKey(key)) {
      return memoryCache[key] as T;
    }

    T? diskData = await readDisk(key);

    if (diskData != null) {
      memoryCache[key] = diskData;
      return diskData;
    }

    T networkData = await fetchNetwork(key);

    memoryCache[key] = networkData;
    await writeDisk(key, networkData);

    return networkData;
  }
}

class FinancialTerminal {
  final StreamController<double> _quoteController =
      StreamController<double>();

  final StreamController<double> _screenController =
      StreamController<double>.broadcast();

  StreamSubscription<double>? _quoteSubscription;

  Timer? _reconnectTimer;

  bool _disposed = false;
  double _previousPrice = 0;

  Stream<double> get screenStream => _screenController.stream;

  Future<void> connect() async {
    if (_disposed) return;

    try {
      _quoteSubscription = _webSocketSimulator().listen(
        (price) async {
          if (_disposed) return;

          if (_previousPrice != 0) {
            double volatility = await Isolate.run(() {
              return (price - _previousPrice).abs() /
                  _previousPrice;
            });

            if (!_disposed) {
              _screenController.add(volatility);
            }
          }

          _previousPrice = price;
        },
        onError: (error) {
          _scheduleReconnect();
        },
        onDone: () {
          _scheduleReconnect();
        },
      );
    } catch (e) {
      _scheduleReconnect();
    }
  }

  Stream<double> _webSocketSimulator() async* {
    final random = Random();
    double price = 90000;

    for (int i = 0; i < 100; i++) {
      if (_disposed) return;

      await Future.delayed(const Duration(milliseconds: 200));

      price += (random.nextDouble() - 0.5) * 1000;

      yield price;
    }
  }

  void _scheduleReconnect() {
    if (_disposed || _reconnectTimer != null) return;

    _reconnectTimer = Timer(
      const Duration(seconds: 2),
      () {
        _reconnectTimer = null;
        connect();
      },
    );
  }

  Stream<double> throttle60FPS(Stream<double> stream) async* {
    DateTime? lastUpdate;

    await for (final value in stream) {
      DateTime now = DateTime.now();

      if (lastUpdate == null ||
          now.difference(lastUpdate).inMilliseconds >= 16) {
        lastUpdate = now;
        yield value;
      }
    }
  }

  Future<void> dispose() async {
    if (_disposed) return;

    _disposed = true;

    _reconnectTimer?.cancel();

    await _quoteSubscription?.cancel();

    await _quoteController.close();
    await _screenController.close();
  }
}
