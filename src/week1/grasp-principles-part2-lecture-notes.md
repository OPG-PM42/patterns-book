# Принципы GRASP для JavaScript/TypeScript/Node.js (Часть 2)

> Конспект лекции по принципам GRASP с практическими пояснениями для JS/TS разработчиков

## Введение

GRASP (General Responsibility Assignment Software Patterns) — это набор из **9 принципов**, представляющих собой общие идеи и философию программирования, а не жёсткие паттерны проектирования (как GoF).

**Важно понимать**: принципы GRASP — это не строгие правила, а руководящие идеи. Они могут применяться по-разному в зависимости от контекста и не привязаны исключительно к ООП.

### Принципы из первой части (уже рассмотренные):
- Information Expert (Информационный эксперт)
- Creator (Создатель)
- Low Coupling (Слабое зацепление)
- High Cohesion (Сильная связность)

### Принципы этой лекции:
- Protected Variations (Устойчивость к изменениям)
- Indirection (Перенаправление)
- Pure Fabrication (Чистая выдумка)
- Polymorphism (Полиморфизм)
- Controller (Контроллер)

---

## 1. Protected Variations (Устойчивость к изменениям)

### Суть принципа

**Protected Variations** говорит о необходимости защитить код от каскадных изменений. Когда мы меняем одну абстракцию, это не должно приводить к необходимости менять всё остальное.

### Проблема

Когда код не защищён от изменений:
- Одно изменение тянет за собой другие
- Работает "закон Диметры" — высокое зацепление между модулями
- Абстракции "лезут в кишки друг друга"

### Пример проблемы в Node.js

```javascript
// ПЛОХО: Лезем во внутренности стримов Node.js
const stream = fs.createReadStream('file.txt');

// Раскопали недокументированную структуру буферизации
const internalBuffer = stream._readableState.buffer;
const chunkCount = internalBuffer.length; // Мониторим внутренние чанки

// При обновлении Node.js внутренняя структура может измениться,
// хотя внешний API стримов остался прежним — и наш код сломается
```

**Что было нарушено:**
- **Information Expert** — залезли в чужую абстракцию
- **Creator** — работаем с данными, которые создавал не наш код
- **Low Coupling** — зацепились за негарантированный контракт

### Решение: работа через интерфейсы

Для защиты кода от изменений необходимо:

1. **Работать через документированные интерфейсы**
2. **Использовать тайпинги** (`.d.ts` файлы для JS, интерфейсы для TS)
3. **Покрывать контракты тестами**
4. **Соблюдать семантическое версионирование**

```typescript
// ХОРОШО: Работаем через публичный API
import { Readable } from 'stream';

function processStream(stream: Readable): void {
  stream.on('data', (chunk) => {
    // Используем только документированные события и методы
    console.log(`Received ${chunk.length} bytes`);
  });

  stream.on('end', () => {
    console.log('Stream finished');
  });
}
```

### Семантическое версионирование и контракты

| Версия | Изменение | Совместимость |
|--------|-----------|---------------|
| `1.0.0` → `1.0.1` | Патч, небольшое улучшение | Полная совместимость |
| `1.0.0` → `1.1.0` | Новые функции | Контракты не изменились |
| `1.0.0` → `2.0.0` | Breaking changes | Контракты изменились |

### Когда можно пожертвовать принципом

**При прототипировании** — когда важно быстро проверить идею:
- Не защищаем код от изменений
- Главная цель — "чтобы хотя бы работало"
- Такой код может быть выброшен

**Для долгоживущего кода** — принцип обязателен:
- Выделяем интерфейсы
- Используем контрактное программирование
- Покрываем тестами

---

## 2. Indirection (Перенаправление)

### Суть принципа

**Indirection** — это введение промежуточной абстракции между двумя другими, чтобы уменьшить их зацепление.

### Зачем нужно

- Снижает coupling между абстракциями
- Позволяет менять одну абстракцию независимо от другой
- Реализует Protected Variations

### Примеры паттернов с Indirection

- **MVC (Model-View-Controller)** — Controller знает о Model и View, но они не знают друг о друга
- **Bridge (Мост)** — третья абстракция связывает две других

### Пример: сервис авторизации

```typescript
// БЕЗ Indirection: Logger и UserRights напрямую связаны с бизнес-логикой
class AuthService {
  async validateUser(credentials: Credentials): Promise<boolean> {
    // Прямые вызовы к логгеру из бизнес-логики
    logger.info('Validating user...');
    const user = await db.findUser(credentials.email);

    if (!user) {
      logger.warn('User not found');
      return false;
    }
    // ...
  }
}

// С Indirection: AuthController как промежуточный слой
class AuthController {
  constructor(
    private logger: Logger,
    private userService: UserService,
    private rightsService: RightsService
  ) {}

  async authenticate(credentials: Credentials): Promise<AuthResult> {
    // Контроллер координирует работу, но не содержит бизнес-логику
    const user = await this.userService.validate(credentials);

    if (!user) {
      this.logger.warn('Authentication failed');
      return { success: false };
    }

    const rights = await this.rightsService.getUserRights(user.id);
    this.logger.info(`User ${user.id} authenticated with rights: ${rights}`);

    return { success: true, user, rights };
  }
}
```

### Способы реализации Indirection

Indirection не привязан к ООП. Это может быть:
- **Класс** — агрегирует две другие абстракции
- **Функция** — принимает зависимости как параметры
- **Модуль** — экспортирует функции, работающие с другими модулями

---

## 3. Pure Fabrication (Чистая выдумка)

### Суть принципа

**Pure Fabrication** — это абстракция, которая не существует в предметной области, но необходима для технической реализации системы.

### Примеры чистых выдумок

| Предметная область | Чистая выдумка |
|-------------------|----------------|
| Пациент, Заказ, Сотрудник | EventEmitter |
| Покупка, Транспорт | AbortController |
| Накладная, Анализ | Таймеры, Сигналы |
| — | Мютексы, Семафоры |
| — | Файловые хендлы |
| — | Сокеты, Стримы |

### Почему это важно

Традиционное ООП учит моделировать реальный мир (кошечки, собачки). Но в системном программировании **всё** — чистая выдумка:

```javascript
// Всё это — чистые выдумки, не существующие в реальном мире
const controller = new AbortController();
const signal = controller.signal;
const emitter = new EventEmitter();
const stream = new Readable();
const mutex = new Mutex();
```

### Уровни чистых выдумок

Чистые выдумки существуют на разных уровнях абстракции:

```
┌─────────────────────────────────────────┐
│  Высокоуровневые: Controller, Endpoint  │
├─────────────────────────────────────────┤
│  Прикладные: EventEmitter, Stream       │
├─────────────────────────────────────────┤
│  Системные: File Handle, Socket         │
├─────────────────────────────────────────┤
│  ОС: Mutex, Semaphore, Memory Page      │
└─────────────────────────────────────────┘
```

### Культура JavaScript

В JS/TS принято моделировать предметную область через простые структуры данных (как записи в БД), а не через классы с методами:

```typescript
// JavaScript культура: простые объекты для предметной области
interface Patient {
  id: string;
  name: string;
  dateOfBirth: Date;
  diagnoses: Diagnosis[];
}

// Чистая выдумка: сервис для работы с пациентами
class PatientService {
  constructor(private db: Database, private events: EventEmitter) {}

  async createPatient(data: CreatePatientDTO): Promise<Patient> {
    const patient = await this.db.patients.create(data);
    this.events.emit('patient:created', patient);
    return patient;
  }
}
```

### Функциональное программирование

Монады и функторы — тоже чистые выдумки:

```typescript
// Maybe — чистая выдумка из ФП
type Maybe<T> = T | null;

// Either — тоже чистая выдумка
type Either<L, R> = { type: 'left'; value: L } | { type: 'right'; value: R };
```

---

## 4. Polymorphism (Полиморфизм в GRASP)

### Отличие от ООП-полиморфизма

GRASP-полиморфизм — это надстройка над классическим полиморфизмом. Он говорит о том, что **за внешним интерфейсом может быть любая реализация**.

### Примеры

```typescript
// Внешний контракт стримов — один
interface ReadableStream {
  read(): Buffer | null;
  on(event: 'data', listener: (chunk: Buffer) => void): void;
  on(event: 'end', listener: () => void): void;
}

// Но реализации могут быть разными:
// - FileReadStream читает с диска
// - HttpIncomingMessage читает из сети
// - MemoryStream читает из памяти
// Внешний код не знает и не должен знать о деталях
```

### Реализации приватности в JavaScript

Один и тот же интерфейс — разные способы скрыть данные:

```typescript
// 1. Символы
const privateData = Symbol('private');
class ServiceV1 {
  [privateData] = { secret: 'value' };
}

// 2. Замыкания
function createServiceV2() {
  const privateData = { secret: 'value' };
  return {
    getPublicData() { return privateData.secret; }
  };
}

// 3. Приватные поля ES2022
class ServiceV3 {
  #privateData = { secret: 'value' };
  getPublicData() { return this.#privateData.secret; }
}

// Внешний контракт у всех одинаков — есть метод getPublicData()
```

### Thenable как пример полиморфизма

```typescript
// Контракт thenable
interface Thenable<T> {
  then<R>(onFulfilled: (value: T) => R): Thenable<R>;
}

// Встроенный Promise — одна реализация
const promise = Promise.resolve(42);

// Кастомная реализация — другая
class CustomThenable<T> implements Thenable<T> {
  constructor(private value: T) {}
  then<R>(onFulfilled: (value: T) => R): CustomThenable<R> {
    return new CustomThenable(onFulfilled(this.value));
  }
}

// Обе работают с async/await благодаря соблюдению контракта
```

---

## 5. Controller (Контроллер)

### Суть принципа

**Controller** — это абстракция, которая принимает внешнюю нагрузку и защищает систему, но **не содержит бизнес-логики**.

### Обязанности контроллера

1. Приём внешних запросов
2. Валидация входных данных
3. Проверка контрактов
4. Криптографическая валидация (токены, подписи)
5. Делегирование работы сервисам

### Чего контроллер НЕ должен делать

- Содержать бизнес-логику
- Создавать объекты предметной области
- Напрямую работать с базой данных
- Принимать бизнес-решения

### Терминология для Node.js

Автор лекции рекомендует использовать термин **endpoint** вместо "контроллер":

| Термин | Значение |
|--------|----------|
| **Front Controller** | Общая точка входа, принимает все запросы |
| **Controller / Endpoint** | Обработчик конкретного маршрута |
| **Service** | Бизнес-логика |

```typescript
// Front Controller — обычно часть фреймворка (Express, Fastify, NestJS)
const app = express();

// Endpoint (контроллер) — принимает запрос, делегирует сервису
app.post('/api/orders', async (req, res) => {
  // 1. Валидация входных данных
  const validation = orderSchema.safeParse(req.body);
  if (!validation.success) {
    return res.status(400).json({ errors: validation.error.issues });
  }

  // 2. Проверка авторизации
  const user = await authService.validateToken(req.headers.authorization);
  if (!user) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  // 3. Делегирование бизнес-логики сервису
  const order = await orderService.createOrder(user.id, validation.data);

  // 4. Формирование ответа
  return res.status(201).json(order);
});

// Service — содержит бизнес-логику
class OrderService {
  async createOrder(userId: string, data: CreateOrderDTO): Promise<Order> {
    // Здесь вся бизнес-логика: расчёты, правила, работа с БД
    const order = new Order(userId, data.items);
    await this.calculateTotals(order);
    await this.applyDiscounts(order);
    return this.orderRepository.save(order);
  }
}
```

### Разные протоколы — одна концепция

Контроллеры/endpoints работают с любым протоколом:

```typescript
// HTTP
app.get('/api/users/:id', userController.getById);

// WebSocket
ws.on('message', (data) => messageController.handle(data));

// gRPC
server.addService(UserService, userController);

// UDP
udpServer.on('message', (msg) => packetController.handle(msg));
```

---

## Когда применять принципы GRASP

### Матрица принятия решений

| Ситуация | Применять строго? | Комментарий |
|----------|-------------------|-------------|
| Прототип на день | Нет | Главное — чтобы работало |
| MVP на неделю | Частично | Базовая структура |
| Продукт на месяц | Да | Полная архитектура |
| Долгоживущая система | Обязательно | Документация + тесты |

### Время vs Качество

Одну и ту же задачу (10 endpoints) можно сделать:
- **За день** — работает, но хрупко
- **За неделю** — удобно поддерживать
- **За месяц** — документировано, оттестировано, оптимизировано
- **За год** — готово к миллиону пользователей

Выбор зависит от бизнес-требований и срока жизни кода.

---

## Ключевые выводы

1. **GRASP — это философия**, а не строгие правила. Применяйте с умом.

2. **Protected Variations** — защищайте код от каскадных изменений через интерфейсы и контракты.

3. **Indirection** — вводите промежуточные абстракции для снижения coupling.

4. **Pure Fabrication** — не бойтесь создавать технические абстракции, не существующие в предметной области.

5. **Polymorphism** — за одним интерфейсом может быть любая реализация.

6. **Controller** — точка входа в систему, без бизнес-логики.

7. **Контекст важен** — при прототипировании можно жертвовать принципами ради скорости.

---

## Связанные материалы

- Лекция по контрактному программированию (рекомендация автора)
- GoF паттерны: Adapter, Facade, Bridge
- MVC, MVP, MVVM архитектуры
- SOLID принципы
