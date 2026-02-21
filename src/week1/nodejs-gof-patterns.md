# Паттерны GoF и архитектура сервера в Node.js: Академический конспект

## Метаданные

- **Источник**: Лекция "Структура классов сервера и GoF паттерны в Node.js"
- **Дата обработки**: 2025-12-06
- **Уровень**: Продвинутый курс по Node.js архитектуре

---

## Содержание

1. [Введение](#введение)
2. [Архитектурные компоненты и их взаимосвязи](#архитектурные-компоненты-и-их-взаимосвязи)
3. [GoF паттерны в Node.js](#gof-паттерны-в-nodejs)
   - [Strategy (Стратегия)](#1-strategy-стратегия)
   - [Proxy (Прокси)](#2-proxy-прокси)
   - [Facade (Фасад)](#3-facade-фасад)
   - [Chain of Responsibility (Цепочка ответственности)](#4-chain-of-responsibility-цепочка-ответственности)
   - [Singleton (Синглтон)](#5-singleton-синглтон)
4. [JavaScript-специфичные паттерны](#javascript-специфичные-паттерны)
   - [EventEmitter vs Publisher-Subscriber](#eventemitter-vs-publisher-subscriber)
   - [Hooks как альтернатива Middleware](#hooks-как-альтернатива-middleware)
5. [Принципы проектирования архитектуры](#принципы-проектирования-архитектуры)
6. [Управление состоянием и изоляция контекстов](#управление-состоянием-и-изоляция-контекстов)
7. [Заключение](#заключение)

---

## Введение

Данный конспект представляет собой академический анализ применения паттернов проектирования из книги "Design Patterns: Elements of Reusable Object-Oriented Software" (Gang of Four, GoF) в контексте разработки серверных приложений на Node.js.

### Почему GoF паттерны выглядят иначе в JavaScript?

JavaScript является **мультипарадигменным языком**, что принципиально отличает его от классических объектно-ориентированных языков (Java, C++, C#), для которых изначально были созданы паттерны GoF. Ключевые отличия:

1. **Функции первого класса** (first-class functions) - функции могут передаваться как значения, храниться в переменных, возвращаться из других функций
2. **Прототипное наследование** вместо классического
3. **Динамическая типизация**
4. **Замыкания** (closures) как механизм инкапсуляции
5. **Асинхронная природа** платформы Node.js

Эти особенности приводят к тому, что некоторые классические паттерны:
- Упрощаются до неузнаваемости
- Заменяются идиоматичными JavaScript-конструкциями
- Получают альтернативные реализации, более подходящие для языка

### Цели архитектуры RPC-сервера

Рассматриваемая архитектура решает следующие задачи:

1. **Интероперабельность протоколов** - поддержка HTTP и WebSocket через единый интерфейс
2. **Изоляция системного слоя** - бизнес-логика не должна видеть детали сетевых протоколов
3. **Управление контекстами** - разделение трех видов состояния (RPC-вызов, клиент, сессия)
4. **Безопасность** - предотвращение race conditions и monkey-patching
5. **Персистентность** - сохранение состояния сессий
6. **Масштабируемость** - возможность добавления аутентификации, авторизации, RBAC

---

## Архитектурные компоненты и их взаимосвязи

### Двухслойная архитектура

Архитектура четко разделена на два слоя с различными уровнями доступа:

#### System Layer (Системный слой)

Компоненты, недоступные из бизнес-логики:

- **Server** - центральный координатор, управляет коллекциями сессий и клиентов
- **Transport** - абстракция над сетевыми протоколами (HTTP/WebSocket)
- **Connection** - WebSocket соединение
- **Request** - HTTP запрос
- **Response** - HTTP ответ

**Почему эти компоненты изолированы?**

1. **Предотвращение утечки абстракций** - бизнес-логика не должна зависеть от конкретного протокола
2. **Безопасность** - прямой доступ к request/response может привести к race conditions
3. **Тестируемость** - бизнес-логику можно тестировать без поднятия реального сервера
4. **Гибкость** - можно менять транспортный протокол без изменения бизнес-логики

#### Userland (Пользовательский слой)

Компоненты, доступные из бизнес-логики:

- **Context** - контекст RPC-вызова
- **Client** - представление клиентского соединения
- **Session** - пользовательская сессия

### Структура классов и их отношения

```
Server (Singleton)
├── sessions: Map<string, Session>  // Глобальная коллекция сессий
├── clients: Set<Client>             // Клиенты текущего порта
├── http: HttpServer                 // HTTP сервер
└── ws: WebSocketServer              // WebSocket сервер

Transport (Strategy pattern)
├── HttpTransport
│   ├── request: IncomingMessage
│   └── response: ServerResponse
└── WebSocketTransport
    ├── request: IncomingMessage
    └── connection: WebSocket

Client (Facade pattern)
├── #transport: Transport            // Приватное поле (скрыто от userland)
├── state: Object                    // Состояние клиента
└── createSession(): Session

Session (Proxy pattern)
├── token: string                    // Уникальный идентификатор
├── state: Proxy                     // Проксируемое состояние
└── context: Context

Context
├── client: Client
├── session: Session | null
└── uuid: string                     // UUID вызова
```

### Поток данных при RPC-вызове

1. Запрос приходит на **Server** (HTTP или WebSocket)
2. **Server** создает соответствующий **Transport** (Strategy)
3. **Transport** оборачивается **Client** (Facade)
4. Создается **Context** с ссылками на Client и Session
5. Вызывается endpoint бизнес-логики с **Context** как единственным параметром
6. Бизнес-логика работает с Context, не зная о деталях протокола
7. Ответ отправляется через Transport, абстрагированный Client

---

## GoF паттерны в Node.js

### 1. Strategy (Стратегия)

#### Академическое определение

**Strategy** (Стратегия) - поведенческий паттерн проектирования, который определяет семейство алгоритмов, инкапсулирует каждый из них и делает их взаимозаменяемыми. Стратегия позволяет изменять алгоритмы независимо от клиентов, которые ими пользуются.

**Классические компоненты паттерна:**
- **Strategy** - общий интерфейс для всех алгоритмов
- **ConcreteStrategy** - конкретные реализации алгоритмов
- **Context** - использует Strategy для выполнения операций

#### Применение в архитектуре: Transport

В данной архитектуре паттерн Strategy применяется для абстрагирования от конкретного транспортного протокола.

**Иерархия классов:**

```javascript
// Базовый класс Transport - определяет интерфейс
class Transport {
  constructor(request) {
    this.request = request;
  }

  // Абстрактный метод - должен быть реализован в наследниках
  write(data) {
    throw new Error('Method write() must be implemented');
  }

  // Общий метод для всех стратегий
  send(data, code = 200, encoding = 'json') {
    // Использует write(), которого еще нет в базовом классе
    // Полагается на реализацию в ConcreteStrategy
    this.write(data);
  }

  error(code, message) {
    this.send({ error: { code, message } });
  }
}

// ConcreteStrategy 1: HTTP транспорт
class HttpTransport extends Transport {
  constructor(request, response) {
    super(request);
    this.response = response;
  }

  write(data) {
    // Специфичная для HTTP реализация
    this.response.writeHead(200, { 'Content-Type': 'application/json' });
    this.response.end(JSON.stringify(data));
  }
}

// ConcreteStrategy 2: WebSocket транспорт
class WebSocketTransport extends Transport {
  constructor(request, connection) {
    super(request);
    this.connection = connection;
  }

  write(data) {
    // Специфичная для WebSocket реализация
    // Нет HTTP кода, нет заголовков - только данные
    this.connection.send(JSON.stringify(data));
  }
}
```

#### Почему именно так в Node.js?

**1. Динамический выбор стратегии**

В Node.js выбор стратегии происходит во время выполнения, основываясь на типе входящего соединения:

```javascript
// В server.js
const createTransport = (request, responseOrConnection) => {
  // Проверяем, было ли HTTP соединение апгрейднуто до WebSocket
  if (request.headers.upgrade === 'websocket') {
    return new WebSocketTransport(request, responseOrConnection);
  }
  return new HttpTransport(request, responseOrConnection);
};
```

**2. Единый интерфейс для различающейся машинерии**

HTTP и WebSocket принципиально различаются:

| Аспект | HTTP | WebSocket |
|--------|------|-----------|
| Тип соединения | Request-Response (одноразовое) | Persistent (постоянное) |
| Направление | Однонаправленное | Двунаправленное |
| Заголовки | Требуются (Content-Type, status code) | Не используются |
| Состояние | Stateless | Stateful |
| Объекты Node.js | IncomingMessage + ServerResponse | WebSocket |

Паттерн Strategy позволяет скрыть эти различия за единым интерфейсом `send()`, `write()`, `error()`.

#### Какую проблему решает?

**Проблема**: Бизнес-логика RPC-сервера не должна знать, по какому протоколу пришел запрос.

**Решение**:
- Все транспорты имеют одинаковый интерфейс
- Клиентский код работает с абстрактным Transport
- Конкретная реализация выбирается при создании соединения
- Добавление новых протоколов (например, gRPC) требует только создания нового ConcreteStrategy

**Преимущества в Node.js контексте:**

1. **Упрощение бизнес-логики** - endpoint'ы не содержат if/else для проверки типа протокола
2. **Открыт для расширения** - новые протоколы добавляются без изменения существующего кода
3. **Закрыт для модификации** - базовый Transport и клиентский код не меняются
4. **Тестируемость** - можно создавать mock-транспорты для тестирования

---

### 2. Proxy (Прокси)

#### Академическое определение

**Proxy** (Прокси, Заместитель) - структурный паттерн проектирования, который предоставляет объект-заместитель вместо реального служебного объекта. Прокси контролирует доступ к оригинальному объекту, позволяя выполнить дополнительную логику до или после передачи вызова оригиналу.

**Типы Proxy:**
- **Remote Proxy** - представляет объект в другом адресном пространстве
- **Virtual Proxy** - отложенная инициализация тяжелых объектов
- **Protection Proxy** - контроль прав доступа
- **Smart Reference** - дополнительная логика при обращении к объекту

#### Применение в архитектуре: Session State

В данной архитектуре используется **Smart Reference Proxy** для перехвата операций чтения и записи свойств состояния сессии.

**Реализация в Метакоме (принцип применим к Node.js проекту):**

```javascript
// Создание Proxy для перехвата доступа к состоянию
const createProxy = (state, save) => {
  return new Proxy(state, {
    // Перехват чтения свойств
    get(target, property) {
      return target[property];
    },

    // Перехват записи свойств
    set(target, property, value) {
      target[property] = value;
      // Важно! При каждом изменении вызываем save()
      save();
      return true;
    },

    // Перехват удаления свойств
    deleteProperty(target, property) {
      delete target[property];
      save();
      return true;
    }
  });
};

// Класс Session использует Proxy
class Session {
  constructor(token, data, saveCallback) {
    this.token = token;
    // Вместо прямого доступа к state, используем Proxy
    this.state = createProxy(data, () => {
      saveCallback(this.token, this.state);
    });
  }
}
```

#### Почему именно так в Node.js?

**1. JavaScript Proxy API**

JavaScript предоставляет встроенный механизм **Proxy** (ES6), который является мощным метапрограммированием:

```javascript
const handler = {
  get(target, property, receiver) { /* перехват чтения */ },
  set(target, property, value, receiver) { /* перехват записи */ },
  deleteProperty(target, property) { /* перехват удаления */ },
  has(target, property) { /* перехват 'in' оператора */ },
  // ... еще 10+ ловушек (traps)
};

const proxy = new Proxy(target, handler);
```

**2. Преимущества над классическим Proxy паттерном**

В классических ООП языках Proxy требует:
- Создания интерфейса для объекта
- Реализации всех методов интерфейса в Proxy
- Явного делегирования вызовов

JavaScript Proxy:
- **Прозрачный** - выглядит как обычный объект
- **Универсальный** - перехватывает все операции, включая динамически добавляемые свойства
- **Минимальный код** - не нужно явно делегировать каждый метод

#### Какую проблему решает?

**Проблема**: Необходимо отслеживать изменения состояния сессии для сохранения в persistent storage (Redis, PostgreSQL).

**Наивное решение (неправильное):**

```javascript
// Плохо: требует явного вызова save()
session.state.username = 'alice';
session.state.role = 'admin';
session.save(); // Легко забыть!
```

**Решение с Proxy:**

```javascript
// Хорошо: save() вызывается автоматически
session.state.username = 'alice';  // Автоматический save()
session.state.role = 'admin';       // Автоматический save()
```

**Дополнительные возможности:**

1. **Вычисление дельты изменений**

```javascript
const createProxy = (state, save) => {
  const changes = new Set();

  return new Proxy(state, {
    set(target, property, value) {
      if (target[property] !== value) {
        changes.add(property);
        target[property] = value;
        save(changes); // Передаем только измененные поля
      }
      return true;
    }
  });
};
```

2. **Валидация данных**

```javascript
set(target, property, value) {
  // Можем добавить валидацию
  if (property === 'age' && typeof value !== 'number') {
    throw new TypeError('Age must be a number');
  }
  target[property] = value;
  save();
  return true;
}
```

3. **Логирование доступа**

```javascript
get(target, property) {
  console.log(`Reading property: ${property}`);
  return target[property];
}
```

#### Архитектурное значение

Proxy в контексте сессий обеспечивает:

1. **Автоматическая персистентность** - состояние сохраняется без явных вызовов
2. **Прозрачность** - код бизнес-логики не знает о механизме сохранения
3. **Производительность** - можно оптимизировать, сохраняя только измененные поля
4. **Отложенная запись** - можно дебаунсить save() для уменьшения нагрузки на БД

**Примечание**: В текущей учебной реализации Node.js проекта Proxy еще не используется, но в production (Метаком) он активно применяется именно таким образом.

---

### 3. Facade (Фасад)

#### Академическое определение

**Facade** (Фасад) - структурный паттерн проектирования, который предоставляет упрощенный интерфейс к сложной подсистеме, содержащей множество взаимосвязанных классов и объектов.

**Назначение паттерна:**
- Упрощение сложного интерфейса
- Уменьшение зависимостей клиента от внутренних классов подсистемы
- Создание единой точки входа в подсистему

#### Применение в архитектуре: Client

Класс **Client** является фасадом, который скрывает сложную машинерию сетевых протоколов от бизнес-логики.

**Что скрывает Client:**

```javascript
class Client {
  // Приватное поле (# синтаксис TypeScript/ES2022)
  #transport;

  constructor(transport) {
    // Скрываем внутренности:
    // - IncomingMessage (Node.js HTTP)
    // - ServerResponse (Node.js HTTP)
    // - WebSocket (ws библиотека)
    this.#transport = transport;

    // Публичный интерфейс - только то, что нужно userland
    this.state = {};
  }

  // Публичные методы фасада
  createSession(token, data) {
    const session = new Session(token, data);
    return session;
  }

  // Скрытый метод для внутреннего использования
  send(data) {
    this.#transport.send(data);
  }

  // Деструктор для очистки ресурсов
  finalization() {
    // Очистка ссылок на сессии, tokens
    this.state = null;
  }
}
```

#### Почему именно так в Node.js?

**1. Приватные поля (#syntax)**

JavaScript/TypeScript поддерживают приватные поля через `#` синтаксис (ES2022):

```javascript
class Client {
  #transport;  // Истинно приватное поле

  // Нельзя обратиться извне:
  // client.#transport  // SyntaxError
}
```

**Альтернативы до ES2022:**
- Замыкания (closures)
- WeakMap для хранения приватных данных
- Соглашение о именовании (_privateField)

**2. Композиция вместо наследования**

Фасад в Node.js часто реализуется через композицию:

```javascript
// Client содержит Transport, но не наследует от него
class Client {
  #transport;  // Композиция

  // Делегирует только необходимые операции
  send(data) {
    return this.#transport.send(data);
  }
}
```

**Почему не наследование?**

```javascript
// Плохо: публикует весь интерфейс Transport
class Client extends Transport {
  // Проблема: весь интерфейс Transport доступен
  // client.write(), client.response, client.connection
}
```

#### Какую проблему решает?

**Проблема**: Бизнес-логика не должна иметь доступ к низкоуровневым деталям HTTP/WebSocket.

**Опасности прямого доступа:**

1. **Race Conditions**

```javascript
// Плохо: прямой доступ к response
async function endpoint(context) {
  const { response } = context;  // Опасно!

  await processData();
  response.end('OK');  // Первый ответ

  await anotherOperation();
  response.end('DONE');  // Ошибка! response уже завершен
}
```

2. **Monkey Patching**

```javascript
// Плохо: изменение поведения протокола
context.response.writeHead = () => {
  // Сломанная логика
};
```

3. **Зависимость от фреймворка**

```javascript
// Плохо: бизнес-логика знает о Node.js HTTP API
if (context.request.headers['content-type'] === 'application/json') {
  // Логика завязана на Node.js IncomingMessage
}
```

**Решение с Facade (Client):**

```javascript
// Хорошо: бизнес-логика работает с абстракцией
async function endpoint(context) {
  const { client } = context;

  // Простой, безопасный интерфейс
  await client.send({ status: 'processing' });

  // Нет прямого доступа к request/response
  // Нет возможности для race conditions
}
```

#### Архитектурное значение

Facade (Client) обеспечивает:

1. **Инкапсуляция** - детали протокола скрыты
2. **Стабильный интерфейс** - изменения в Transport не влияют на бизнес-логику
3. **Безопасность** - невозможно напрямую манипулировать сокетами
4. **Тестируемость** - Client можно легко замокировать

**Примечание о визуализации:**

В схеме архитектуры пунктирные зеленые стрелки показывают приватные поля (#):
- Client → Transport (приватная ссылка)
- Transport → Connection/Response/Request (приватные ссылки)

Бизнес-логика видит только Client, но не то, что за ним скрыто.

---

### 4. Chain of Responsibility (Цепочка ответственности)

#### Академическое определение

**Chain of Responsibility** (Цепочка обязанностей, Цепочка ответственности) - поведенческий паттерн проектирования, который позволяет передавать запросы последовательно по цепочке обработчиков. Каждый обработчик решает, может ли он обработать запрос, и стоит ли передавать запрос дальше по цепочке.

**Ключевые характеристики классического паттерна:**

1. **Один обработчик** - только один элемент цепочки должен обработать запрос
2. **Прекращение цепочки** - обработка останавливается, когда найден подходящий обработчик
3. **Слабая связанность** - отправитель не знает, какой обработчик обработает запрос

**Классическая реализация:**

```javascript
class Handler {
  setNext(handler) {
    this.next = handler;
    return handler;
  }

  handle(request) {
    if (this.canHandle(request)) {
      return this.process(request);
    }
    if (this.next) {
      return this.next.handle(request);  // Передача дальше
    }
    return null;  // Никто не обработал
  }
}
```

#### Проблемы адаптации для HTTP/Node.js: Middleware

В Node.js/Express экосистеме широко распространен паттерн **Middleware**, который является **нарушенной реализацией** Chain of Responsibility.

**Что такое Middleware?**

```javascript
// Express/Connect middleware
app.use((req, res, next) => {
  // Некоторая обработка
  console.log('Middleware 1');
  next();  // Передача следующему
});

app.use((req, res, next) => {
  console.log('Middleware 2');
  next();
});

app.use((req, res, next) => {
  console.log('Middleware 3');
  res.send('Done');
});
```

#### Почему Middleware - это "сломанный" Chain of Responsibility?

**Проблема 1: Все обрабатывают запрос, а не один**

```javascript
// Chain of Responsibility: ОДИН обработчик
handler1.handle(request);  // Если обработал, цепочка прерывается
// ИЛИ
handler2.handle(request);  // Выполнится только если handler1 не обработал

// Middleware: ВСЕ обработчики
middleware1(req, res, next);  // Выполняется
middleware2(req, res, next);  // Выполняется
middleware3(req, res, next);  // Выполняется тоже
```

**Проблема 2: Race Conditions**

```javascript
// Плохо: несколько middleware могут отправить ответ
app.use((req, res, next) => {
  res.send('Response 1');  // Первый ответ
  next();  // Продолжаем выполнение!
});

app.use((req, res, next) => {
  res.send('Response 2');  // Ошибка! Заголовки уже отправлены
});
```

**Проблема 3: Неявная ответственность**

```javascript
app.use(parseCookies);       // Меняет req.cookies
app.use(checkSession);       // Меняет req.session
app.use(checkPermissions);   // Меняет req.user
app.use(handleRequest);      // Использует все вышеперечисленное

// Кто ответственен за какое состояние? Неясно!
```

**Проблема 4: Мутация общего состояния**

```javascript
app.use((req, res, next) => {
  req.customField = 'value1';
  next();
});

app.use((req, res, next) => {
  req.customField = 'value2';  // Перезаписали!
  next();
});
```

#### Решение в данной архитектуре: Map-based Routing

Вместо Chain of Responsibility или Middleware, используется **прямой роутинг** через коллекцию:

```javascript
// Роутинг через Map (или Object)
const routes = new Map();

// Регистрация endpoints
routes.set('user.login', async (context) => {
  // Обработка login
});

routes.set('user.logout', async (context) => {
  // Обработка logout
});

// Вызов конкретного endpoint
const handler = routes.get(methodName);
if (handler) {
  await handler(context);  // Только один обработчик!
} else {
  throw new Error('Method not found');
}
```

**Преимущества:**

1. **Явная ответственность** - один endpoint отвечает за один метод
2. **Нет race conditions** - только один обработчик вызывается
3. **Нет мутации общего состояния** - каждый endpoint изолирован
4. **Производительность** - O(1) поиск обработчика вместо O(n)

#### Альтернатива Middleware: Hooks (Fastify)

**Hooks** - это событийная модель, которая безопаснее Middleware:

```javascript
// Fastify hooks
fastify.addHook('onRequest', async (request, reply) => {
  // Выполняется перед обработкой запроса
  // Не может отправить ответ (reply недоступен)
});

fastify.addHook('preHandler', async (request, reply) => {
  // Выполняется перед handler
  // Может прервать выполнение через reply.send()
});

fastify.addHook('onSend', async (request, reply, payload) => {
  // Выполняется перед отправкой ответа
  // Может модифицировать payload
  return modifiedPayload;
});

fastify.get('/route', async (request, reply) => {
  // Основной handler - единственный ответственный за ответ
  return { data: 'value' };
});
```

**Почему Hooks лучше Middleware?**

1. **Четкие фазы** - каждый hook срабатывает на определенной стадии lifecycle
2. **Явная ответственность** - основной handler ответственен за ответ
3. **Нет race conditions** - hooks не могут случайно отправить несколько ответов
4. **Изоляция** - hooks для разных маршрутов не влияют друг на друга

#### Рекомендации для архитектуры

Для добавления cross-cutting concerns (логирование, аутентификация, валидация):

**Используйте Hooks, не Middleware:**

```javascript
// Хорошо: Hook для проверки аутентификации
server.addHook('preHandler', async (context) => {
  if (!context.session) {
    throw new Error('Not authenticated');
  }
});

// Основной handler
async function endpoint(context) {
  // Аутентификация уже проверена
  return { data: 'secure data' };
}
```

**Или используйте декораторы/обертки:**

```javascript
// Декоратор для проверки прав
const requireAuth = (handler) => {
  return async (context) => {
    if (!context.session) {
      throw new Error('Not authenticated');
    }
    return handler(context);
  };
};

// Применение
routes.set('user.profile', requireAuth(async (context) => {
  return { profile: context.session.user };
}));
```

---

### 5. Singleton (Синглтон)

#### Академическое определение

**Singleton** (Одиночка) - порождающий паттерн проектирования, который гарантирует, что у класса есть только один экземпляр, и предоставляет глобальную точку доступа к этому экземпляру.

#### Применение в архитектуре: Server

Класс **Server** функционирует как синглтон в контексте одного процесса Node.js:

```javascript
class Server {
  constructor(config) {
    this.config = config;
    this.sessions = new Map();      // Глобальная коллекция сессий
    this.clients = new Set();       // Клиенты текущего сервера
    this.routes = new Map();        // Роутинг
  }

  listen(port) {
    // Один сервер слушает один порт
    this.http = http.createServer(/* ... */);
    this.ws = new WebSocketServer(/* ... */);

    this.http.listen(port);
  }
}

// В приложении
const server = new Server(config);
server.listen(8000);
```

#### Почему "почти" синглтон?

**Формально не синглтон:**
- Можно создать несколько экземпляров `new Server()`
- Нет принудительного ограничения на создание

**Практически синглтон:**
- Один сервер = один порт
- Глобальные коллекции сессий и клиентов
- Обычно одно приложение = один экземпляр Server

**Модуль Node.js как синглтон:**

```javascript
// server.js
class Server { /* ... */ }

// Экспорт единственного экземпляра
module.exports = new Server(config);

// При импорте получаем тот же экземпляр
const server = require('./server');  // Всегда один и тот же объект
```

Node.js кеширует модули - `require()` для одного модуля возвращает один и тот же экспортированный объект.

---

## JavaScript-специфичные паттерны

### EventEmitter vs Publisher-Subscriber

#### Классический Publisher-Subscriber (GoF)

**Publisher-Subscriber** (Издатель-Подписчик, Observer) - поведенческий паттерн, который определяет зависимость "один-ко-многим" между объектами таким образом, что при изменении состояния одного объекта все зависящие от него оповещаются автоматически.

**Классическая реализация в ООП:**

```java
// Java пример
interface Subscriber {
  void update(Event event);  // Все подписчики должны иметь этот метод
}

class Publisher {
  private List<Subscriber> subscribers = new ArrayList<>();

  public void subscribe(Subscriber subscriber) {
    subscribers.add(subscriber);
  }

  public void notify(Event event) {
    for (Subscriber subscriber : subscribers) {
      subscriber.update(event);  // Вызываем известный метод
    }
  }
}

class ConcreteSubscriber implements Subscriber {
  public void update(Event event) {
    // Обработка события
  }
}
```

**Проблемы в JavaScript:**

1. **Необходимость интерфейса** - все подписчики должны реализовать `update()`
2. **Избыточность** - передаем ссылку на весь объект, а нужна только функция
3. **Негибкость** - фиксированное имя метода (`update`)

#### EventEmitter в Node.js

Node.js предоставляет встроенный класс **EventEmitter**, который является идиоматичной JavaScript-реализацией Publisher-Subscriber.

**Почему EventEmitter лучше для JavaScript?**

**1. Функции первого класса**

```javascript
const EventEmitter = require('events');

class Server extends EventEmitter {
  handleConnection(client) {
    // Уведомляем подписчиков
    this.emit('connection', client);
  }
}

const server = new Server();

// Подписчики передают функции, не объекты!
server.on('connection', (client) => {
  console.log('Client connected');
});

server.on('connection', (client) => {
  console.log('Another handler for same event');
});
```

**Преимущества:**

- **Не нужен интерфейс** - подписчик = любая функция
- **Множественные подписчики** - несколько функций на одно событие
- **Различные события** - не ограничены одним методом `update()`

**2. Отсутствие необходимости в наследовании подписчиков**

```javascript
// Классический Publisher-Subscriber требует:
class MySubscriber implements Subscriber { /* ... */ }

// EventEmitter - любая функция:
const handler = (data) => console.log(data);
server.on('data', handler);

// Или даже анонимная:
server.on('data', (data) => console.log(data));
```

**3. Именованные события**

```javascript
server.on('connection', handleConnection);
server.on('close', handleClose);
server.on('error', handleError);
server.on('data', handleData);

// В классическом Publisher-Subscriber:
// Все события идут через один метод update()
// Требуется switch/if для различения типов
```

**4. Встроенная функциональность**

```javascript
// Подписка на одно срабатывание
server.once('ready', () => {
  console.log('Server ready');
});

// Отписка
server.off('connection', handler);

// Удаление всех подписчиков
server.removeAllListeners('connection');

// Получение списка подписчиков
const listeners = server.listeners('connection');

// Установка максимума подписчиков (memory leak warning)
server.setMaxListeners(20);
```

#### Использование в архитектуре

**Client с EventEmitter:**

```javascript
const EventEmitter = require('events');

class Client extends EventEmitter {
  constructor(transport) {
    super();  // Инициализация EventEmitter
    this.#transport = transport;

    // Подписываемся на события транспорта
    this.#transport.on('data', (data) => {
      this.emit('message', data);  // Пробрасываем событие
    });

    this.#transport.on('close', () => {
      this.emit('disconnect');
    });
  }

  send(data) {
    this.#transport.send(data);
  }
}

// Использование
const client = new Client(transport);

client.on('message', (data) => {
  console.log('Received:', data);
});

client.on('disconnect', () => {
  console.log('Client disconnected');
});
```

#### Сравнительная таблица

| Аспект | Publisher-Subscriber (классический) | EventEmitter (Node.js) |
|--------|-------------------------------------|------------------------|
| Подписчик | Объект с методом `update()` | Любая функция |
| Интерфейс | Требуется | Не требуется |
| Типы событий | Обычно один тип (или enum) | Именованные строки |
| Язык | Типичен для Java, C#, C++ | Идиоматичен для JavaScript |
| Отписка | Вручную удаление из списка | `off()`, `removeListener()` |
| Одноразовые подписки | Требуют дополнительной логики | `once()` |

---

### Hooks как альтернатива Middleware

#### Проблема паттерна Middleware

Как обсуждалось ранее, Middleware имеет фундаментальные проблемы:

1. Все обработчики выполняются, а не один
2. Возможны race conditions при отправке ответа
3. Мутация общего состояния (request/response)
4. Неявная ответственность

#### Паттерн Hook

**Hook** (Перехватчик, Крючок) - паттерн, при котором в определенных точках жизненного цикла объекта вызываются зарегистрированные обработчики событий.

**Отличия от Middleware:**

| Аспект | Middleware | Hooks |
|--------|-----------|-------|
| Концепция | Цепочка обработчиков | События жизненного цикла |
| Ответственность | Все обрабатывают | Каждый hook - свою фазу |
| Ответ клиенту | Любой middleware может отправить | Только основной handler |
| Порядок | Важен | Определен жизненным циклом |
| Изоляция | Общий request/response | Четкие границы |

#### Реализация Hooks в Fastify

```javascript
const fastify = require('fastify')();

// Хуки жизненного цикла запроса в Fastify
fastify.addHook('onRequest', async (request, reply) => {
  // Фаза 1: Запрос получен, но еще не обработан
  console.log('Request received');
  // reply еще нельзя использовать для отправки ответа
});

fastify.addHook('preParsing', async (request, reply) => {
  // Фаза 2: Перед парсингом тела запроса
  // Можно модифицировать поток данных
});

fastify.addHook('preValidation', async (request, reply) => {
  // Фаза 3: Перед валидацией
  // Тело уже распарсено, но не валидировано
});

fastify.addHook('preHandler', async (request, reply) => {
  // Фаза 4: Перед основным handler
  // Проверка аутентификации, прав доступа

  if (!request.session) {
    reply.code(401).send({ error: 'Unauthorized' });
    // Основной handler НЕ выполнится
  }
});

// Основной handler - ЕДИНСТВЕННЫЙ ответственный за бизнес-логику
fastify.get('/api/data', async (request, reply) => {
  return { data: 'value' };
});

fastify.addHook('onSend', async (request, reply, payload) => {
  // Фаза 5: Перед отправкой ответа
  // Можно модифицировать payload
  const modified = { timestamp: Date.now(), ...payload };
  return modified;
});

fastify.addHook('onResponse', async (request, reply) => {
  // Фаза 6: После отправки ответа
  // Логирование, метрики
  console.log(`Response sent in ${reply.getResponseTime()}ms`);
});

fastify.addHook('onError', async (request, reply, error) => {
  // Обработка ошибок
  console.error('Error occurred:', error);
});
```

#### Жизненный цикл запроса с Hooks

```
Incoming Request
      ↓
┌─────────────┐
│ onRequest   │ ← Логирование, установка request ID
└─────────────┘
      ↓
┌─────────────┐
│ preParsing  │ ← Декомпрессия, трансформация потока
└─────────────┘
      ↓
  [Парсинг тела]
      ↓
┌─────────────────┐
│ preValidation   │ ← Добавление данных для валидации
└─────────────────┘
      ↓
  [Валидация схемы]
      ↓
┌─────────────┐
│ preHandler  │ ← Аутентификация, авторизация
└─────────────┘
      ↓
┌─────────────┐
│   Handler   │ ← ОСНОВНАЯ БИЗНЕС-ЛОГИКА (единственная ответственность)
└─────────────┘
      ↓
┌─────────────┐
│   onSend    │ ← Сериализация, сжатие
└─────────────┘
      ↓
┌─────────────┐
│ onResponse  │ ← Метрики, очистка ресурсов
└─────────────┘
      ↓
  Response Sent

(в любой момент может произойти)
      ↓
┌─────────────┐
│   onError   │ ← Обработка ошибок
└─────────────┘
```

#### Преимущества Hooks

**1. Четкая ответственность**

```javascript
// Каждый hook знает свою роль
fastify.addHook('preHandler', checkAuth);      // Только аутентификация
fastify.addHook('preHandler', checkRBAC);      // Только авторизация
fastify.addHook('onSend', compressResponse);   // Только сжатие

// Handler знает только бизнес-логику
fastify.get('/user/:id', getUserHandler);      // Только получение пользователя
```

**2. Нет race conditions**

```javascript
// Невозможно отправить несколько ответов
fastify.addHook('preHandler', async (request, reply) => {
  if (condition) {
    reply.send('Early response');
    // Основной handler НЕ выполнится - Fastify это контролирует
  }
});
```

**3. Scope hooks (локальные хуки)**

```javascript
// Глобальный hook - для всех маршрутов
fastify.addHook('onRequest', globalLogger);

// Hook только для конкретного маршрута
fastify.get('/admin', {
  onRequest: [checkAdmin]  // Только для этого маршрута
}, adminHandler);

// Hook для группы маршрутов (плагин)
fastify.register(async (fastify) => {
  // Хуки здесь применяются только к маршрутам в этом плагине
  fastify.addHook('preHandler', requireAuth);

  fastify.get('/profile', profileHandler);
  fastify.get('/settings', settingsHandler);
});
```

**4. Тестируемость**

```javascript
// Хуки можно тестировать независимо
test('checkAuth hook', async (t) => {
  const request = mockRequest({ session: null });
  const reply = mockReply();

  await checkAuth(request, reply);

  t.equal(reply.statusCode, 401);
});

// Handler тестируется без хуков
test('getUserHandler', async (t) => {
  const request = mockRequest({ params: { id: '123' } });
  const result = await getUserHandler(request);

  t.equal(result.id, '123');
});
```

#### Реализация простого Hook системы для Node.js проекта

```javascript
// Простая реализация hook системы
class HookSystem {
  constructor() {
    this.hooks = new Map();
  }

  addHook(name, handler) {
    if (!this.hooks.has(name)) {
      this.hooks.set(name, []);
    }
    this.hooks.get(name).push(handler);
  }

  async executeHooks(name, ...args) {
    const handlers = this.hooks.get(name) || [];
    for (const handler of handlers) {
      await handler(...args);
    }
  }
}

// Использование в сервере
class Server extends HookSystem {
  constructor() {
    super();
  }

  async handleRequest(context) {
    // Lifecycle с hooks
    await this.executeHooks('onRequest', context);
    await this.executeHooks('preHandler', context);

    // Основной handler
    const result = await this.routes.get(context.method)(context);

    await this.executeHooks('onSend', context, result);

    return result;
  }
}

// Регистрация hooks
server.addHook('preHandler', async (context) => {
  if (!context.session) {
    throw new Error('Not authenticated');
  }
});

server.addHook('onSend', async (context, result) => {
  console.log('Sending response:', result);
});
```

---

## Принципы проектирования архитектуры

### 1. Separation of Concerns (Разделение ответственности)

**Принцип**: Различные аспекты функциональности должны быть изолированы в отдельных компонентах.

**Применение в архитектуре:**

```
System Layer (протоколы)     ┃ Userland (бизнес-логика)
                              ┃
Server                        ┃
Transport (HTTP/WebSocket)    ┃ Context
Request, Response, Connection ┃ Client
                              ┃ Session
```

**Почему важно:**
- Бизнес-логика не знает о HTTP/WebSocket
- Смена протокола не требует изменения бизнес-логики
- Можно тестировать бизнес-логику без сети

### 2. Dependency Inversion Principle (Принцип инверсии зависимостей)

**Принцип**: Модули верхнего уровня не должны зависеть от модулей нижнего уровня. Оба должны зависеть от абстракций.

**Применение:**

```javascript
// Плохо: прямая зависимость от конкретной реализации
class Handler {
  constructor() {
    this.transport = new HttpTransport();  // Жесткая зависимость
  }
}

// Хорошо: зависимость от абстракции
class Handler {
  constructor(transport) {  // Transport - абстракция
    this.transport = transport;  // Может быть HTTP или WebSocket
  }
}
```

### 3. Principle of Least Privilege (Принцип минимальных привилегий)

**Принцип**: Код должен иметь доступ только к тому, что ему необходимо для работы.

**Применение:**

```javascript
// Бизнес-логика получает только Context
async function endpoint(context) {
  // Доступно:
  // - context.client (ограниченный интерфейс)
  // - context.session (состояние пользователя)

  // Недоступно:
  // - transport (скрыт за приватным полем)
  // - request/response (не передаются)
  // - connection (изолирован)
}
```

### 4. Fail-Safe Defaults (Безопасные значения по умолчанию)

**Принцип**: По умолчанию система должна быть в безопасном состоянии.

**Применение:**

```javascript
// HTTP transport всегда возвращает 200 и JSON по умолчанию
send(data, code = 200, encoding = 'json') {
  // Безопасные дефолты
}

// Сессия может быть null (незалогиненный пользователь)
class Context {
  constructor(client, session = null) {
    this.client = client;
    this.session = session;  // null по умолчанию - безопасно
  }
}
```

### 5. Single Responsibility Principle (Принцип единственной ответственности)

**Принцип**: Каждый класс должен иметь одну причину для изменения.

**Применение:**

- **Server** - управление соединениями и роутинг
- **Transport** - отправка/получение данных по протоколу
- **Client** - фасад для взаимодействия с userland
- **Session** - управление состоянием пользователя
- **Context** - контейнер данных для RPC вызова

### 6. Open/Closed Principle (Принцип открытости/закрытости)

**Принцип**: Классы должны быть открыты для расширения, но закрыты для модификации.

**Применение:**

```javascript
// Добавление нового транспорта не требует изменения существующих классов
class GrpcTransport extends Transport {
  write(data) {
    // Специфичная для gRPC реализация
  }
}

// Server и Client работают с любым Transport
```

---

## Управление состоянием и изоляция контекстов

### Три типа состояния

#### 1. State RPC-вызова (Context State)

**Характеристики:**
- **Время жизни**: От начала вызова до получения ответа
- **Область видимости**: Только текущий вызов
- **Персистентность**: Не сохраняется
- **Изоляция**: Полная - каждый вызов имеет свой контекст

**Назначение:**
- Хранение параметров вызова
- Временные данные обработки
- UUID для трейсинга

**Пример:**

```javascript
class Context {
  constructor(client, session) {
    this.uuid = randomUUID();  // Уникальный для каждого вызова
    this.client = client;
    this.session = session;
    this.timestamp = Date.now();
    // Временное состояние, уничтожается после ответа
  }
}

// Каждый вызов создает новый контекст
const context1 = new Context(client, session);
await endpoint(context1);
// context1 больше не нужен

const context2 = new Context(client, session);
await endpoint(context2);
// context2 независим от context1
```

#### 2. State клиента (Client State)

**Характеристики:**
- **Время жизни**: От подключения до отключения клиента
- **Область видимости**: Все RPC вызовы от данного соединения
- **Персистентность**: Не сохраняется (пропадает при отключении)
- **Изоляция**: Между клиентами

**Назначение:**
- Временные данные соединения
- Счетчики, метрики текущего соединения
- Кеш для оптимизации

**Пример:**

```javascript
class Client {
  constructor(transport) {
    this.state = {
      connectedAt: Date.now(),
      requestCount: 0,
      lastActivity: Date.now()
    };
  }

  async handleRequest(context) {
    this.state.requestCount++;
    this.state.lastActivity = Date.now();
    // State доступен между вызовами
  }
}
```

**Use case:**

```javascript
// Ограничение rate limit на основе Client State
async function rateLimit(context) {
  const { client } = context;

  if (!client.state.rateLimitWindow) {
    client.state.rateLimitWindow = Date.now();
    client.state.requestsInWindow = 0;
  }

  const elapsed = Date.now() - client.state.rateLimitWindow;

  if (elapsed > 60000) {
    // Сброс окна каждую минуту
    client.state.rateLimitWindow = Date.now();
    client.state.requestsInWindow = 0;
  }

  if (client.state.requestsInWindow >= 100) {
    throw new Error('Rate limit exceeded');
  }

  client.state.requestsInWindow++;
}
```

#### 3. State сессии (Session State)

**Характеристики:**
- **Время жизни**: От создания сессии до её истечения (expiration)
- **Область видимости**: Все соединения пользователя (может подключиться с разных устройств)
- **Персистентность**: Сохраняется в БД (Redis, PostgreSQL)
- **Изоляция**: Между пользователями

**Назначение:**
- Данные аутентификации
- Пользовательские настройки
- Корзина покупок
- Прогресс работы

**Пример:**

```javascript
class Session {
  constructor(token, data) {
    this.token = token;  // Уникальный токен сессии
    this.state = data;   // Персистентное состояние
    this.createdAt = Date.now();
  }

  // State может содержать:
  // {
  //   userId: '123',
  //   username: 'alice',
  //   roles: ['user', 'moderator'],
  //   preferences: { theme: 'dark' },
  //   cart: [{ productId: 'abc', quantity: 2 }]
  // }
}
```

**Персистентность с Proxy (как обсуждалось ранее):**

```javascript
// При изменении state автоматически сохраняется в БД
session.state.cart.push({ productId: 'xyz', quantity: 1 });
// Proxy перехватывает изменение и вызывает save()
```

### Сравнительная таблица

| Аспект | Context State | Client State | Session State |
|--------|---------------|--------------|---------------|
| Время жизни | Вызов | Соединение | До expiration |
| Персистентность | Нет | Нет | Да (БД) |
| Общий между вызовами | Нет | Да (одно соединение) | Да (все соединения) |
| Пример данных | UUID, timestamp | Request count, cache | User data, cart |
| Уничтожается при | Ответ отправлен | Отключение | Logout/Expiration |

### Изоляция контекстов

**Проблема**: В многопользовательском сервере критически важно, чтобы один пользователь не мог получить доступ к данным другого.

**Решение в архитектуре:**

**1. Каждый запрос создает новый Context**

```javascript
// В server.js
async function handleRPC(methodName, params, client, session) {
  // Создаем НОВЫЙ контекст для каждого вызова
  const context = new Context(client, session);

  try {
    const handler = routes.get(methodName);
    const result = await handler(context);  // Передаем только context
    return result;
  } finally {
    // context уничтожается после вызова
    // Garbage collector освободит память
  }
}
```

**2. Сессии хранятся в Map по токену**

```javascript
// Глобальная коллекция сессий
const sessions = new Map();

// Каждая сессия изолирована по токену
sessions.set('token-alice', aliceSession);
sessions.set('token-bob', bobSession);

// Получение сессии по токену из запроса
const token = extractTokenFromRequest(request);
const session = sessions.get(token);  // Только своя сессия
```

**3. Клиенты хранятся в Set**

```javascript
// Коллекция клиентов
const clients = new Set();

clients.add(clientAlice);
clients.add(clientBob);

// Каждый клиент независим
// clientAlice не может получить доступ к clientBob
```

**4. Бизнес-логика не имеет глобального доступа**

```javascript
// Плохо: глобальный доступ к коллекциям
async function badEndpoint() {
  const allSessions = server.sessions;  // Опасно!
  // Может получить доступ к чужим сессиям
}

// Хорошо: только свой контекст
async function goodEndpoint(context) {
  const mySession = context.session;  // Только своя сессия
  const myClient = context.client;    // Только свой клиент
  // Изолировано
}
```

### Предотвращение утечек состояния

**1. Замыкания для приватного доступа**

```javascript
// В server.js
function createServer(config) {
  const sessions = new Map();  // Приватная коллекция в замыкании
  const clients = new Set();

  class Server {
    // Методы имеют доступ через замыкание
    getSession(token) {
      return sessions.get(token);
    }

    addClient(client) {
      clients.add(client);
    }
  }

  return new Server();
}

// sessions и clients недоступны извне
```

**2. Приватные поля (#)**

```javascript
class Client {
  #transport;  // Недоступен извне

  constructor(transport) {
    this.#transport = transport;
  }

  // Публичный интерфейс безопасен
  send(data) {
    this.#transport.send(data);
  }
}

// client.#transport  // SyntaxError!
```

**3. Freezing и Sealing объектов**

```javascript
// Предотвращение модификации конфигурации
const config = Object.freeze({
  port: 8000,
  host: 'localhost'
});

config.port = 9000;  // Ошибка в strict mode, игнорируется в sloppy
console.log(config.port);  // 8000
```

---

## Заключение

### Ключевые выводы

#### 1. JavaScript - не классический ООП

Паттерны GoF созданы для статически типизированных классических ООП языков. В JavaScript:

- **Функции первого класса** упрощают Observer → EventEmitter
- **Замыкания** заменяют приватные поля классов
- **Прототипное наследование** более гибкое
- **Динамическая типизация** позволяет обойтись без интерфейсов
- **Proxy API** делает паттерн Proxy тривиальным

#### 2. Не все GoF паттерны полезны в Node.js

**Полезные паттерны:**
- **Strategy** - для абстракций (Transport)
- **Facade** - для скрытия сложности (Client)
- **Proxy** - для перехвата доступа (Session state)
- **Singleton** - для глобальных сервисов (Server)

**Проблемные паттерны:**
- **Chain of Responsibility** → используйте Map-routing или Hooks
- **Publisher-Subscriber** → используйте EventEmitter
- **Middleware** → используйте Hooks (Fastify)

#### 3. Архитектурные принципы важнее паттернов

Паттерны - это средство, а не цель. Важнее:

1. **Separation of Concerns** - System Layer vs Userland
2. **Dependency Inversion** - зависимости от абстракций
3. **Single Responsibility** - один класс, одна роль
4. **Principle of Least Privilege** - минимальный доступ
5. **Изоляция** - контексты не должны пересекаться

#### 4. Управление состоянием - критично для безопасности

Три типа state (Context, Client, Session) должны быть четко разделены:

- **Context** - временный, одноразовый
- **Client** - на время соединения
- **Session** - персистентный, между соединениями

Изоляция обеспечивается через:
- Новый Context для каждого вызова
- Map/Set для хранения сессий/клиентов по ключам
- Приватные поля и замыкания
- Передача только необходимого контекста

#### 5. JavaScript-идиоматичные подходы

Вместо слепого следования GoF:

- **EventEmitter** вместо Publisher-Subscriber
- **Hooks** вместо Chain of Responsibility
- **Map-routing** вместо Middleware
- **Proxy API** для метапрограммирования
- **Замыкания** для инкапсуляции

### Следующие шаги

Следующие темы для изучения в контексте данной архитектуры:

1. **Аутентификация и идентификация**
   - Создание и валидация токенов
   - Безопасное хранение паролей (bcrypt, scrypt)
   - Session management

2. **Role-Based Access Control (RBAC)**
   - Роли и права
   - Проверка прав доступа к endpoints
   - Композиция прав из нескольких ролей

3. **Revealing Constructor Pattern**
   - Для работы с Streams
   - Контролируемое раскрытие функциональности

4. **CORS и Security Headers**
   - Реализация CORS вручную
   - Безопасные заголовки HTTP

5. **Streams и backpressure**
   - Обработка больших файлов
   - Управление потоками данных

### Рекомендуемая литература

**Книги:**
- "Design Patterns: Elements of Reusable Object-Oriented Software" (GoF) - для понимания классических паттернов
- "JavaScript Patterns" by Stoyan Stefanov - паттерны специфичные для JavaScript
- "Node.js Design Patterns" by Mario Casciaro - паттерны для Node.js

**Документация:**
- [Node.js Events API](https://nodejs.org/api/events.html) - EventEmitter
- [Node.js HTTP API](https://nodejs.org/api/http.html) - HTTP сервер
- [Fastify Hooks](https://www.fastify.io/docs/latest/Reference/Hooks/) - lifecycle hooks

**Кодовая база:**
- [Metarhia/Metaсom](https://github.com/metarhia/metacom) - production реализация
- Текущий проект - учебная реализация

---

**Автор конспекта**: Академический анализ лекции
**Дата**: 2025-12-06
**Версия**: 1.0
