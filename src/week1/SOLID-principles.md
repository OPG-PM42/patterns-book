# SOLID Принципы в JavaScript, TypeScript и Node.js

> **Полный образовательный конспект**
>
> Принципы объектно-ориентированного проектирования, адаптированные для JavaScript/TypeScript экосистемы

---

## Введение

### Что такое SOLID?

**SOLID** - это акроним пяти основных принципов объектно-ориентированного проектирования, которые помогают создавать гибкие, масштабируемые и поддерживаемые системы:

- **S** - Single Responsibility Principle (Принцип единственной ответственности)
- **O** - Open-Closed Principle (Принцип открытости/закрытости)
- **L** - Liskov Substitution Principle (Принцип подстановки Лисков)
- **I** - Interface Segregation Principle (Принцип разделения интерфейсов)
- **D** - Dependency Inversion Principle (Принцип инверсии зависимостей)

### 💡 Почему SOLID важен для JavaScript/TypeScript?

Многие разработчики ошибочно считают, что SOLID применим только к статически типизированным языкам (Java, C#). Однако эти принципы **универсальны** и работают в:

- Процедурном программировании
- Функциональном программировании
- Реактивном программировании
- Прототипном программировании (JavaScript)

**Ключевое отличие:** В JavaScript/TypeScript мы адаптируем эти принципы к специфике языка, а не копируем слепо подходы из Java.

### 🎯 Цели применения SOLID

1. **Снижение связанности** между модулями
2. **Повышение когезии** внутри модуля
3. **Упрощение тестирования**
4. **Облегчение поддержки и рефакторинга**
5. **Улучшение читаемости кода**

### ⚠️ Культура языка vs Догмы

JavaScript имеет свою уникальную культуру, сформированную под влиянием:
- Event-driven модель (браузер, Node.js)
- Функции первого класса
- Динамическая типизация
- Прототипное наследование

**Не копируйте Java подходы слепо!** Адаптируйте принципы к культуре JavaScript.

---

## SRP - Single Responsibility Principle

> **Принцип единственной ответственности**
>
> Каждая абстракция (класс, функция, модуль) должна иметь только **одну причину для изменения**.

### 📚 Определение и суть принципа

**SRP утверждает:** Каждая абстракция отвечает за **одну конкретную часть функциональности** и эта ответственность должна быть **полностью инкапсулирована**.

**Почему это важно:**
- Изменения в одной части системы не влияют на другие
- Код легче понимать и тестировать
- Упрощается рефакторинг
- Снижается риск регрессии при изменениях

### 🔴 Применение в JavaScript/TypeScript

SRP применим не только к классам, но и к:
- **Функциям** - одна функция делает одно дело
- **Модулям** - один модуль имеет одну ответственность
- **Компонентам** - один компонент решает одну задачу

### ❌ Пример НАРУШЕНИЯ SRP

```typescript
// ПЛОХО: класс Employee делает слишком много
class Employee {
  name: string;
  surname: string;
  dateOfBirth: Date;
  salary: number;
  position: string;

  // Бизнес-логика
  calculateSalary(): number {
    // Логика расчета зарплаты
  }

  sendToVacation(): void {
    // Бизнес-процесс отправки в отпуск
  }

  // Работа с БД (инфраструктура)
  save(): Promise<void> {
    // Сохранение в БД
  }

  delete(): Promise<void> {
    // Удаление из БД
  }

  // Отчетность
  generateYearReport(): Report {
    // Генерация годового отчета
  }

  // Системные функции
  changePassword(newPassword: string): void {
    // Изменение пароля входа в систему
  }
}
```

**Проблемы этого кода:**
1. Смешивает **предметную область** (бизнес-логику) и **инфраструктуру** (БД)
2. Нарушает разделение ответственности между:
   - Доменной логикой (расчет зарплаты)
   - Персистентностью (сохранение в БД)
   - Отчетностью (генерация отчетов)
   - Аутентификацией (смена пароля)
3. Любое изменение в БД, отчетах или аутентификации требует модификации класса Employee

### ✅ Пример СОБЛЮДЕНИЯ SRP

```typescript
// ХОРОШО: Анемичная модель - только данные
class Employee {
  constructor(
    public readonly id: string,
    public name: string,
    public surname: string,
    public dateOfBirth: Date,
    public salary: number,
    public position: string
  ) {}
}

// Бизнес-логика в отдельном сервисе
class EmployeeService {
  calculateSalary(employee: Employee): number {
    // Логика расчета зарплаты
    return employee.salary;
  }

  sendToVacation(employee: Employee, startDate: Date, endDate: Date): void {
    // Бизнес-процесс отправки в отпуск
  }
}

// Работа с БД в отдельном репозитории (Data Access Layer)
class EmployeeRepository {
  async save(employee: Employee): Promise<void> {
    // Сохранение в БД
  }

  async delete(employeeId: string): Promise<void> {
    // Удаление из БД
  }

  async findById(id: string): Promise<Employee | null> {
    // Поиск по ID
  }
}

// Отчетность в отдельном сервисе
class EmployeeReportService {
  generateYearReport(employee: Employee): Report {
    // Генерация годового отчета
  }
}

// Аутентификация в отдельном модуле
class AuthService {
  async changePassword(userId: string, newPassword: string): Promise<void> {
    // Изменение пароля
  }
}
```

**Преимущества:**
- ✅ Каждый класс имеет **одну ответственность**
- ✅ Изменения в БД не затрагивают бизнес-логику
- ✅ Легко тестировать каждую часть независимо
- ✅ Легко заменить реализацию (например, сменить БД)

### 🚫 Active Record - Антипаттерн

**Active Record** - это паттерн, который смешивает данные и методы работы с БД в одном классе.

```typescript
// АНТИПАТТЕРН: Active Record
class User {
  id: number;
  name: string;
  email: string;

  // Методы работы с БД прямо в модели
  async save(): Promise<void> {
    await db.query('INSERT INTO users ...', [this.name, this.email]);
  }

  async delete(): Promise<void> {
    await db.query('DELETE FROM users WHERE id = ?', [this.id]);
  }

  static async find(id: number): Promise<User> {
    const result = await db.query('SELECT * FROM users WHERE id = ?', [id]);
    return new User(result);
  }
}
```

**Почему это плохо:**

1. **Нарушает SRP** - класс отвечает и за данные, и за персистентность
2. **Нарушает OCP** - при добавлении новой БД нужно модифицировать класс
3. **Затрудняет тестирование** - сложно тестировать бизнес-логику без БД
4. **Проблемы масштабирования:**
   - Работа с несколькими БД
   - Немапинг 1:1 (одна сущность хранится в нескольких таблицах)
   - Разная логика сохранения для разных контекстов

### ✅ Альтернатива: Repository Pattern + Data Access Layer

```typescript
// Анемичная модель
class User {
  constructor(
    public id: number,
    public name: string,
    public email: string
  ) {}
}

// Интерфейс репозитория
interface IUserRepository {
  save(user: User): Promise<void>;
  delete(userId: number): Promise<void>;
  findById(id: number): Promise<User | null>;
  findAll(): Promise<User[]>;
}

// Конкретная реализация для PostgreSQL
class PostgresUserRepository implements IUserRepository {
  constructor(private db: PostgresClient) {}

  async save(user: User): Promise<void> {
    await this.db.query(
      'INSERT INTO users (name, email) VALUES ($1, $2)',
      [user.name, user.email]
    );
  }

  async delete(userId: number): Promise<void> {
    await this.db.query('DELETE FROM users WHERE id = $1', [userId]);
  }

  async findById(id: number): Promise<User | null> {
    const result = await this.db.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    return result.rows[0] ? new User(
      result.rows[0].id,
      result.rows[0].name,
      result.rows[0].email
    ) : null;
  }

  async findAll(): Promise<User[]> {
    const result = await this.db.query('SELECT * FROM users');
    return result.rows.map(row => new User(row.id, row.name, row.email));
  }
}

// Легко заменить реализацию
class MongoUserRepository implements IUserRepository {
  constructor(private db: MongoClient) {}

  async save(user: User): Promise<void> {
    await this.db.collection('users').insertOne({
      name: user.name,
      email: user.email
    });
  }

  // ... остальные методы
}
```

**Преимущества Repository Pattern:**
- ✅ Разделение ответственности (SRP)
- ✅ Легко менять реализацию (OCP)
- ✅ Легко тестировать (можно создать mock репозиторий)
- ✅ Поддержка нескольких БД

### 🎯 SRP в функциональном стиле

SRP применим не только к классам, но и к функциям:

```typescript
// ❌ ПЛОХО: функция делает слишком много
async function processUser(userId: string) {
  const user = await db.getUser(userId); // Доступ к БД
  const validated = validateUser(user);  // Валидация
  await sendEmail(user.email);          // Отправка email
  await logAction('user_processed');     // Логирование
  return user;
}

// ✅ ХОРОШО: разделение ответственности
async function getUserFromDb(userId: string): Promise<User> {
  return await db.getUser(userId);
}

async function notifyUser(user: User): Promise<void> {
  await sendEmail(user.email);
}

async function logUserAction(action: string): Promise<void> {
  await logAction(action);
}

// Композиция функций
async function processUser(userId: string): Promise<User> {
  const user = await getUserFromDb(userId);
  await notifyUser(user);
  await logUserAction('user_processed');
  return user;
}
```

### 💡 SRP на уровне модулей (Node.js)

```
user/
  ├── user.model.ts         # Модель данных (анемичная)
  ├── user.repository.ts    # Работа с БД
  ├── user.service.ts       # Бизнес-логика
  ├── user.validator.ts     # Валидация
  ├── user.controller.ts    # HTTP endpoints
  └── index.ts              # Экспорты модуля
```

### 🔑 Ключевые выводы по SRP

> **"Класс должен иметь только одну причину для изменения"**

1. **Разделяйте** предметную область и инфраструктуру
2. **Избегайте** Active Record паттерна
3. **Используйте** Repository Pattern для работы с данными
4. **Применяйте** SRP к функциям, классам и модулям
5. **Создавайте** анемичные модели данных
6. **Выносите** бизнес-логику в сервисы

---

## OCP - Open-Closed Principle

> **Принцип открытости/закрытости**
>
> Программные сущности должны быть **открыты для расширения**, но **закрыты для модификации**.

### 📚 Определение и суть принципа

**OCP утверждает:** Вы должны иметь возможность добавлять новую функциональность **без изменения существующего кода**.

**Почему это важно:**
- Снижается риск регрессии - старый код не меняется
- Новая функциональность добавляется через расширение
- Код становится более стабильным
- Упрощается поддержка и тестирование

### 🔴 Как Active Record нарушает OCP

```typescript
// ПЛОХО: Active Record нарушает OCP
class Product {
  id: number;
  name: string;
  price: number;

  // Базовая реализация для одной БД
  async save(): Promise<void> {
    await db.query('INSERT INTO products ...', [this.name, this.price]);
  }
}

// При необходимости работы с другой БД приходится МОДИФИЦИРОВАТЬ класс
class Product {
  id: number;
  name: string;
  price: number;

  database: string; // Добавили поле - МОДИФИКАЦИЯ

  // Переписали метод - МОДИФИКАЦИЯ
  async save(): Promise<void> {
    if (this.database === 'postgres') {
      await postgres.query('INSERT INTO products ...');
    } else if (this.database === 'mongo') {
      await mongo.insertOne({ name: this.name, price: this.price });
    }
    // Нарушение OCP - для каждой новой БД модифицируем метод
  }
}
```

**Проблемы:**
- ❌ Каждая новая БД требует модификации класса Product
- ❌ Нарушается принцип LSP (разные наследники ведут себя по-разному)
- ❌ Растет сложность метода save()

### ✅ Решение: Strategy Pattern + Dependency Injection

```typescript
// Интерфейс стратегии сохранения
interface SaveStrategy {
  save(product: Product): Promise<void>;
  delete(productId: number): Promise<void>;
}

// Стратегия для PostgreSQL
class PostgresSaveStrategy implements SaveStrategy {
  constructor(private db: PostgresClient) {}

  async save(product: Product): Promise<void> {
    await this.db.query(
      'INSERT INTO products (name, price) VALUES ($1, $2)',
      [product.name, product.price]
    );
  }

  async delete(productId: number): Promise<void> {
    await this.db.query('DELETE FROM products WHERE id = $1', [productId]);
  }
}

// Стратегия для MongoDB - РАСШИРЕНИЕ без модификации
class MongoSaveStrategy implements SaveStrategy {
  constructor(private db: MongoClient) {}

  async save(product: Product): Promise<void> {
    await this.db.collection('products').insertOne({
      name: product.name,
      price: product.price
    });
  }

  async delete(productId: number): Promise<void> {
    await this.db.collection('products').deleteOne({ id: productId });
  }
}

// Анемичная модель
class Product {
  constructor(
    public id: number,
    public name: string,
    public price: number
  ) {}
}

// Репозиторий использует стратегию
class ProductRepository {
  constructor(private saveStrategy: SaveStrategy) {}

  async save(product: Product): Promise<void> {
    await this.saveStrategy.save(product);
  }

  async delete(productId: number): Promise<void> {
    await this.saveStrategy.delete(productId);
  }
}

// Использование
const postgresStrategy = new PostgresSaveStrategy(postgresClient);
const productRepo = new ProductRepository(postgresStrategy);

// Легко заменить на MongoDB без модификации Product или ProductRepository
const mongoStrategy = new MongoSaveStrategy(mongoClient);
const productRepoMongo = new ProductRepository(mongoStrategy);
```

**Преимущества:**
- ✅ Добавление новой БД - **расширение** через новую стратегию
- ✅ Класс Product остается неизменным
- ✅ Легко тестировать - можно создать mock стратегию

### 🎯 OCP в функциональном программировании

```typescript
// Базовая функция
type Validator<T> = (value: T) => boolean;

// Создаем валидаторы через композицию
const isNotEmpty: Validator<string> = (value) => value.length > 0;
const isEmail: Validator<string> = (value) => /\S+@\S+\.\S+/.test(value);
const minLength = (min: number): Validator<string> =>
  (value) => value.length >= min;

// Комбинируем валидаторы - РАСШИРЕНИЕ
const and = <T>(...validators: Validator<T>[]): Validator<T> =>
  (value) => validators.every(v => v(value));

const or = <T>(...validators: Validator<T>[]): Validator<T> =>
  (value) => validators.some(v => v(value));

// Использование
const validateEmail = and(isNotEmpty, isEmail);
const validatePassword = and(isNotEmpty, minLength(8));

// Легко добавить новые правила без модификации существующих
const hasUpperCase: Validator<string> = (value) => /[A-Z]/.test(value);
const hasNumber: Validator<string> = (value) => /\d/.test(value);

const validateStrongPassword = and(
  isNotEmpty,
  minLength(8),
  hasUpperCase,
  hasNumber
);
```

### 💡 OCP в Express middleware

```typescript
// Каждый middleware - это расширение без модификации цепочки
import express, { Request, Response, NextFunction } from 'express';

const app = express();

// Базовые middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Добавляем новую функциональность через middleware - РАСШИРЕНИЕ
const logger = (req: Request, res: Response, next: NextFunction) => {
  console.log(`${req.method} ${req.path}`);
  next();
};

const authenticate = (req: Request, res: Response, next: NextFunction) => {
  // Проверка аутентификации
  next();
};

const validateBody = (schema: any) =>
  (req: Request, res: Response, next: NextFunction) => {
    // Валидация тела запроса
    next();
  };

// Применяем middleware - расширяем функциональность
app.use(logger);
app.use(authenticate);

app.post('/users', validateBody(userSchema), (req, res) => {
  // Обработчик
});
```

**Преимущества middleware подхода:**
- ✅ Новая функциональность добавляется без модификации существующего кода
- ✅ Каждый middleware делает одну вещь (SRP)
- ✅ Легко комбинировать и переиспользовать

### 🔑 Ключевые выводы по OCP

> **"Открыт для расширения, закрыт для модификации"**

1. **Используйте** интерфейсы и абстракции
2. **Применяйте** Strategy Pattern для вариативного поведения
3. **Избегайте** модификации существующего кода при добавлении функций
4. **Расширяйте** через наследование, композицию или плагины
5. **Проектируйте** API так, чтобы новые фичи не ломали старый код

---

## LSP - Liskov Substitution Principle

> **Принцип подстановки Лисков**
>
> Объекты подтипа должны быть **заменяемы объектами базового типа** без нарушения работы программы.

### 📚 Определение и суть принципа

**LSP утверждает:** Если класс B наследует класс A, то везде, где ожидается A, можно передать B **без изменения корректности программы**.

**Формальное определение (Барбара Лисков):**
- Предусловия не могут быть усилены в подтипе
- Постусловия не могут быть ослаблены в подтипе
- Инварианты базового типа должны сохраняться в подтипе
- **Наследник не должен добавлять новые исключения**

**Почему это важно:**
- Код становится предсказуемым
- Полиморфизм работает корректно
- Упрощается тестирование
- Снижается количество условных операторов (if/else)

### 🔴 Нарушение LSP в Node.js: EventEmitter vs Stream

```typescript
import { EventEmitter } from 'events';
import { Readable } from 'stream';

// EventEmitter - базовый класс
class BaseEmitter extends EventEmitter {
  doSomething() {
    // Ошибки обрабатываются через событие 'error'
    this.emit('error', new Error('Something went wrong'));
  }
}

// Stream наследует EventEmitter, но меняет поведение ошибок
class DataStream extends Readable {
  _read() {
    // В Stream ошибки могут быть unhandled exceptions
    // Если нет подписчика на 'error', приложение упадет
    throw new Error('Stream error'); // Unhandled exception!
  }
}

// Проблема: нарушение LSP
function handleEmitter(emitter: EventEmitter) {
  // Ожидаем, что ошибки через событие 'error'
  emitter.on('error', (err) => {
    console.error('Caught error:', err);
  });
}

const baseEmitter = new BaseEmitter();
handleEmitter(baseEmitter); // Работает корректно

const stream = new DataStream();
handleEmitter(stream); // Может упасть с unhandled exception!
```

**Проблема:** Stream ведет себя иначе чем EventEmitter при обработке ошибок, что нарушает LSP.

### ❌ Классический пример нарушения LSP: Rectangle vs Square

```typescript
// ПЛОХО: нарушение LSP
class Rectangle {
  constructor(
    protected width: number,
    protected height: number
  ) {}

  setWidth(width: number): void {
    this.width = width;
  }

  setHeight(height: number): void {
    this.height = height;
  }

  getArea(): number {
    return this.width * this.height;
  }
}

// Квадрат - это прямоугольник, правда? НЕТ в ООП!
class Square extends Rectangle {
  constructor(size: number) {
    super(size, size);
  }

  // Нарушение LSP: меняем поведение
  setWidth(width: number): void {
    this.width = width;
    this.height = width; // Побочный эффект!
  }

  setHeight(height: number): void {
    this.width = height; // Побочный эффект!
    this.height = height;
  }
}

// Тест
function testRectangle(rect: Rectangle) {
  rect.setWidth(5);
  rect.setHeight(10);

  // Ожидаем площадь 50
  console.assert(rect.getArea() === 50, 'Area should be 50');
}

const rectangle = new Rectangle(0, 0);
testRectangle(rectangle); // ✅ Работает: площадь = 50

const square = new Square(0);
testRectangle(square); // ❌ Не работает: площадь = 100, а не 50!
```

**Проблема:** Square меняет поведение базового класса Rectangle, что нарушает ожидания.

### ✅ Решение: композиция вместо наследования

```typescript
// ХОРОШО: используем композицию
interface Shape {
  getArea(): number;
}

class Rectangle implements Shape {
  constructor(
    private width: number,
    private height: number
  ) {}

  setWidth(width: number): void {
    this.width = width;
  }

  setHeight(height: number): void {
    this.height = height;
  }

  getArea(): number {
    return this.width * this.height;
  }
}

class Square implements Shape {
  constructor(private size: number) {}

  setSize(size: number): void {
    this.size = size;
  }

  getArea(): number {
    return this.size * this.size;
  }
}

// Используем общий интерфейс
function printArea(shape: Shape): void {
  console.log(`Area: ${shape.getArea()}`);
}

printArea(new Rectangle(5, 10)); // ✅ Area: 50
printArea(new Square(5));        // ✅ Area: 25
```

### 🎯 LSP в TypeScript: Type Guards

```typescript
// Используем Type Guards для безопасной работы с подтипами
interface Animal {
  name: string;
  makeSound(): void;
}

interface Bird extends Animal {
  fly(): void;
}

interface Fish extends Animal {
  swim(): void;
}

// Type Guard
function isBird(animal: Animal): animal is Bird {
  return 'fly' in animal;
}

function isFish(animal: Animal): animal is Fish {
  return 'swim' in animal;
}

// Функция работает с любым Animal
function handleAnimal(animal: Animal): void {
  animal.makeSound(); // Общее поведение

  // Специфичное поведение через Type Guard
  if (isBird(animal)) {
    animal.fly();
  } else if (isFish(animal)) {
    animal.swim();
  }
}
```

### 💡 LSP и Active Record

```typescript
// ПРОБЛЕМА: Active Record создает слишком общий базовый класс
class Record {
  async save(): Promise<void> {
    // Общая логика сохранения
  }

  async delete(): Promise<void> {
    // Общая логика удаления
  }

  async load(id: number): Promise<void> {
    // Общая логика загрузки
  }
}

// Все сущности наследуют Record
class User extends Record {
  // User имеет свою специфическую логику
  // но вынужден переопределять save/delete/load
}

class Order extends Record {
  // Order может иметь совершенно другую логику сохранения
  // (например, в несколько таблиц)
}
```

**Проблема:** Слишком общий базовый класс Record создает ситуацию, где наследники вынуждены сильно менять поведение, нарушая LSP.

### 🔑 Ключевые выводы по LSP

> **"Наследник не должен ломать ожидания базового типа"**

1. **Наследник не должен** добавлять новые исключения
2. **Наследник не должен** усиливать предусловия
3. **Наследник не должен** ослаблять постусловия
4. **Используйте** композицию вместо наследования где возможно
5. **Применяйте** Type Guards в TypeScript для безопасной работы с подтипами
6. **Избегайте** слишком общих базовых классов

---

## ISP - Interface Segregation Principle

> **Принцип разделения интерфейсов**
>
> Клиенты не должны зависеть от **интерфейсов, которые они не используют**.

### 📚 Определение и суть принципа

**ISP утверждает:** Лучше иметь **много маленьких специализированных интерфейсов**, чем один большой универсальный.

**Почему это важно:**
- Уменьшается связанность
- Упрощается реализация интерфейса
- Легче тестировать
- Понятнее назначение каждого интерфейса

### 🔴 Особенность JavaScript: отсутствие интерфейсов

В JavaScript нет интерфейсов в классическом понимании (как в Java/C#), но:

1. **Duck Typing** - "если это ходит как утка и крякает как утка..."
2. **TypeScript интерфейсы** - compile-time проверки
3. **Контракты** - неявные соглашения о структуре объектов

### ❌ Пример нарушения ISP

```typescript
// ПЛОХО: один большой интерфейс
interface DataSource {
  // Операции чтения
  read(id: string): Promise<Data>;
  readAll(): Promise<Data[]>;
  query(filter: Filter): Promise<Data[]>;

  // Операции записи
  write(data: Data): Promise<void>;
  update(id: string, data: Partial<Data>): Promise<void>;
  delete(id: string): Promise<void>;

  // Операции массовой обработки
  bulkInsert(data: Data[]): Promise<void>;
  bulkUpdate(updates: Update[]): Promise<void>;
  bulkDelete(ids: string[]): Promise<void>;

  // Операции бэкапа
  backup(): Promise<Backup>;
  restore(backup: Backup): Promise<void>;

  // Операции мониторинга
  getStats(): Promise<Stats>;
  healthCheck(): Promise<boolean>;
}

// Проблема: ReadOnlyCache должен реализовать ВСЕ методы
class ReadOnlyCache implements DataSource {
  read(id: string): Promise<Data> {
    // Реализация
  }

  readAll(): Promise<Data[]> {
    // Реализация
  }

  // Эти методы не нужны для ReadOnly, но приходится реализовывать!
  write(data: Data): Promise<void> {
    throw new Error('Read-only cache does not support write operations');
  }

  update(id: string, data: Partial<Data>): Promise<void> {
    throw new Error('Read-only cache does not support update operations');
  }

  delete(id: string): Promise<void> {
    throw new Error('Read-only cache does not support delete operations');
  }

  // ... и еще куча методов, которые не нужны
}
```

**Проблемы:**
- ❌ Клиент зависит от методов, которые не использует
- ❌ Приходится писать заглушки или бросать исключения
- ❌ Интерфейс слишком сложный для понимания

### ✅ Решение: разделение интерфейсов

```typescript
// ХОРОШО: много маленьких интерфейсов
interface Readable<T> {
  read(id: string): Promise<T>;
  readAll(): Promise<T[]>;
}

interface Queryable<T> {
  query(filter: Filter): Promise<T[]>;
}

interface Writable<T> {
  write(data: T): Promise<void>;
  update(id: string, data: Partial<T>): Promise<void>;
  delete(id: string): Promise<void>;
}

interface BulkWritable<T> {
  bulkInsert(data: T[]): Promise<void>;
  bulkUpdate(updates: Update[]): Promise<void>;
  bulkDelete(ids: string[]): Promise<void>;
}

interface Backupable {
  backup(): Promise<Backup>;
  restore(backup: Backup): Promise<void>;
}

interface Monitorable {
  getStats(): Promise<Stats>;
  healthCheck(): Promise<boolean>;
}

// Теперь можем комбинировать только нужные интерфейсы
class ReadOnlyCache<T> implements Readable<T>, Queryable<T> {
  async read(id: string): Promise<T> {
    // Реализация чтения из кеша
  }

  async readAll(): Promise<T[]> {
    // Реализация получения всех данных
  }

  async query(filter: Filter): Promise<T[]> {
    // Реализация запроса с фильтром
  }
}

// Полнофункциональное хранилище
class Database<T> implements
  Readable<T>,
  Queryable<T>,
  Writable<T>,
  BulkWritable<T>,
  Backupable,
  Monitorable
{
  // Реализует все интерфейсы
}

// Функция, которая работает только с чтением
function loadData<T>(source: Readable<T>, id: string): Promise<T> {
  return source.read(id);
}

// Можем передать любую реализацию Readable
loadData(new ReadOnlyCache<User>(), '123');
loadData(new Database<User>(), '123');
```

**Преимущества:**
- ✅ Каждый интерфейс специализирован
- ✅ Клиент зависит только от того, что использует
- ✅ Легко реализовать частичную функциональность

### 🎯 ISP в Node.js: Stream интерфейсы

Node.js отлично демонстрирует ISP через разделение Stream интерфейсов:

```typescript
import { Readable, Writable, Duplex, Transform } from 'stream';

// Маленькие специализированные интерфейсы
interface ReadableStream {
  read(size?: number): any;
  pipe<T extends Writable>(destination: T): T;
  on(event: 'data', listener: (chunk: any) => void): this;
  on(event: 'end', listener: () => void): this;
}

interface WritableStream {
  write(chunk: any, callback?: (error?: Error) => void): boolean;
  end(callback?: () => void): void;
  on(event: 'finish', listener: () => void): this;
}

// Duplex = Readable + Writable (композиция интерфейсов)
interface DuplexStream extends ReadableStream, WritableStream {}

// Transform = специальный вид Duplex
interface TransformStream extends DuplexStream {
  _transform(chunk: any, encoding: string, callback: Function): void;
}

// Примеры использования
import fs from 'fs';
import zlib from 'zlib';

// Функция, которая нужна только Readable
function consumeReadable(stream: Readable): void {
  stream.on('data', (chunk) => {
    console.log(`Received ${chunk.length} bytes`);
  });
}

// Функция, которой нужен только Writable
function produceToWritable(stream: Writable): void {
  stream.write('Hello, World!');
  stream.end();
}

// Использование
const readStream = fs.createReadStream('file.txt');
consumeReadable(readStream); // ✅ Работает

const writeStream = fs.createWriteStream('output.txt');
produceToWritable(writeStream); // ✅ Работает

// Duplex stream можно использовать в обоих случаях
const gzip = zlib.createGzip();
consumeReadable(gzip);  // ✅ Работает как Readable
produceToWritable(gzip); // ✅ Работает как Writable
```

### 💡 ISP в JavaScript: Iterator и Iterable

```typescript
// Два маленьких интерфейса вместо одного большого
interface Iterable<T> {
  [Symbol.iterator](): Iterator<T>;
}

interface Iterator<T> {
  next(): IteratorResult<T>;
}

interface IteratorResult<T> {
  done: boolean;
  value: T;
}

// Класс может реализовать только Iterable
class Range implements Iterable<number> {
  constructor(
    private start: number,
    private end: number
  ) {}

  [Symbol.iterator](): Iterator<number> {
    let current = this.start;
    const end = this.end;

    return {
      next(): IteratorResult<number> {
        if (current <= end) {
          return { done: false, value: current++ };
        }
        return { done: true, value: undefined as any };
      }
    };
  }
}

// Использование
for (const num of new Range(1, 5)) {
  console.log(num); // 1, 2, 3, 4, 5
}
```

### 🔑 Ключевые выводы по ISP

> **"Много маленьких интерфейсов лучше одного большого"**

1. **Разделяйте** большие интерфейсы на маленькие специализированные
2. **В JavaScript** используйте duck typing и TypeScript интерфейсы
3. **Примеры в Node.js:** Stream interfaces, Iterator/Iterable
4. **Клиент** должен зависеть только от методов, которые использует
5. **Композиция** интерфейсов лучше одного универсального

---

## DIP - Dependency Inversion Principle

> **Принцип инверсии зависимостей**
>
> 1. Модули верхнего уровня не должны зависеть от модулей нижнего уровня. **Оба должны зависеть от абстракций**.
> 2. Абстракции не должны зависеть от деталей. **Детали должны зависеть от абстракций**.

### 📚 Определение и суть принципа

**DIP утверждает:** Зависимость должна быть направлена на **абстракции** (интерфейсы), а не на **конкретные реализации**.

**Почему это важно:**
- Упрощается замена реализаций
- Код становится более гибким
- Легче тестировать (можно использовать mock)
- Снижается связанность между модулями

### 🔴 Проблема в JavaScript: отсутствие интерфейсов

В JavaScript/TypeScript нет интерфейсов в runtime (только в compile-time), поэтому:

1. **TypeScript интерфейсы** компилируются в JavaScript и исчезают
2. **Duck typing** - неявные контракты
3. **Dependency Injection** работает через передачу зависимостей

### ❌ Пример нарушения DIP

```typescript
// ПЛОХО: жесткая зависимость от конкретной реализации
class MySQLDatabase {
  async query(sql: string, params: any[]): Promise<any> {
    // Логика работы с MySQL
  }
}

class UserService {
  private db: MySQLDatabase;

  constructor() {
    // Жесткая зависимость - создаем внутри
    this.db = new MySQLDatabase();
  }

  async getUser(id: string): Promise<User> {
    const result = await this.db.query(
      'SELECT * FROM users WHERE id = ?',
      [id]
    );
    return result[0];
  }

  async createUser(user: User): Promise<void> {
    await this.db.query(
      'INSERT INTO users (name, email) VALUES (?, ?)',
      [user.name, user.email]
    );
  }
}
```

**Проблемы:**
- ❌ UserService жестко связан с MySQLDatabase
- ❌ Невозможно заменить БД без изменения UserService
- ❌ Сложно тестировать - нужна реальная БД
- ❌ Нарушается OCP - при смене БД придется модифицировать UserService

### ✅ Решение: Dependency Injection + интерфейсы

```typescript
// ХОРОШО: зависимость от абстракции
interface Database {
  query(sql: string, params: any[]): Promise<any>;
}

// Конкретная реализация для MySQL
class MySQLDatabase implements Database {
  async query(sql: string, params: any[]): Promise<any> {
    // Логика работы с MySQL
    console.log('MySQL query:', sql, params);
    return [];
  }
}

// Конкретная реализация для PostgreSQL
class PostgresDatabase implements Database {
  async query(sql: string, params: any[]): Promise<any> {
    // Логика работы с PostgreSQL
    console.log('Postgres query:', sql, params);
    return [];
  }
}

// Mock для тестирования
class MockDatabase implements Database {
  private data: Map<string, User> = new Map();

  async query(sql: string, params: any[]): Promise<any> {
    // Простая реализация для тестов
    if (sql.includes('SELECT')) {
      const id = params[0];
      return [this.data.get(id)];
    }
    if (sql.includes('INSERT')) {
      const [name, email] = params;
      this.data.set('1', { id: '1', name, email });
      return [];
    }
    return [];
  }
}

// Сервис зависит от абстракции, а не от конкретной реализации
class UserService {
  constructor(private db: Database) {} // Dependency Injection!

  async getUser(id: string): Promise<User> {
    const result = await this.db.query(
      'SELECT * FROM users WHERE id = ?',
      [id]
    );
    return result[0];
  }

  async createUser(user: User): Promise<void> {
    await this.db.query(
      'INSERT INTO users (name, email) VALUES (?, ?)',
      [user.name, user.email]
    );
  }
}

// Использование в продакшене
const mysqlDb = new MySQLDatabase();
const userService = new UserService(mysqlDb);

// Легко заменить на PostgreSQL
const postgresDb = new PostgresDatabase();
const userServicePg = new UserService(postgresDb);

// Легко тестировать с mock
const mockDb = new MockDatabase();
const userServiceTest = new UserService(mockDb);
```

**Преимущества:**
- ✅ UserService не зависит от конкретной БД
- ✅ Легко заменить реализацию
- ✅ Легко тестировать с mock
- ✅ Соблюдается OCP

### 🎯 DI Container в Node.js

```typescript
// Простой DI Container
class Container {
  private services = new Map<string, any>();

  // Регистрация фабрики
  register<T>(name: string, factory: (container: Container) => T): void {
    this.services.set(name, factory);
  }

  // Резолв зависимости
  resolve<T>(name: string): T {
    const factory = this.services.get(name);
    if (!factory) {
      throw new Error(`Service ${name} not found`);
    }
    return factory(this);
  }
}

// Использование
const container = new Container();

// Регистрируем зависимости
container.register('database', () => new MySQLDatabase());

container.register('userRepository', (c) =>
  new UserRepository(c.resolve('database'))
);

container.register('userService', (c) =>
  new UserService(c.resolve('userRepository'))
);

// Резолвим зависимости
const userService = container.resolve<UserService>('userService');
```

### 💡 DIP в функциональном программировании

```typescript
// Функциональный подход к DI
type Logger = (message: string) => void;
type Database = {
  query: (sql: string, params: any[]) => Promise<any>;
};

// Функция зависит от абстракций (функций)
const createUserService = (db: Database, logger: Logger) => ({
  async getUser(id: string): Promise<User> {
    logger(`Getting user ${id}`);
    const result = await db.query(
      'SELECT * FROM users WHERE id = ?',
      [id]
    );
    return result[0];
  },

  async createUser(user: User): Promise<void> {
    logger(`Creating user ${user.name}`);
    await db.query(
      'INSERT INTO users (name, email) VALUES (?, ?)',
      [user.name, user.email]
    );
  }
});

// Конкретные реализации
const consoleLogger: Logger = (message) => console.log(message);
const fileLogger: Logger = (message) => fs.appendFileSync('log.txt', message + '\n');

const mysqlDb: Database = {
  query: async (sql, params) => { /* ... */ }
};

// Использование
const userService = createUserService(mysqlDb, consoleLogger);

// Легко заменить логгер
const userServiceWithFileLog = createUserService(mysqlDb, fileLogger);
```

### 🔑 Ключевые выводы по DIP

> **"Зависимость от абстракций, а не от конкретных реализаций"**

1. **Используйте** Dependency Injection для внедрения зависимостей
2. **Зависьте** от интерфейсов/абстракций, а не от конкретных классов
3. **В JavaScript** используйте duck typing или TypeScript интерфейсы
4. **Применяйте** DI Container для управления зависимостями
5. **В функциональном стиле** передавайте функции как зависимости

---

## Применение в асинхронном программировании

Асинхронность добавляет новый уровень сложности к применению SOLID принципов. Рассмотрим специфику работы с async/await, Promise и callback patterns.

### 🎯 SRP в асинхронном коде

```typescript
// ❌ ПЛОХО: функция делает слишком много
async function processUser(userId: string) {
  // Доступ к БД
  const user = await db.getUser(userId);

  // Валидация
  if (!user.email) {
    throw new Error('User has no email');
  }

  // Отправка email
  await sendEmail(user.email, 'Welcome!');

  // Логирование
  await logAction('user_processed', userId);

  // Обновление статистики
  await incrementUserProcessedCount();

  return user;
}
```

**Проблемы:**
- ❌ Смешивает доступ к БД, валидацию, отправку email и логирование
- ❌ Сложно тестировать
- ❌ Любое изменение требует модификации функции

```typescript
// ✅ ХОРОШО: разделение ответственности
async function getUserFromDb(userId: string): Promise<User> {
  return await db.getUser(userId);
}

function validateUser(user: User): void {
  if (!user.email) {
    throw new Error('User has no email');
  }
}

async function notifyUser(user: User): Promise<void> {
  await sendEmail(user.email, 'Welcome!');
}

async function logUserAction(action: string, userId: string): Promise<void> {
  await logAction(action, userId);
}

async function updateStatistics(): Promise<void> {
  await incrementUserProcessedCount();
}

// Композиция - каждая функция имеет одну ответственность
async function processUser(userId: string): Promise<User> {
  const user = await getUserFromDb(userId);
  validateUser(user);

  // Параллельное выполнение независимых операций
  await Promise.all([
    notifyUser(user),
    logUserAction('user_processed', userId),
    updateStatistics()
  ]);

  return user;
}
```

### 🎯 Обработка ошибок в асинхронном коде

```typescript
// Разные способы обработки ошибок
interface ErrorHandler {
  handleError(error: Error): void;
}

// 1. Try-catch для async/await
async function withTryCatch(handler: ErrorHandler) {
  try {
    await someAsyncOperation();
  } catch (error) {
    handler.handleError(error as Error);
  }
}

// 2. .catch() для Promise chains
function withCatch(handler: ErrorHandler) {
  return someAsyncOperation()
    .catch(error => handler.handleError(error));
}

// 3. Error-first callbacks (старый стиль)
function withCallback(callback: (error: Error | null, result?: any) => void) {
  someOperation((error, result) => {
    if (error) {
      callback(error);
    } else {
      callback(null, result);
    }
  });
}

// 4. Событие 'error' для streams
import { Readable } from 'stream';

function handleStreamErrors(stream: Readable, handler: ErrorHandler) {
  stream.on('error', (error) => {
    handler.handleError(error);
  });
}
```

### 🎯 Async Iterator Pattern

```typescript
// Интерфейс для асинхронного итератора
interface AsyncIterable<T> {
  [Symbol.asyncIterator](): AsyncIterator<T>;
}

interface AsyncIterator<T> {
  next(): Promise<IteratorResult<T>>;
}

// Реализация курсора для БД
class DatabaseCursor<T> implements AsyncIterable<T> {
  constructor(
    private query: string,
    private db: Database
  ) {}

  async *[Symbol.asyncIterator](): AsyncIterator<T> {
    let offset = 0;
    const limit = 100;

    while (true) {
      const results = await this.db.query(
        `${this.query} LIMIT ${limit} OFFSET ${offset}`,
        []
      );

      if (results.length === 0) break;

      for (const result of results) {
        yield result;
      }

      offset += limit;
    }
  }
}

// Использование
async function processAllUsers() {
  const cursor = new DatabaseCursor<User>('SELECT * FROM users', db);

  for await (const user of cursor) {
    await processUser(user.id);
  }
}
```

### 🎯 Promise композиция

```typescript
// Композиция асинхронных операций
class AsyncPipeline<T> {
  constructor(private value: Promise<T>) {}

  pipe<U>(fn: (value: T) => Promise<U>): AsyncPipeline<U> {
    return new AsyncPipeline(this.value.then(fn));
  }

  async execute(): Promise<T> {
    return await this.value;
  }
}

// Использование
const result = await new AsyncPipeline(Promise.resolve('user123'))
  .pipe(getUserFromDb)
  .pipe(async (user) => {
    validateUser(user);
    return user;
  })
  .pipe(notifyUser)
  .execute();
```

### 🎯 Retry Pattern с экспоненциальной задержкой

```typescript
// Функция retry - одна ответственность
async function retry<T>(
  fn: () => Promise<T>,
  options: {
    maxAttempts: number;
    delay: number;
    backoff: number;
  }
): Promise<T> {
  let lastError: Error;

  for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      if (attempt < options.maxAttempts) {
        const delay = options.delay * Math.pow(options.backoff, attempt - 1);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError!;
}

// Использование
const user = await retry(
  () => getUserFromDb('123'),
  { maxAttempts: 3, delay: 1000, backoff: 2 }
);
```

### 🔑 Ключевые выводы по async программированию

1. **SRP** остается актуальным - одна async функция = одна ответственность
2. **Обработка ошибок** должна быть согласованной (try/catch, .catch(), events)
3. **Promise.all()** для параллельных операций
4. **Async generators** для потоковой обработки данных
5. **Retry/Circuit Breaker** паттерны для устойчивости

---

## Специфика Node.js

Node.js имеет уникальную архитектуру, которая влияет на применение SOLID принципов.

### 🎯 Модульная система и SRP

```typescript
// Хорошая структура модулей по SRP
// user/
//   ├── user.model.ts         - Модель данных
//   ├── user.repository.ts    - Работа с БД
//   ├── user.service.ts       - Бизнес-логика
//   ├── user.validator.ts     - Валидация
//   ├── user.controller.ts    - HTTP endpoints
//   └── index.ts              - Экспорты

// user.model.ts
export class User {
  constructor(
    public readonly id: string,
    public name: string,
    public email: string
  ) {}
}

// user.repository.ts
export interface IUserRepository {
  save(user: User): Promise<void>;
  findById(id: string): Promise<User | null>;
}

export class UserRepository implements IUserRepository {
  constructor(private db: Database) {}
  // Реализация
}

// user.service.ts
export class UserService {
  constructor(private repository: IUserRepository) {}

  async createUser(name: string, email: string): Promise<User> {
    const user = new User(generateId(), name, email);
    await this.repository.save(user);
    return user;
  }
}

// user.validator.ts
export class UserValidator {
  validateEmail(email: string): boolean {
    return /\S+@\S+\.\S+/.test(email);
  }

  validateName(name: string): boolean {
    return name.length >= 2 && name.length <= 50;
  }
}

// user.controller.ts
export class UserController {
  constructor(
    private service: UserService,
    private validator: UserValidator
  ) {}

  async createUser(req: Request, res: Response): Promise<void> {
    const { name, email } = req.body;

    if (!this.validator.validateEmail(email)) {
      res.status(400).json({ error: 'Invalid email' });
      return;
    }

    if (!this.validator.validateName(name)) {
      res.status(400).json({ error: 'Invalid name' });
      return;
    }

    const user = await this.service.createUser(name, email);
    res.status(201).json(user);
  }
}

// index.ts - композиция зависимостей
import { Database } from '../database';

const db = new Database();
const repository = new UserRepository(db);
const service = new UserService(repository);
const validator = new UserValidator();
const controller = new UserController(service, validator);

export { User, UserService, UserController };
```

### 🎯 Middleware Pattern и ISP

```typescript
import express, { Request, Response, NextFunction } from 'express';

// Каждый middleware - маленький специализированный интерфейс
type Middleware = (req: Request, res: Response, next: NextFunction) => void;

// Логирование
const logger: Middleware = (req, res, next) => {
  console.log(`${new Date().toISOString()} ${req.method} ${req.path}`);
  next();
};

// Аутентификация
const authenticate: Middleware = (req, res, next) => {
  const token = req.headers.authorization;

  if (!token) {
    res.status(401).json({ error: 'Unauthorized' });
    return;
  }

  // Проверка токена
  next();
};

// Валидация тела запроса
const validateBody = (schema: any): Middleware =>
  (req, res, next) => {
    // Валидация с использованием schema
    next();
  };

// CORS
const cors: Middleware = (req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', '*');
  next();
};

// Применение - композиция маленьких middleware
const app = express();

app.use(logger);        // ISP - маленький интерфейс
app.use(cors);          // ISP - маленький интерфейс
app.use(authenticate);  // ISP - маленький интерфейс

app.post('/users',
  validateBody(userSchema),  // ISP - маленький интерфейс
  (req, res) => {
    // Обработчик
  }
);
```

### 🎯 Event-driven архитектура

```typescript
import { EventEmitter } from 'events';

// Интерфейс событий
interface UserEvents {
  created: (user: User) => void;
  updated: (user: User) => void;
  deleted: (userId: string) => void;
}

// Type-safe EventEmitter
class TypedEventEmitter<Events> {
  private emitter = new EventEmitter();

  on<K extends keyof Events>(event: K, listener: Events[K]): void {
    this.emitter.on(event as string, listener as any);
  }

  emit<K extends keyof Events>(
    event: K,
    ...args: Parameters<Events[K]>
  ): void {
    this.emitter.emit(event as string, ...args);
  }
}

// Сервис с событиями
class UserService extends TypedEventEmitter<UserEvents> {
  constructor(private repository: IUserRepository) {
    super();
  }

  async createUser(name: string, email: string): Promise<User> {
    const user = new User(generateId(), name, email);
    await this.repository.save(user);

    // Emit событие
    this.emit('created', user);

    return user;
  }
}

// Подписчики - каждый имеет одну ответственность (SRP)
class EmailNotificationService {
  constructor(userService: UserService) {
    userService.on('created', (user) => {
      this.sendWelcomeEmail(user);
    });
  }

  private async sendWelcomeEmail(user: User): Promise<void> {
    console.log(`Sending welcome email to ${user.email}`);
  }
}

class AnalyticsService {
  constructor(userService: UserService) {
    userService.on('created', (user) => {
      this.trackUserCreation(user);
    });
  }

  private async trackUserCreation(user: User): Promise<void> {
    console.log(`Tracking user creation: ${user.id}`);
  }
}

// Использование
const userService = new UserService(repository);
new EmailNotificationService(userService);
new AnalyticsService(userService);

await userService.createUser('John', 'john@example.com');
// Автоматически вызовутся обработчики событий
```

### 🎯 Stream Pipeline

```typescript
import { Transform, pipeline } from 'stream';
import fs from 'fs';
import zlib from 'zlib';

// Каждый transform - одна ответственность (SRP)
class JsonParseTransform extends Transform {
  _transform(chunk: any, encoding: string, callback: Function): void {
    try {
      const data = JSON.parse(chunk.toString());
      this.push(data);
      callback();
    } catch (error) {
      callback(error);
    }
  }
}

class FilterTransform extends Transform {
  constructor(private predicate: (data: any) => boolean) {
    super({ objectMode: true });
  }

  _transform(chunk: any, encoding: string, callback: Function): void {
    if (this.predicate(chunk)) {
      this.push(chunk);
    }
    callback();
  }
}

class MapTransform extends Transform {
  constructor(private mapper: (data: any) => any) {
    super({ objectMode: true });
  }

  _transform(chunk: any, encoding: string, callback: Function): void {
    this.push(this.mapper(chunk));
    callback();
  }
}

// Композиция stream pipeline
pipeline(
  fs.createReadStream('users.ndjson'),
  new JsonParseTransform(),
  new FilterTransform((user) => user.age >= 18),
  new MapTransform((user) => ({ id: user.id, name: user.name })),
  fs.createWriteStream('adults.json'),
  (err) => {
    if (err) console.error('Pipeline failed:', err);
    else console.log('Pipeline succeeded');
  }
);
```

### 🔑 Ключевые выводы по Node.js

1. **Модульная система** - один модуль = одна ответственность (SRP)
2. **Middleware** - отличный пример ISP (маленькие интерфейсы)
3. **Event-driven** - Observer pattern встроен в платформу
4. **Streams** - ISP через разделение Readable/Writable/Duplex
5. **Dependency Injection** легко реализуется через конструкторы

---

## Практические рекомендации

### ✅ DO - Что делать

1. **Разделяйте бизнес-логику и инфраструктуру**
   ```typescript
   // ✅ ХОРОШО
   class OrderService {
     constructor(private repository: IOrderRepository) {}
   }
   ```

2. **Используйте Dependency Injection**
   ```typescript
   // ✅ ХОРОШО
   class UserService {
     constructor(
       private repository: IUserRepository,
       private emailService: IEmailService
     ) {}
   }
   ```

3. **Создавайте маленькие специализированные интерфейсы**
   ```typescript
   // ✅ ХОРОШО
   interface Readable<T> {
     read(id: string): Promise<T>;
   }

   interface Writable<T> {
     write(data: T): Promise<void>;
   }
   ```

4. **Применяйте композицию вместо наследования**
   ```typescript
   // ✅ ХОРОШО
   class UserService {
     constructor(
       private validator: UserValidator,
       private repository: UserRepository
     ) {}
   }
   ```

5. **Используйте TypeScript для явных контрактов**
   ```typescript
   // ✅ ХОРОШО
   interface IUserService {
     createUser(name: string, email: string): Promise<User>;
     getUser(id: string): Promise<User | null>;
   }
   ```

6. **Обрабатывайте ошибки согласованно**
   ```typescript
   // ✅ ХОРОШО - единый подход к ошибкам
   async function handleRequest() {
     try {
       await operation();
     } catch (error) {
       logger.error(error);
       throw new AppError('Operation failed', error);
     }
   }
   ```

7. **Пишите тесты для проверки принципов**
   ```typescript
   // ✅ ХОРОШО - тесты помогают соблюдать SOLID
   describe('UserService', () => {
     it('should use injected repository', async () => {
       const mockRepo = new MockUserRepository();
       const service = new UserService(mockRepo);
       // Тест
     });
   });
   ```

### ❌ DON'T - Что НЕ делать

1. **НЕ используйте Active Record**
   ```typescript
   // ❌ ПЛОХО
   class User {
     async save() { /* БД */ }
     async delete() { /* БД */ }
   }
   ```

2. **НЕ смешивайте разные ответственности**
   ```typescript
   // ❌ ПЛОХО
   async function processUser(id: string) {
     const user = await db.getUser(id);
     await sendEmail(user.email);
     await logAction('processed');
     return user;
   }
   ```

3. **НЕ создавайте "божественные объекты" (God Objects)**
   ```typescript
   // ❌ ПЛОХО
   class ApplicationManager {
     handleAuth() {}
     manageDB() {}
     sendEmails() {}
     generateReports() {}
     // ... 100 методов
   }
   ```

4. **НЕ делайте жесткие связи между модулями**
   ```typescript
   // ❌ ПЛОХО
   class UserService {
     constructor() {
       this.db = new MySQLDatabase(); // Жесткая связь
     }
   }
   ```

5. **НЕ игнорируйте async/await best practices**
   ```typescript
   // ❌ ПЛОХО - не await в async функции
   async function badAsync() {
     someAsyncOperation(); // Забыли await
   }
   ```

6. **НЕ переопределяйте методы с изменением контракта**
   ```typescript
   // ❌ ПЛОХО - нарушение LSP
   class Square extends Rectangle {
     setWidth(w: number) {
       this.width = w;
       this.height = w; // Меняем контракт!
     }
   }
   ```

7. **НЕ думайте, что SOLID только для ООП**
   ```typescript
   // ✅ SOLID работает в функциональном стиле
   const createUserService = (repo, emailService) => ({
     createUser: async (name, email) => {
       // Функциональный подход к SOLID
     }
   });
   ```

### 📊 Сравнение Java vs JavaScript культур

| Аспект | Java | JavaScript/Node.js |
|--------|------|-------------------|
| **Observer Pattern** | 2 интерфейса (Subject, Observer) | EventEmitter + функции |
| **Интерфейсы** | Compile-time и runtime | Только compile-time (TS) |
| **Наследование** | Классовое | Прототипное (+ классы как сахар) |
| **Зависимости** | Через интерфейсы | Через duck typing + DI |
| **Асинхронность** | CompletableFuture (Java 8+) | Промисы, async/await (нативно) |
| **Культура** | Строгая типизация | Динамика + гибкость |

**Почему разница?**
- JavaScript имеет функции первого класса
- Event-driven модель встроена в язык
- Культура сформирована браузером и Node.js
- В Java долго не было lambda, сформировалась культура классов

### 🎯 Чек-лист проверки SOLID

При написании кода задавайте себе вопросы:

**SRP:**
- [ ] Этот класс/функция делает только одну вещь?
- [ ] Есть ли только одна причина для изменения этого кода?
- [ ] Не смешиваю ли я бизнес-логику и инфраструктуру?

**OCP:**
- [ ] Могу ли я добавить новую функциональность без модификации существующего кода?
- [ ] Использую ли я интерфейсы для расширения?

**LSP:**
- [ ] Можно ли заменить базовый класс наследником без поломок?
- [ ] Не добавляет ли наследник новые исключения?
- [ ] Сохраняется ли контракт базового класса?

**ISP:**
- [ ] Не зависит ли клиент от методов, которые не использует?
- [ ] Можно ли разбить интерфейс на более маленькие?

**DIP:**
- [ ] Зависит ли код от абстракций, а не от конкретных реализаций?
- [ ] Используется ли Dependency Injection?
- [ ] Легко ли заменить реализацию?

---

## Заключение

### 🎓 Основные выводы

1. **SOLID применим к JavaScript/TypeScript**, но требует адаптации к культуре языка

2. **Active Record - антипаттерн** в контексте современной разработки
   - Используйте Repository Pattern
   - Разделяйте данные и логику доступа к БД

3. **SRP - самый важный принцип**
   - Один класс/функция = одна ответственность
   - Разделяйте предметную область и инфраструктуру

4. **Асинхронность добавляет сложность**
   - SRP остается актуальным для async функций
   - Нужны специальные паттерны (retry, circuit breaker)

5. **Event-driven архитектура** - естественная часть JavaScript
   - EventEmitter - встроенный Observer Pattern
   - Middleware - пример ISP

6. **Культура языка важна**
   - Не копируйте Java подходы слепо
   - Используйте функции первого класса
   - Применяйте композицию вместо наследования

7. **Композиция > Наследование** в JavaScript
   - Гибкость через DI
   - Переиспользование через функции

8. **TypeScript помогает** с явными интерфейсами и контрактами
   - Compile-time проверки
   - Документирование API

9. **Маленькие модули** лучше больших
   - Один файл = одна ответственность
   - Легче поддерживать и тестировать

10. **Практика важнее теории**
    - Используйте реальные примеры
    - Пишите тесты
    - Рефакторите регулярно

### 📚 Дальнейшее изучение

**Паттерны проектирования:**
- Repository Pattern
- Factory Pattern
- Strategy Pattern
- Observer Pattern
- Decorator Pattern

**Архитектурные подходы:**
- Clean Architecture
- Hexagonal Architecture
- Domain-Driven Design (DDD)
- CQRS и Event Sourcing

**Тестирование:**
- Unit тестирование SOLID кода
- Mocking зависимостей
- Integration тесты

**Продвинутые темы:**
- Функциональное программирование и SOLID
- Микросервисная архитектура
- Reactive programming

### 💡 Финальные советы

1. **Не перестарайтесь** - SOLID это инструмент, а не догма
2. **Баланс** между чистотой кода и прагматизмом
3. **Рефакторинг** - постепенное улучшение, а не переписывание с нуля
4. **Команда** - договоритесь о соглашениях и следуйте им
5. **Code Review** - лучший способ научиться и обучить других

---

> **"Любой дурак может написать код, который поймет компьютер. Хорошие программисты пишут код, который понимают люди."**
>
> — Martin Fowler

---

**Автор конспекта:** На основе лекции по SOLID принципам
**Дата:** 2025-12-06
**Версия:** 1.0

**Материалы для изучения:**
- [Clean Code](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) - Robert C. Martin
- [Design Patterns: Elements of Reusable Object-Oriented Software](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612) - Gang of Four
- [Node.js Design Patterns](https://www.nodejsdesignpatterns.com/) - Mario Casciaro
