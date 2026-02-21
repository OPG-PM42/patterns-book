# Декомпозиция, Инстанцирование, Инкапсуляция, Агрегация, Композиция, Ассоциация, Делегирование

## Обзор

Эта лекция посвящена фундаментальным понятиям объектно-ориентированного проектирования, которые помогают разделять абстракции на части, делая их менее сложными и более гибкими в компоновке. Мы рассмотрим различные способы связывания объектов и управления их зависимостями, что является ключевым навыком для создания масштабируемых и поддерживаемых приложений.

**Основные темы:**
- Декомпозиция сложных абстракций
- Способы связывания объектов (ассоциация, агрегация, композиция)
- Инстанцирование и владение объектами
- Делегирование ответственности
- Инкапсуляция внутренней структуры

---

## 1. Инстанцирование и Ассоциация

### Концепция

**Инстанцирование** — это процесс создания экземпляров объектов из классов или фабричных функций. При инстанцировании мы связываем различные абстракции друг с другом.

**Ассоциация** — это самый слабый тип связи между объектами, когда они знают друг о друге, но не владеют друг другом. Это наименее желательный способ связывания объектов.

### Проблема внешней ассоциации

Когда мы связываем объекты извне, мы создаем жесткую связь и нарушаем инкапсуляцию.

#### Пример: Плохая практика (внешняя ассоциация)

```javascript
// Определение класса Logger
class Logger {
  constructor() {
    this.stream = null; // Стрим будет присвоен извне
  }

  log(message) {
    if (this.stream) {
      this.stream.write(message + '\n');
    }
  }
}

// Использование - ПЛОХОЙ СПОСОБ
const fs = require('fs');
const logger = new Logger();

// Внешняя ассоциация - присваивание свойства извне
logger.stream = fs.createWriteStream('./app.log');

logger.log('Сообщение в лог');
```

**Проблемы этого подхода:**
- Нарушается инкапсуляция — внутренняя структура Logger доступна извне
- Используется практически примесь (mixin), что создает неявные зависимости
- Сложно контролировать валидность состояния объекта
- Нет гарантий, что stream будет установлен корректно

---

## 2. Агрегация

### Концепция

**Агрегация** — это способ связывания объектов, при котором один объект содержит ссылку на другой, но **не владеет** им и не отвечает за его жизненный цикл. Объект передается извне (обычно через конструктор или сеттер) и может существовать независимо.

### Характеристики агрегации

- Объект получает зависимость извне
- Не отвечает за создание зависимости
- Не отвечает за уничтожение зависимости
- Зависимость может быть разделена между несколькими объектами
- Обеспечивает слабую связанность (loose coupling)

### Пример: Агрегация с использованием классов

```javascript
const fs = require('fs');

// Logger получает stream извне и использует его
class Logger {
  constructor(stream) {
    this.stream = stream; // Сохраняем ссылку на переданный stream
  }

  log(message) {
    this.stream.write(message + '\n');
  }
}

// Использование
const outputStream = fs.createWriteStream('./app.log');
const logger = new Logger(outputStream);

logger.log('Первое сообщение');
logger.log('Второе сообщение');

// Logger не отвечает за закрытие stream
// Это должно быть сделано внешним кодом
outputStream.end();
```

**Ключевые моменты:**
- `Logger` знает о интерфейсе `stream` (методы `write`, `end` и т.д.)
- `stream` ничего не знает о `Logger`
- `Logger` **не является владельцем** stream
- Все операции жизненного цикла stream (открытие, закрытие, удаление файла) выполняются внешним кодом
- `Logger` имеет только одну обязанность — писать в переданный stream

### Пример: Агрегация через замыкание

```javascript
const fs = require('fs');

// Фабричная функция для создания logger
function createLogger(stream) {
  // stream попадает в замыкание
  return {
    log(message) {
      stream.write(message + '\n');
    }
  };
}

// Использование
const outputStream = fs.createWriteStream('./app.log');
const logger = createLogger(outputStream);

logger.log('Сообщение через замыкание');

// Каждый вызов createLogger создает новый экземпляр объекта,
// замкнутый на переданный stream
const anotherLogger = createLogger(outputStream);
anotherLogger.log('Другое сообщение');

outputStream.end();
```

**Преимущества функционального подхода:**
- Нет необходимости в ключевом слове `class`
- Stream естественно скрыт в замыкании
- Более функциональный стиль кода
- Легко тестировать, передавая mock-объекты

---

## 3. Композиция

### Концепция

**Композиция** — это способ связывания объектов, при котором один объект **создает** и **владеет** другим объектом. Владелец отвечает за полный жизненный цикл вложенного объекта, включая его создание и уничтожение.

### Характеристики композиции

- Объект создает свои зависимости внутри себя
- Полностью отвечает за жизненный цикл зависимостей
- Зависимость не может существовать независимо от владельца
- Обеспечивает сильную связанность, но упрощает управление

### Пример: Композиция с использованием классов

```javascript
const fs = require('fs');

class Logger {
  constructor(filename) {
    // Logger САМ создает stream внутри
    this.stream = fs.createWriteStream(filename);
    this.filename = filename;
  }

  log(message) {
    this.stream.write(message + '\n');
  }

  // Logger отвечает за уничтожение stream
  close() {
    this.stream.end();
  }

  // Дополнительные методы управления
  destroy() {
    this.close();
    fs.unlinkSync(this.filename); // Удаление файла
  }
}

// Использование
const logger = new Logger('./app.log');
logger.log('Сообщение');
logger.log('Другое сообщение');

// Logger отвечает за закрытие stream
logger.close();
```

**Преимущества композиции:**
- Полный контроль над зависимостями
- Гарантия корректного создания и уничтожения
- Упрощение использования — пользователю не нужно создавать stream

**Недостатки:**
- Меньшая гибкость
- Сложнее тестировать (нужно мокировать файловую систему)
- Жесткая связь с конкретной реализацией

### Пример: Композиция через замыкание

```javascript
const fs = require('fs');

function createLogger(filename) {
  // Stream создается ВНУТРИ функции
  const stream = fs.createWriteStream(filename);

  // Возвращаемый объект замкнут на этот stream
  return {
    log(message) {
      stream.write(message + '\n');
    },

    close() {
      stream.end();
    }
  };
}

// Использование
const logger = createLogger('./app.log');
logger.log('Сообщение через композицию');
logger.close();
```

**Разница между агрегацией и композицией через замыкания:**

| Агрегация | Композиция |
|-----------|------------|
| `createLogger(stream)` — stream передается | `createLogger(filename)` — stream создается внутри |
| Объект не владеет зависимостью | Объект владеет зависимостью |
| Не отвечает за уничтожение | Отвечает за уничтожение |

---

## 4. Функциональная композиция

### Концепция

**Функциональная композиция** — это объединение нескольких функций в одну цепочку обработки данных. Это отличается от композиции классов, но может сочетаться с агрегацией абстракций.

### Пример: Функциональная композиция с агрегацией

```javascript
const fs = require('fs');

// Вспомогательная функция compose для композиции функций
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);

// Функция форматирования
const formatMessage = (message) => {
  const timestamp = new Date().toISOString();
  return `[${timestamp}] ${message}`;
};

// Функция записи в stream (каррированная)
const writeToStream = (stream) => (message) => {
  stream.write(message + '\n');
  return message;
};

// Использование - АГРЕГАЦИЯ stream
const outputStream = fs.createWriteStream('./app.log');

// Функциональная КОМПОЗИЦИЯ функций + АГРЕГАЦИЯ stream
const log = compose(
  writeToStream(outputStream), // stream передан извне (агрегация)
  formatMessage                 // сначала форматирование
);

// Композицию нужно читать справа налево:
// 1. formatMessage - форматирует сообщение
// 2. writeToStream - записывает в stream

log('Тестовое сообщение');
log('Другое сообщение');

outputStream.end();
```

**Ключевые особенности:**
- **Функциональная композиция** — объединение функций `formatMessage` и `writeToStream`
- **Агрегация абстракций** — stream передается извне, не создается внутри
- Разделение ответственности: форматирование и запись — отдельные функции
- Композиция читается справа налево (или снизу вверх)

### Альтернативная реализация с pipe

```javascript
// pipe читается слева направо (более интуитивно)
const pipe = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);

const log = pipe(
  formatMessage,                // сначала форматирование
  writeToStream(outputStream)   // затем запись
);

log('Сообщение через pipe');
```

---

## 5. Переход от наследования к композиции

### Проблема наследования

Наследование создает жесткую иерархию классов, которая может привести к:
- Сложности изменения базовых классов
- Проблемам множественного наследования
- Жесткой связанности между родителем и потомком
- Сложности понимания глубоких иерархий

### Пример: Использование наследования (не рекомендуется)

```javascript
const EventEmitter = require('events');

// Channel наследует EventEmitter
class Channel extends EventEmitter {
  constructor(name) {
    super();
    this.name = name;
    this.messages = []; // История сообщений
  }

  send(message, author) {
    this.messages.push({ message, author, timestamp: Date.now() });
    this.emit('message', { channel: this.name, message, author });
  }
}

// Application также наследует EventEmitter
class Application extends EventEmitter {
  constructor() {
    super();
    this.channels = new Map(); // Коллекция каналов
  }

  createChannel(name) {
    const channel = new Channel(name);
    this.channels.set(name, channel);

    // Эмитим событие на Application
    this.emit('channel:created', { name });

    return channel;
  }
}

// Использование
const app = new Application();

app.on('channel:created', ({ name }) => {
  console.log(`Создан канал: ${name}`);
});

const general = app.createChannel('general');

general.on('message', ({ channel, message, author }) => {
  console.log(`[${channel}] ${author}: ${message}`);
});

general.send('Привет всем!', 'Alice');
```

**Проблемы этого подхода:**

1. **Жесткая связанность:** И `Channel`, и `Application` жестко привязаны к `EventEmitter`
2. **Сложность иерархии:** При расширении функциональности иерархия становится глубже
3. **Смешение ответственностей:** Channel занимается и хранением сообщений, и эмиттингом событий
4. **Сложность тестирования:** Трудно изолировать функциональность событий от бизнес-логики

### Улучшенный подход: Композиция вместо наследования

```javascript
const EventEmitter = require('events');

// Channel использует КОМПОЗИЦИЮ - создает свой EventEmitter
class Channel {
  constructor(name) {
    this.name = name;
    this.messages = [];
    this.emitter = new EventEmitter(); // Композиция!
  }

  // Делегирование методов EventEmitter
  on(event, handler) {
    this.emitter.on(event, handler);
  }

  emit(event, data) {
    this.emitter.emit(event, data);
  }

  send(message, author) {
    this.messages.push({ message, author, timestamp: Date.now() });
    this.emit('message', { channel: this.name, message, author });
  }
}

// Application также использует КОМПОЗИЦИЮ
class Application {
  constructor() {
    this.channels = new Map();
    this.emitter = new EventEmitter(); // Композиция!
  }

  // Делегирование
  on(event, handler) {
    this.emitter.on(event, handler);
  }

  emit(event, data) {
    this.emitter.emit(event, data);
  }

  createChannel(name) {
    const channel = new Channel(name);
    this.channels.set(name, channel);
    this.emit('channel:created', { name });
    return channel;
  }
}

// Использование - идентично предыдущему примеру
const app = new Application();
app.on('channel:created', ({ name }) => {
  console.log(`Создан канал: ${name}`);
});

const general = app.createChannel('general');
general.on('message', ({ channel, message, author }) => {
  console.log(`[${channel}] ${author}: ${message}`);
});

general.send('Привет через композицию!', 'Bob');
```

**Преимущества композиции:**
- `Channel` и `Application` не зависят от иерархии `EventEmitter`
- Четкое разделение ответственностей
- Гибкость: можно легко заменить `EventEmitter` на другую реализацию
- Лучшая тестируемость: можно мокировать emitter

---

## 6. Делегирование

### Концепция

**Делегирование** — это паттерн, при котором объект передает выполнение определенных операций другому объекту (делегату), скрывая детали реализации за своим интерфейсом.

### Проблема прямого доступа к внутренностям

```javascript
// ПЛОХОЙ ПРИМЕР - прямой доступ к вложенным объектам
class Application {
  constructor() {
    this.channels = new Map();
    this.emitter = new EventEmitter();
  }

  createChannel(name) {
    const channel = new Channel(name);
    this.channels.set(name, channel);

    // Прямой доступ к вложенному объекту - НЕ РЕКОМЕНДУЕТСЯ
    this.emitter.emit('channel:created', { name });

    return channel;
  }
}

// Использование - лазание в "кишки" объекта
const app = new Application();

// Плохо: прямой доступ к внутреннему emitter
app.emitter.on('channel:created', (data) => {
  console.log('Канал создан:', data.name);
});
```

**Проблемы:**
- Нарушение инкапсуляции
- Пользователи класса зависят от внутренней структуры
- Сложно изменить внутреннюю реализацию без breaking changes
- Возможность случайного изменения внутреннего состояния

### Правильное делегирование

```javascript
class Application {
  constructor() {
    this.channels = new Map();
    this.emitter = new EventEmitter(); // Приватная деталь реализации
  }

  // ДЕЛЕГИРОВАНИЕ - предоставляем публичный интерфейс
  on(event, handler) {
    // Делегируем вызов внутреннему emitter
    this.emitter.on(event, handler);
  }

  emit(event, data) {
    // Делегируем вызов внутреннему emitter
    this.emitter.emit(event, data);
  }

  createChannel(name) {
    const channel = new Channel(name);
    this.channels.set(name, channel);

    // Используем публичный интерфейс
    this.emit('channel:created', { name });

    return channel;
  }
}

// ПРАВИЛЬНОЕ использование
const app = new Application();

// Хорошо: используем делегированный метод
app.on('channel:created', (data) => {
  console.log('Канал создан:', data.name);
});
```

### Расширенный пример делегирования с адаптацией интерфейса

```javascript
class Application {
  constructor() {
    this.channels = new Map();
    this.emitter = new EventEmitter();
  }

  // Делегирование с адаптацией интерфейса
  sendMessage(channelName, message, author) {
    const channel = this.channels.get(channelName);

    if (!channel) {
      throw new Error(`Channel ${channelName} not found`);
    }

    // Формируем данные для внутреннего события
    const eventData = {
      channel: channelName,
      message,
      author,
      timestamp: Date.now()
    };

    // Сохраняем в историю канала
    channel.send(message, author);

    // Эмитим событие на уровне Application
    // (адаптируем интерфейс channel для application)
    this.emitter.emit('message', eventData);
  }

  // Делегирование подписки
  onMessage(handler) {
    this.emitter.on('message', handler);
  }

  createChannel(name) {
    const channel = new Channel(name);
    this.channels.set(name, channel);
    this.emitter.emit('channel:created', { name });
    return channel;
  }
}

// Использование
const app = new Application();

// Подписываемся через делегированный метод
app.onMessage(({ channel, message, author, timestamp }) => {
  const date = new Date(timestamp).toISOString();
  console.log(`[${date}] ${channel}/${author}: ${message}`);
});

app.createChannel('general');
app.sendMessage('general', 'Привет!', 'Alice');
```

**Преимущества делегирования:**
- Скрыта внутренняя реализация
- Можно изменить внутреннюю структуру без изменения публичного API
- Возможность адаптации интерфейса (изменение формата данных)
- Централизованный контроль над операциями

---

## 7. Композиция с агрегацией и делегированием

### Комплексный пример

```javascript
const EventEmitter = require('events');

class Channel {
  constructor(app, name) {
    this.app = app;        // АГРЕГАЦИЯ - ссылка на Application
    this.name = name;
    this.messages = [];
    this.emitter = new EventEmitter(); // КОМПОЗИЦИЯ - владеем emitter
  }

  // ДЕЛЕГИРОВАНИЕ методов emitter
  on(event, handler) {
    this.emitter.on(event, handler);
  }

  send(message, author) {
    const messageData = {
      message,
      author,
      timestamp: Date.now()
    };

    // Сохраняем локально
    this.messages.push(messageData);

    // Эмитим локальное событие
    this.emitter.emit('message', messageData);

    // ДЕЛЕГИРОВАНИЕ глобального события на Application
    this.app.emit('message', {
      channel: this.name,
      ...messageData
    });
  }

  getHistory() {
    return [...this.messages]; // Возвращаем копию для иммутабельности
  }
}

class Application {
  constructor() {
    this.channels = new Map();
    this.emitter = new EventEmitter(); // КОМПОЗИЦИЯ
  }

  // ДЕЛЕГИРОВАНИЕ
  on(event, handler) {
    this.emitter.on(event, handler);
  }

  emit(event, data) {
    this.emitter.emit(event, data);
  }

  // ФАБРИЧНЫЙ МЕТОД с передачей зависимости
  createChannel(name) {
    // Передаем this в Channel - он использует АГРЕГАЦИЮ
    const channel = new Channel(this, name);
    this.channels.set(name, channel);

    this.emit('channel:created', { name });

    return channel;
  }

  // Метод для корректного удаления канала
  removeChannel(name) {
    const channel = this.channels.get(name);

    if (channel) {
      // Очищаем слушателей (предотвращаем утечки памяти)
      channel.emitter.removeAllListeners();
      this.channels.delete(name);
      this.emit('channel:removed', { name });
    }
  }

  getChannel(name) {
    return this.channels.get(name);
  }
}

// Использование
const app = new Application();

// Глобальная подписка на все сообщения
app.on('message', ({ channel, message, author }) => {
  console.log(`[Global] ${channel}/${author}: ${message}`);
});

// Создаем каналы
const general = app.createChannel('general');
const random = app.createChannel('random');

// Локальная подписка на конкретный канал
general.on('message', ({ message, author }) => {
  console.log(`[General только] ${author}: ${message}`);
});

// Отправка сообщений
general.send('Привет!', 'Alice');
random.send('Случайное сообщение', 'Bob');

// Получение истории
console.log('История general:', general.getHistory());

// Корректное удаление канала
app.removeChannel('random');
```

**Что здесь используется:**

1. **Композиция:**
   - `Application` владеет `EventEmitter`
   - `Channel` владеет своим `EventEmitter`
   - Оба отвечают за создание и уничтожение

2. **Агрегация:**
   - `Channel` получает ссылку на `Application`
   - Не владеет `Application`
   - Не отвечает за его уничтожение

3. **Делегирование:**
   - `Application.on()` делегирует `emitter.on()`
   - `Application.emit()` делегирует `emitter.emit()`
   - `Channel.send()` делегирует событие на `Application`

4. **Инкапсуляция:**
   - Внутренние `emitter` скрыты
   - Доступ только через публичные методы
   - `getHistory()` возвращает копию массива

5. **Управление жизненным циклом:**
   - `removeChannel()` корректно очищает ресурсы
   - Предотвращает утечки памяти

---

## 8. Сравнение подходов

### Таблица: Наследование vs Композиция vs Агрегация

| Аспект | Наследование | Композиция | Агрегация |
|--------|-------------|-----------|-----------|
| **Связанность** | Сильная | Средняя | Слабая |
| **Владение** | Неявное | Да (владеет) | Нет (использует) |
| **Жизненный цикл** | Связан с родителем | Полный контроль | Независимый |
| **Гибкость** | Низкая | Средняя | Высокая |
| **Повторное использование** | Сложно | Среднее | Легко |
| **Тестируемость** | Сложная | Средняя | Легкая |
| **Изменяемость** | Сложная | Средняя | Легкая |

### Когда использовать каждый подход

#### Используйте Агрегацию когда:
- Объект должен работать с зависимостью, но не владеть ей
- Зависимость создается и управляется извне
- Нужна максимальная гибкость и тестируемость
- Зависимость может быть разделена между объектами

```javascript
// Пример: Logger использует переданный stream
const logger = new Logger(outputStream);
```

#### Используйте Композицию когда:
- Объект должен полностью контролировать зависимость
- Зависимость является частью объекта
- Нужна гарантия корректного создания и уничтожения
- Упрощение API для пользователей

```javascript
// Пример: Logger создает и владеет stream
const logger = new Logger('./app.log');
```

#### Используйте Наследование когда:
- Есть четкое отношение "является" (is-a)
- Базовый класс специально спроектирован для расширения
- Необходимо полиморфное поведение
- Иерархия неглубокая (1-2 уровня)

```javascript
// Пример: Stream наследует EventEmitter
// потому что Stream ЯВЛЯЕТСЯ EventEmitter
class ReadableStream extends EventEmitter {
  // ...
}
```

**Общее правило:** Предпочитайте композицию и агрегацию наследованию (Composition over Inheritance).

---

## 9. Инкапсуляция и защита внутренностей

### Проблема: Доступ к вложенным объектам

```javascript
// ПЛОХО - цепочка точек раскрывает внутреннюю структуру
class Application {
  constructor() {
    this.state = {
      channels: new Map(),
      users: new Map()
    };
  }
}

const app = new Application();

// Плохо - знание о внутренней структуре
app.state.channels.set('general', new Channel('general'));
app.state.users.set('user1', { name: 'Alice' });

// Очень плохо - глубокая вложенность
console.log(app.state.channels.get('general').messages[0].text);
```

**Проблемы:**
- Клиент должен знать внутреннюю структуру
- Невозможно изменить структуру без breaking changes
- Нет валидации данных
- Легко создать некорректное состояние

### Решение: Инкапсуляция с делегированием

```javascript
class Application {
  constructor() {
    // Приватные поля (convention с _)
    this._channels = new Map();
    this._users = new Map();
  }

  // Публичные методы, скрывающие внутреннюю структуру
  addChannel(name) {
    if (this._channels.has(name)) {
      throw new Error(`Channel ${name} already exists`);
    }

    const channel = new Channel(this, name);
    this._channels.set(name, channel);
    return channel;
  }

  getChannel(name) {
    return this._channels.get(name);
  }

  hasChannel(name) {
    return this._channels.has(name);
  }

  removeChannel(name) {
    const channel = this._channels.get(name);
    if (channel) {
      channel.destroy(); // Корректная очистка
      this._channels.delete(name);
    }
  }

  // Получение сообщений без раскрытия структуры
  getChannelMessages(channelName, limit = 10) {
    const channel = this._channels.get(channelName);

    if (!channel) {
      throw new Error(`Channel ${channelName} not found`);
    }

    // Возвращаем копию с ограничением
    return channel.getHistory().slice(-limit);
  }

  // Итератор для каналов
  *channels() {
    yield* this._channels.values();
  }
}

// ПРАВИЛЬНОЕ использование
const app = new Application();

// Хорошо - использование публичного API
const general = app.addChannel('general');
general.send('Сообщение', 'Alice');

// Хорошо - контролируемый доступ к данным
const messages = app.getChannelMessages('general', 5);
console.log(messages);

// Хорошо - использование итератора
for (const channel of app.channels()) {
  console.log(channel.name);
}
```

### Использование приватных полей (современный JavaScript)

```javascript
class Application {
  // Настоящие приватные поля (ES2022)
  #channels = new Map();
  #users = new Map();
  #emitter = new EventEmitter();

  addChannel(name) {
    if (this.#channels.has(name)) {
      throw new Error(`Channel ${name} already exists`);
    }

    const channel = new Channel(this, name);
    this.#channels.set(name, channel);
    this.#emitter.emit('channel:created', { name });
    return channel;
  }

  // Делегирование с контролем
  on(event, handler) {
    // Можем валидировать события
    const allowedEvents = ['channel:created', 'channel:removed', 'message'];
    if (!allowedEvents.includes(event)) {
      throw new Error(`Unknown event: ${event}`);
    }

    this.#emitter.on(event, handler);
  }

  getChannel(name) {
    return this.#channels.get(name);
  }
}

// Попытка доступа к приватному полю вызовет ошибку
const app = new Application();
// app.#channels // SyntaxError: Private field '#channels' must be declared in an enclosing class
```

---

## 10. Практические рекомендации

### Чек-лист при проектировании

1. **Избегайте внешней ассоциации**
   - Не присваивайте свойства объектам извне
   - Используйте конструкторы, фабрики или сеттеры

2. **Предпочитайте агрегацию и композицию наследованию**
   - Наследование только для отношений "является"
   - Композиция для "содержит" с владением
   - Агрегация для "использует" без владения

3. **Скрывайте внутренности**
   - Не раскрывайте внутреннюю структуру
   - Используйте приватные поля или convention с `_`
   - Избегайте цепочек вызовов через точку

4. **Делегируйте ответственность**
   - Создавайте публичные методы вместо прямого доступа
   - Адаптируйте интерфейсы при необходимости
   - Валидируйте данные в точке входа

5. **Управляйте жизненным циклом**
   - Владелец отвечает за уничтожение
   - Очищайте слушателей событий
   - Предотвращайте утечки памяти

### Пример: Полный цикл от плохого к хорошему коду

#### Плохо: Внешняя ассоциация, доступ к внутренностям

```javascript
class Application {
  constructor() {
    this.channels = [];
    this.emitter = new EventEmitter();
  }
}

const app = new Application();
const channel = new Channel('general');

// Плохо - внешняя ассоциация
app.channels.push(channel);

// Плохо - прямой доступ
app.emitter.emit('channel:created', channel);

// Плохо - лазание в кишки
app.channels[0].messages.push({ text: 'Привет' });
```

#### Средне: Методы есть, но слабая инкапсуляция

```javascript
class Application {
  constructor() {
    this.channels = new Map();
    this.emitter = new EventEmitter();
  }

  addChannel(channel) {
    this.channels.set(channel.name, channel);
    this.emitter.emit('channel:created', channel);
  }
}

const app = new Application();

// Средне - метод есть, но создание снаружи
const channel = new Channel('general');
app.addChannel(channel);

// Все еще можно лазить
app.channels.get('general').messages.push({ text: 'Привет' });
```

#### Хорошо: Полная инкапсуляция, делегирование, управление

```javascript
class Channel {
  #messages = [];
  #emitter = new EventEmitter();

  constructor(app, name) {
    this.app = app;
    this.name = name;
  }

  on(event, handler) {
    this.#emitter.on(event, handler);
  }

  send(message, author) {
    const data = { message, author, timestamp: Date.now() };
    this.#messages.push(data);
    this.#emitter.emit('message', data);
    this.app.notifyMessage(this.name, data);
  }

  getHistory(limit = 100) {
    return this.#messages.slice(-limit).map(msg => ({ ...msg }));
  }

  destroy() {
    this.#emitter.removeAllListeners();
    this.#messages = [];
  }
}

class Application {
  #channels = new Map();
  #emitter = new EventEmitter();

  on(event, handler) {
    this.#emitter.on(event, handler);
  }

  createChannel(name) {
    if (this.#channels.has(name)) {
      throw new Error(`Channel ${name} already exists`);
    }

    const channel = new Channel(this, name);
    this.#channels.set(name, channel);
    this.#emitter.emit('channel:created', { name });
    return channel;
  }

  getChannel(name) {
    return this.#channels.get(name);
  }

  removeChannel(name) {
    const channel = this.#channels.get(name);
    if (channel) {
      channel.destroy();
      this.#channels.delete(name);
      this.#emitter.emit('channel:removed', { name });
    }
  }

  notifyMessage(channelName, messageData) {
    this.#emitter.emit('message', {
      channel: channelName,
      ...messageData
    });
  }

  *channels() {
    yield* this.#channels.values();
  }
}

// ПРАВИЛЬНОЕ использование
const app = new Application();

app.on('channel:created', ({ name }) => {
  console.log(`Канал создан: ${name}`);
});

app.on('message', ({ channel, message, author }) => {
  console.log(`[${channel}] ${author}: ${message}`);
});

const general = app.createChannel('general');
general.send('Привет!', 'Alice');

const history = general.getHistory(10);
console.log('История:', history);

app.removeChannel('general');
```

---

## Резюме

### Ключевые понятия

1. **Инстанцирование** — создание экземпляров и связывание абстракций

2. **Ассоциация** — слабая связь объектов (избегать внешней ассоциации)

3. **Агрегация** — использование объекта без владения (передача через конструктор)
   - Не отвечает за создание
   - Не отвечает за уничтожение
   - Высокая гибкость

4. **Композиция** — владение объектом (создание внутри)
   - Полная ответственность за жизненный цикл
   - Сильная связанность, но упрощение API
   - Подходит для внутренних деталей реализации

5. **Делегирование** — передача ответственности через публичный интерфейс
   - Скрывает внутреннюю структуру
   - Позволяет адаптировать интерфейсы
   - Обеспечивает инкапсуляцию

6. **Инкапсуляция** — сокрытие внутренних деталей реализации
   - Используйте приватные поля
   - Предоставляйте публичные методы
   - Избегайте цепочек доступа через точку

### Принципы проектирования

- **Composition over Inheritance** — предпочитайте композицию наследованию
- **Information Hiding** — скрывайте детали реализации
- **Separation of Concerns** — разделяйте ответственности
- **Loose Coupling** — стремитесь к слабой связанности
- **Dependency Injection** — передавайте зависимости извне (агрегация)

### Практические советы

1. Начинайте с агрегации — это дает максимальную гибкость
2. Переходите к композиции, когда нужен контроль жизненного цикла
3. Используйте делегирование для скрытия деталей реализации
4. Избегайте глубоких иерархий наследования (более 2 уровней)
5. Всегда предоставляйте методы для корректного уничтожения объектов
6. Помните о предотвращении утечек памяти при работе с событиями

---

## Дополнительные ресурсы

### Связанные паттерны

- **Dependency Injection** — передача зависимостей извне
- **Factory Pattern** — создание объектов через фабричные методы
- **Adapter Pattern** — адаптация интерфейсов при делегировании
- **Facade Pattern** — упрощенный интерфейс к сложной системе
- **Proxy Pattern** — контроль доступа к объекту

### Принципы SOLID

- **Single Responsibility** — класс должен иметь одну ответственность
- **Open/Closed** — открыт для расширения, закрыт для модификации
- **Liskov Substitution** — подклассы должны заменять базовые классы
- **Interface Segregation** — много специализированных интерфейсов лучше одного общего
- **Dependency Inversion** — зависеть от абстракций, а не от конкретики

### Для дальнейшего изучения

- Patterns of Enterprise Application Architecture (Martin Fowler)
- Design Patterns: Elements of Reusable Object-Oriented Software (Gang of Four)
- Clean Architecture (Robert Martin)
- Domain-Driven Design (Eric Evans)
