# Конспект семинара Тимура Шемсединова: Переосмысление паттернов (GRASP, SOLID, GoF) в JavaScript

## Введение

Основная идея семинара: **использовать паттерны проектирования нужно осторожно**. Нельзя слепо копировать паттерны из других языков программирования в JavaScript без учета особенностей самого JS и его runtime (V8).

Ключевая посылка: паттерны - это инструменты, а не закон природы. Их применение должно быть обоснованным, а не автоматическим.

---

## Часть 1: Проблема слепого применения паттернов

### Конфликт между читаемостью и производительностью

**Проблема**: Книги по "чистому коду" (Uncle Bob) пропагандируют применение паттернов для читаемости, но не учитывают производительность, особенно на серверной части.

**Исторический пример**: Малоизвестный разработчик провел сравнение:
- Взял классический пример из "Clean Code" (на C++)
- Реализовал его по паттерну (с классами, абстракциями)
- Реализовал то же самое простым кодом (процедурным)
- **Результат**: разница в производительности примерно в 200-400 раз на серверных нагрузках

**Вывод**: Uncle Bob рассуждает о "сферическом коде в вакууме", как об искусстве, а разработчики механически копируют его рекомендации в production, где нагрузки могут быть колоссальными (сотни и тысячи одновременных пользователей).

### Важный принцип: знание языка программирования

> "Если вы программисты и используете язык программирования, то знайте, что стоит за каждой инструкцией, которую вы используете"

Нельзя просто копировать код от одного языка к другому. Например:
- Python: скобочки убрали из Java → новый язык
- Lisp: добавили скобочки к Python → древний (но первый) функциональный язык (1959)

Каждый язык имеет свою философию, и реализация одного и того же паттерна может быть совершенно разной на уровне компилятора или интерпретатора.

---

## Часть 2: V8 Engine и объекты в JavaScript

### Формы объектов в V8

V8 (JavaScript engine Chrome и Node.js) использует концепцию **форм объектов** (shapes/hidden classes) для оптимизации.

#### 1. **Мономорфная форма (Monomorphic)**
- V8 точно знает, какой формы будет объект
- Объект имеет фиксированный набор свойств
- Свойства не добавляются, не удаляются, не изменяют тип
- **Это максимально эффективно для V8**

```javascript
class Timer {
  constructor(interval) {
    this.interval = interval;
    this.callback = null;
  }
}

// V8 создаст мономорфную форму для Timer
const timer = new Timer(1000); // Всегда один тип объекта
```

#### 2. **Полиморфная форма (Polymorphic)**
- Объекты могут быть разных форм
- V8 добавляет проверки: "если форма A, то делай X; если форма B, то делай Y"
- Максимум проверок ~16-32 различные формы
- **Медленнее мономорфной, но приемлемо**

```javascript
class Interval {
  constructor(interval) {
    this.interval = interval;
  }
}

// Если конструктор может вернуть разные формы объектов
// то V8 переходит в полиморфный режим
```

#### 3. **Словарный режим (Megamorphic/Dictionary)**
- Более 32 разных форм объектов
- V8 переходит в режим словаря (как обычный object hash map)
- **Это самый медленный способ**

```javascript
// Если у одного класса будет 40+ разных вариантов объектов
// V8 сдается и переходит в словарный режим - работает медленнее
```

### Пример из семинара: Factory и мономорфизм

```javascript
// КЛАССИЧЕСКАЯ РЕАЛИЗАЦИЯ ИЗ КНИГ (как в Java/C++)
class Timer {
  constructor(interval) {
    this.interval = interval;
    this.callback = null;
    // ... другие тяжелые свойства
  }
  
  setCallback(cb) {
    this.callback = cb;
  }
}

class TimerFactory {
  static timers = new Map();
  
  static getTimer(interval) {
    if (!this.timers.has(interval)) {
      this.timers.set(interval, new Timer(interval));
    }
    return this.timers.get(interval);
  }
}

class Interval {
  constructor(interval, callback) {
    this.interval = interval;
    this.callback = callback;
    this.timer = TimerFactory.getTimer(interval);
  }
}

// ПРОБЛЕМА:
// 1. TimerFactory создает Timer с разными свойствами
//    в зависимости от параметров
// 2. V8 видит, что один и тот же класс возвращает
//    разные формы объектов
// 3. V8 переходит в полиморфный режим (создает проверки формы)
// 4. Это медленнее, чем когда все объекты одной формы
```

**JavaScript-way решение**:

```javascript
// OPTIMIZED VERSION ДЛЯ V8
class Timer {
  constructor(interval) {
    this.interval = interval;
    this.callback = null;
  }
}

class TimerFactory {
  static timers = new Map();
  
  static getTimer(interval) {
    if (!this.timers.has(interval)) {
      this.timers.set(interval, new Timer(interval));
    }
    return this.timers.get(interval);
  }
}

class Interval {
  constructor(interval, callback) {
    // Здесь всегда одна форма объекта - мономорфная
    this.interval = interval;
    this.callback = callback;
    this.timer = TimerFactory.getTimer(interval);
  }
}

// Или еще лучше - просто не создавайте классы где не нужны
```

---

## Часть 3: GRASP паттерны

GRASP (General Responsibility Assignment Software Patterns) - это 9 принципов распределения ответственности между объектами.

### 1. **Information Expert (Информационный эксперт)**

Ответственность за операцию должна лежать на объекте, который содержит информацию, необходимую для выполнения операции.

```javascript
// ❌ ПЛОХО - нарушение Information Expert
class Order {
  constructor(items) {
    this.items = items;
  }
}

class OrderPriceCalculator {
  calculateTotal(order) {
    return order.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

// ✅ ХОРОШО - Order сам знает, как считать стоимость
class Order {
  constructor(items) {
    this.items = items;
  }
  
  calculateTotal() {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

const order = new Order([
  { price: 100, quantity: 2 },
  { price: 50, quantity: 3 }
]);
console.log(order.calculateTotal()); // 350
```

### 2. **Creator (Создатель)**

Какой класс должен создавать новые объекты?
- Если A содержит B
- Если A агрегирует B
- Если A регулярно использует B
- Если A имеет данные для инициализации B

То A должен создавать B.

```javascript
// ❌ ПЛОХО
class DatabaseConnection {}
class UserRepository {
  constructor() {
    // Кто создает connection?
  }
}

// ✅ ХОРОШО
class DatabaseConnection {}
class UserRepository {
  constructor(connection) {
    this.connection = connection;
  }
}

// Создатель - это Application/Factory, который знает о взаимозависимостях
class Application {
  constructor() {
    this.connection = new DatabaseConnection();
    this.userRepository = new UserRepository(this.connection);
  }
}
```

### 3. **Polymorphism (Полиморфизм)**

Используйте полиморфизм для обработки альтернативных поведений вместо if-else цепочек.

```javascript
// ❌ ПЛОХО
class PaymentProcessor {
  process(payment) {
    if (payment.type === 'credit-card') {
      // логика для карты
    } else if (payment.type === 'paypal') {
      // логика для PayPal
    } else if (payment.type === 'bitcoin') {
      // логика для Bitcoin
    }
  }
}

// ✅ ХОРОШО
class CreditCardPayment {
  process() {
    // логика для карты
  }
}

class PayPalPayment {
  process() {
    // логика для PayPal
  }
}

class BitcoinPayment {
  process() {
    // логика для Bitcoin
  }
}

class PaymentProcessor {
  constructor(payment) {
    this.payment = payment;
  }
  
  process() {
    this.payment.process();
  }
}

// Использование
const paymentProcessor = new PaymentProcessor(new CreditCardPayment());
paymentProcessor.process();
```

### 4. **Pure Fabrication (Чистая выдумка)**

Когда операция не подходит ни одному реальному классу, создайте вспомогательный класс.

```javascript
// ✅ ХОРОШО
class Logger {
  log(message) {
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
}

class UserService {
  constructor(logger) {
    this.logger = logger;
  }
  
  createUser(userData) {
    this.logger.log(`Creating user: ${userData.name}`);
    // логика создания
  }
}
```

### 5. **Indirection (Косвенность)**

Добавьте промежуточный объект для разделения связанности между классами.

```javascript
// ❌ ПЛОХО - прямая зависимость
class EmailNotifier {
  notify(user) {
    // отправка email
  }
}

class UserService {
  constructor() {
    this.emailNotifier = new EmailNotifier();
  }
  
  createUser(userData) {
    // ...
    this.emailNotifier.notify(userData);
  }
}

// ✅ ХОРОШО - через интерфейс (abstraction)
class Notifier {
  notify(user) {
    throw new Error('Not implemented');
  }
}

class EmailNotifier extends Notifier {
  notify(user) {
    // отправка email
  }
}

class SMSNotifier extends Notifier {
  notify(user) {
    // отправка SMS
  }
}

class UserService {
  constructor(notifier) {
    this.notifier = notifier; // зависит от интерфейса
  }
  
  createUser(userData) {
    // ...
    this.notifier.notify(userData);
  }
}
```

### 6. **Controller (Контроллер)**

Объект, который обрабатывает системное событие и делегирует работу другим объектам.

```javascript
// ✅ ХОРОШО
class CreateUserController {
  constructor(userService) {
    this.userService = userService;
  }
  
  handle(request) {
    const userData = request.body;
    const user = this.userService.createUser(userData);
    return { status: 201, data: user };
  }
}

class UserService {
  createUser(userData) {
    // настоящая бизнес-логика
  }
}
```

### 7. **Low Coupling (Слабая связанность)**

Минимизируйте зависимости между классами.

```javascript
// ❌ ПЛОХО - сильная связанность
class Order {
  constructor(itemService, paymentService, emailService) {
    this.itemService = itemService;
    this.paymentService = paymentService;
    this.emailService = emailService;
  }
  
  complete() {
    this.itemService.decrementInventory(this.items);
    this.paymentService.charge(this.total);
    this.emailService.sendConfirmation(this.customer);
  }
}

// ✅ ХОРОШО - слабая связанность
class Order {
  constructor(items) {
    this.items = items;
  }
  
  complete() {
    return {
      inventoryDecrement: this.items,
      paymentCharge: this.total,
      customerEmail: this.customer.email
    };
  }
}

class OrderProcessor {
  constructor(itemService, paymentService, emailService) {
    this.itemService = itemService;
    this.paymentService = paymentService;
    this.emailService = emailService;
  }
  
  process(order) {
    const result = order.complete();
    this.itemService.decrementInventory(result.inventoryDecrement);
    this.paymentService.charge(result.paymentCharge);
    this.emailService.sendConfirmation(result.customerEmail);
  }
}
```

### 8. **High Cohesion (Высокая связность)**

Класс должен делать одно, и делать его хорошо. Все методы класса должны быть тесно связаны.

```javascript
// ❌ ПЛОХО - низкая связность
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
  
  validateEmail() { /* ... */ }
  sendEmail() { /* ... */ }
  encrypt() { /* ... */ }
  decode() { /* ... */ }
  queryDatabase() { /* ... */ }
  calculateSalary() { /* ... */ }
}

// ✅ ХОРОШО - высокая связность
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
  
  validate() { /* только User-специфичное */ }
}

class EmailService {
  send(email) { /* email-специфичное */ }
}

class CryptoService {
  encrypt(data) { /* криптография */ }
}
```

### 9. **Protected Variations (Защита от вариаций)**

Изолируйте точки вариации в коде - места, где может измениться логика.

```javascript
// ❌ ПЛОХО
class ReportGenerator {
  generate(format) {
    if (format === 'pdf') {
      // генерация PDF
    } else if (format === 'excel') {
      // генерация Excel
    }
  }
}

// ✅ ХОРОШО
class ReportFormatter {
  format(data) {
    throw new Error('Not implemented');
  }
}

class PDFFormatter extends ReportFormatter {
  format(data) { /* ... */ }
}

class ExcelFormatter extends ReportFormatter {
  format(data) { /* ... */ }
}

class ReportGenerator {
  constructor(formatter) {
    this.formatter = formatter;
  }
  
  generate(data) {
    return this.formatter.format(data);
  }
}
```

---

## Часть 4: SOLID принципы в JavaScript

### 1. **S - Single Responsibility Principle (SRP)**

Класс должен иметь только одну причину для изменения.

```javascript
// ❌ ПЛОХО - множественная ответственность
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
  
  save() {
    // сохранение в БД
  }
  
  sendWelcomeEmail() {
    // отправка email
  }
  
  generateReport() {
    // генерация отчета
  }
}

// ✅ ХОРОШО - одна ответственность
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

class UserRepository {
  save(user) {
    // сохранение в БД
  }
}

class EmailService {
  sendWelcomeEmail(user) {
    // отправка email
  }
}

class UserReportGenerator {
  generate(user) {
    // генерация отчета
  }
}
```

### 2. **O - Open/Closed Principle (OCP)**

Класс должен быть открыт для расширения, но закрыт для модификации.

```javascript
// ❌ ПЛОХО - закрыт для расширения
class Logger {
  log(message, type) {
    if (type === 'file') {
      // запись в файл
    } else if (type === 'console') {
      // вывод в консоль
    }
    // чтобы добавить новый тип, нужно менять класс
  }
}

// ✅ ХОРОШО - открыт для расширения
class Logger {
  constructor(handler) {
    this.handler = handler;
  }
  
  log(message) {
    this.handler.handle(message);
  }
}

class FileHandler {
  handle(message) {
    // запись в файл
  }
}

class ConsoleHandler {
  handle(message) {
    console.log(message);
  }
}

class DatabaseHandler {
  handle(message) {
    // сохранение в БД
  }
}

// Легко добавить новый handler, не меняя Logger
const logger = new Logger(new DatabaseHandler());
```

### 3. **L - Liskov Substitution Principle (LSP)**

Объекты подклассов могут заменять объекты суперклассов без нарушения функциональности.

```javascript
// ❌ ПЛОХО - нарушение LSP
class Bird {
  fly() {
    return 'Flying...';
  }
}

class Penguin extends Bird {
  fly() {
    throw new Error('Penguins cannot fly!');
  }
}

// Код, ожидающий Bird, сломается с Penguin
function makeBirdFly(bird) {
  console.log(bird.fly()); // crash с Penguin
}

// ✅ ХОРОШО - соблюдение LSP
class Bird {
  move() {
    throw new Error('Not implemented');
  }
}

class Sparrow extends Bird {
  move() {
    return 'Flying...';
  }
}

class Penguin extends Bird {
  move() {
    return 'Swimming...';
  }
}

function makeBirdMove(bird) {
  console.log(bird.move()); // работает с любым Bird
}
```

### 4. **I - Interface Segregation Principle (ISP)**

Класс не должен зависеть от интерфейсов, которые он не использует.

```javascript
// ❌ ПЛОХО - большой интерфейс
class Worker {
  work() {}
  eat() {}
}

class Robot implements Worker {
  work() { /* ... */ }
  eat() { // Robot не должен есть!
    throw new Error('Robot cannot eat');
  }
}

// ✅ ХОРОШО - маленькие, специфичные интерфейсы
class Workable {
  work() {}
}

class Eatable {
  eat() {}
}

class Human {
  work() { /* ... */ }
  eat() { /* ... */ }
}

class Robot {
  work() { /* ... */ }
  // Robot не реализует Eatable
}
```

### 5. **D - Dependency Inversion Principle (DIP)**

Зависимости должны идти вверх по иерархии абстракций.

```javascript
// ❌ ПЛОХО - зависит от конкретных деталей
class UserService {
  constructor() {
    this.database = new MySQLDatabase(); // конкретный класс
  }
  
  getUser(id) {
    return this.database.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// ✅ ХОРОШО - зависит от абстракции
class Database {
  query(sql) {
    throw new Error('Not implemented');
  }
}

class MySQLDatabase extends Database {
  query(sql) {
    // MySQL-специфичная реализация
  }
}

class UserService {
  constructor(database) {
    this.database = database; // абстракция
  }
  
  getUser(id) {
    return this.database.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// Можно легко переключиться на другую БД
const service = new UserService(new PostgresDatabase());
```

---

## Часть 5: Gang of Four (GoF) паттерны

### Паттерны Создания (Creational)

#### 1. **Singleton**

Гарантирует, что класс имеет только один экземпляр.

```javascript
// ❌ ПЛОХО в JavaScript
class Database {
  static instance = null;
  
  static getInstance() {
    if (!Database.instance) {
      Database.instance = new Database();
    }
    return Database.instance;
  }
  
  constructor() {
    if (Database.instance) {
      return Database.instance;
    }
  }
}

// ✅ ХОРОШО в JavaScript - просто используй модуль
const db = {
  connect() { /* ... */ },
  query(sql) { /* ... */ }
};

export default db;

// И в коде просто импортируй
import db from './database';
db.query('SELECT * FROM users');
```

Почему Singleton плохо в JavaScript? Потому что модули уже являются singleton-ами!

#### 2. **Factory Method**

Создает объекты без указания их конкретных классов.

```javascript
// ✅ ХОРОШО
class PaymentFactory {
  static create(type, config) {
    switch(type) {
      case 'credit-card':
        return new CreditCardPayment(config);
      case 'paypal':
        return new PayPalPayment(config);
      case 'bitcoin':
        return new BitcoinPayment(config);
      default:
        throw new Error(`Unknown payment type: ${type}`);
    }
  }
}

class CreditCardPayment {
  constructor(config) {
    this.cardNumber = config.cardNumber;
  }
}

const payment = PaymentFactory.create('credit-card', { cardNumber: '1234-5678' });
```

#### 3. **Abstract Factory**

Создает семейства связанных объектов.

```javascript
class UIFactory {
  createButton() {
    throw new Error('Not implemented');
  }
  
  createCheckbox() {
    throw new Error('Not implemented');
  }
}

class WindowsUIFactory extends UIFactory {
  createButton() {
    return new WindowsButton();
  }
  
  createCheckbox() {
    return new WindowsCheckbox();
  }
}

class MacUIFactory extends UIFactory {
  createButton() {
    return new MacButton();
  }
  
  createCheckbox() {
    return new MacCheckbox();
  }
}

class Application {
  constructor(factory) {
    this.button = factory.createButton();
    this.checkbox = factory.createCheckbox();
  }
  
  render() {
    this.button.draw();
    this.checkbox.draw();
  }
}

const app = new Application(
  process.platform === 'win32' ? new WindowsUIFactory() : new MacUIFactory()
);
app.render();
```

#### 4. **Builder**

Разделяет конструирование сложного объекта от его представления.

```javascript
// ✅ ХОРОШО
class SQLQueryBuilder {
  constructor() {
    this.query = {};
  }
  
  select(...columns) {
    this.query.select = columns;
    return this;
  }
  
  from(table) {
    this.query.from = table;
    return this;
  }
  
  where(condition) {
    this.query.where = condition;
    return this;
  }
  
  limit(n) {
    this.query.limit = n;
    return this;
  }
  
  build() {
    const { select, from, where, limit } = this.query;
    let sql = `SELECT ${select.join(', ')} FROM ${from}`;
    if (where) sql += ` WHERE ${where}`;
    if (limit) sql += ` LIMIT ${limit}`;
    return sql;
  }
}

const query = new SQLQueryBuilder()
  .select('id', 'name', 'email')
  .from('users')
  .where('age > 18')
  .limit(10)
  .build();

console.log(query);
// SELECT id, name, email FROM users WHERE age > 18 LIMIT 10
```

### Паттерны Структуры (Structural)

#### 5. **Adapter (Adapter Pattern)**

Преобразует интерфейс класса в другой интерфейс, который ожидается клиентом.

```javascript
// Старая библиотека
class OldAuthService {
  loginUser(username, password) {
    return { token: 'old-token-' + username };
  }
}

// Новая библиотека ожидает другой интерфейс
class NewAuthService {
  authenticate(credentials) {
    return { accessToken: 'new-' + credentials.user };
  }
}

// Adapter
class AuthAdapter {
  constructor(oldService) {
    this.oldService = oldService;
  }
  
  authenticate(credentials) {
    const { token } = this.oldService.loginUser(
      credentials.user,
      credentials.pass
    );
    return { accessToken: token };
  }
}

const newService = new AuthAdapter(new OldAuthService());
console.log(newService.authenticate({ user: 'john', pass: '123' }));
```

#### 6. **Decorator**

Динамически добавляет новую функциональность к объекту.

```javascript
// Базовый класс
class Coffee {
  cost() {
    return 5;
  }
  
  description() {
    return 'Coffee';
  }
}

// Декораторы
class MilkDecorator {
  constructor(coffee) {
    this.coffee = coffee;
  }
  
  cost() {
    return this.coffee.cost() + 2;
  }
  
  description() {
    return this.coffee.description() + ', Milk';
  }
}

class SugarDecorator {
  constructor(coffee) {
    this.coffee = coffee;
  }
  
  cost() {
    return this.coffee.cost() + 0.5;
  }
  
  description() {
    return this.coffee.description() + ', Sugar';
  }
}

let coffee = new Coffee();
console.log(`${coffee.description()} - $${coffee.cost()}`); // Coffee - $5

coffee = new MilkDecorator(coffee);
console.log(`${coffee.description()} - $${coffee.cost()}`); // Coffee, Milk - $7

coffee = new SugarDecorator(coffee);
console.log(`${coffee.description()} - $${coffee.cost()}`); // Coffee, Milk, Sugar - $7.5
```

#### 7. **Facade**

Предоставляет единый упрощенный интерфейс к сложной системе.

```javascript
// Сложная система
class CPU {
  execute(instruction) {
    console.log(`Executing: ${instruction}`);
  }
}

class Memory {
  load(address) {
    return `Data from ${address}`;
  }
}

class HardDrive {
  read(address) {
    return `File from ${address}`;
  }
}

// Facade
class Computer {
  constructor() {
    this.cpu = new CPU();
    this.memory = new Memory();
    this.hardDrive = new HardDrive();
  }
  
  start() {
    this.cpu.execute('START');
    this.memory.load('0x0000');
    this.hardDrive.read('boot.bin');
  }
  
  shutdown() {
    this.cpu.execute('SHUTDOWN');
  }
}

// Клиент просто использует Computer, не беспокоясь о деталях
const computer = new Computer();
computer.start();
computer.shutdown();
```

#### 8. **Proxy**

Предоставляет заместителя для контроля доступа к другому объекту.

```javascript
class DatabaseConnection {
  query(sql) {
    // Дорогостоящая операция
    console.log(`Executing: ${sql}`);
    return { result: 'data' };
  }
}

class DatabaseProxy {
  constructor() {
    this.connection = null;
    this.cache = new Map();
  }
  
  query(sql) {
    // Кэширование
    if (this.cache.has(sql)) {
      console.log('Cache hit!');
      return this.cache.get(sql);
    }
    
    // Ленивая инициализация
    if (!this.connection) {
      this.connection = new DatabaseConnection();
    }
    
    const result = this.connection.query(sql);
    this.cache.set(sql, result);
    return result;
  }
}

const db = new DatabaseProxy();
db.query('SELECT * FROM users'); // Выполняет запрос
db.query('SELECT * FROM users'); // Из кэша
```

### Паттерны Поведения (Behavioral)

#### 9. **Observer (Наблюдатель)**

Определяет отношение "один ко многим" между объектами так, чтобы при изменении состояния одного объекта автоматически изменялись и обновлялись все зависящие от него объекты.

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
  }
  
  on(event, handler) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(handler);
  }
  
  emit(event, data) {
    if (this.events[event]) {
      this.events[event].forEach(handler => handler(data));
    }
  }
}

class UserService extends EventEmitter {
  createUser(userData) {
    const user = { id: 1, ...userData };
    this.emit('user:created', user);
    return user;
  }
}

const userService = new UserService();

userService.on('user:created', (user) => {
  console.log(`Welcome email sent to ${user.email}`);
});

userService.on('user:created', (user) => {
  console.log(`User ${user.name} added to mailing list`);
});

userService.createUser({ name: 'John', email: 'john@example.com' });
// Welcome email sent to john@example.com
// User John added to mailing list
```

#### 10. **Strategy (Стратегия)**

Определяет семейство алгоритмов, инкапсулирует каждый из них и делает их взаимозаменяемыми.

```javascript
class SortingStrategy {
  sort(array) {
    throw new Error('Not implemented');
  }
}

class BubbleSort extends SortingStrategy {
  sort(array) {
    const arr = [...array];
    for (let i = 0; i < arr.length; i++) {
      for (let j = 0; j < arr.length - 1; j++) {
        if (arr[j] > arr[j + 1]) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }
    return arr;
  }
}

class QuickSort extends SortingStrategy {
  sort(array) {
    if (array.length <= 1) return array;
    const pivot = array[0];
    const less = array.slice(1).filter(x => x <= pivot);
    const greater = array.slice(1).filter(x => x > pivot);
    return [...this.sort(less), pivot, ...this.sort(greater)];
  }
}

class Sorter {
  constructor(strategy) {
    this.strategy = strategy;
  }
  
  sort(array) {
    return this.strategy.sort(array);
  }
}

const data = [5, 2, 8, 1, 9];

const bubbleSorter = new Sorter(new BubbleSort());
console.log(bubbleSorter.sort(data)); // [1, 2, 5, 8, 9]

const quickSorter = new Sorter(new QuickSort());
console.log(quickSorter.sort(data)); // [1, 2, 5, 8, 9]
```

#### 11. **State (Состояние)**

Позволяет объекту изменять поведение в зависимости от его внутреннего состояния.

```javascript
class TrafficLightState {
  enter() {}
  exit() {}
}

class RedLightState extends TrafficLightState {
  enter() {
    console.log('🔴 Red light - STOP');
  }
  
  next() {
    return new GreenLightState();
  }
}

class GreenLightState extends TrafficLightState {
  enter() {
    console.log('🟢 Green light - GO');
  }
  
  next() {
    return new YellowLightState();
  }
}

class YellowLightState extends TrafficLightState {
  enter() {
    console.log('🟡 Yellow light - PREPARE TO STOP');
  }
  
  next() {
    return new RedLightState();
  }
}

class TrafficLight {
  constructor() {
    this.state = new RedLightState();
    this.state.enter();
  }
  
  change() {
    this.state = this.state.next();
    this.state.enter();
  }
}

const light = new TrafficLight();
light.change(); // 🟢 Green light - GO
light.change(); // 🟡 Yellow light - PREPARE TO STOP
light.change(); // 🔴 Red light - STOP
```

#### 12. **Command (Команда)**

Инкапсулирует запрос как объект, позволяя параметризовать клиентов с различными запросами.

```javascript
class Command {
  execute() {
    throw new Error('Not implemented');
  }
  
  undo() {
    throw new Error('Not implemented');
  }
}

class AddNumberCommand extends Command {
  constructor(receiver, number) {
    super();
    this.receiver = receiver;
    this.number = number;
  }
  
  execute() {
    this.receiver.add(this.number);
  }
  
  undo() {
    this.receiver.subtract(this.number);
  }
}

class SubtractNumberCommand extends Command {
  constructor(receiver, number) {
    super();
    this.receiver = receiver;
    this.number = number;
  }
  
  execute() {
    this.receiver.subtract(this.number);
  }
  
  undo() {
    this.receiver.add(this.number);
  }
}

class Calculator {
  constructor() {
    this.value = 0;
  }
  
  add(number) {
    this.value += number;
    console.log(`Value: ${this.value}`);
  }
  
  subtract(number) {
    this.value -= number;
    console.log(`Value: ${this.value}`);
  }
}

class CommandInvoker {
  constructor() {
    this.history = [];
  }
  
  execute(command) {
    command.execute();
    this.history.push(command);
  }
  
  undo() {
    const command = this.history.pop();
    if (command) {
      command.undo();
    }
  }
}

const calculator = new Calculator();
const invoker = new CommandInvoker();

invoker.execute(new AddNumberCommand(calculator, 5)); // Value: 5
invoker.execute(new AddNumberCommand(calculator, 3)); // Value: 8
invoker.execute(new SubtractNumberCommand(calculator, 2)); // Value: 6
invoker.undo(); // Value: 8
invoker.undo(); // Value: 5
```

#### 13. **Iterator (Итератор)**

Предоставляет способ последовательного доступа к элементам совокупного объекта без раскрытия его основного представления.

```javascript
class MyCollection {
  constructor(items) {
    this.items = items;
  }
  
  [Symbol.iterator]() {
    let index = 0;
    const items = this.items;
    
    return {
      next: () => {
        if (index < items.length) {
          return { value: items[index++], done: false };
        }
        return { done: true };
      }
    };
  }
}

const collection = new MyCollection([1, 2, 3, 4, 5]);

for (const item of collection) {
  console.log(item);
}
// 1
// 2
// 3
// 4
// 5
```

#### 14. **Template Method (Шаблонный метод)**

Определяет скелет алгоритма в базовом классе, но оставляет определение отдельных этапов для подклассов.

```javascript
class DataProcessor {
  process(data) {
    const parsed = this.parse(data);
    const validated = this.validate(parsed);
    const result = this.transform(validated);
    return this.save(result);
  }
  
  parse(data) {
    throw new Error('Not implemented');
  }
  
  validate(data) {
    throw new Error('Not implemented');
  }
  
  transform(data) {
    throw new Error('Not implemented');
  }
  
  save(data) {
    throw new Error('Not implemented');
  }
}

class JSONProcessor extends DataProcessor {
  parse(data) {
    console.log('Parsing JSON...');
    return JSON.parse(data);
  }
  
  validate(data) {
    console.log('Validating JSON...');
    return data;
  }
  
  transform(data) {
    console.log('Transforming JSON...');
    return { ...data, processed: true };
  }
  
  save(data) {
    console.log('Saving to database...');
    return { success: true };
  }
}

const processor = new JSONProcessor();
processor.process('{"name": "John", "age": 30}');
```

---

## Часть 6: Flyweight паттерн в деталях

### Что это такое?

**Flyweight** - это паттерн оптимизации памяти. Его основная идея: если у вас есть много объектов, которые отличаются только несколькими параметрами, можно переиспользовать "легкие" объекты и хранить только различающиеся данные.

### Классический пример: таймеры

```javascript
// ❌ ПЛОХО - создаем по таймеру на каждый интервал
class Timer {
  constructor(interval) {
    this.interval = interval;
    this.callback = null;
    this.startTime = Date.now();
    this.isRunning = false;
    // ... еще много свойств
  }
}

// У нас есть 10000 интервалов с интервалом 1000мс
// Создаем 10000 разных Timer объектов
for (let i = 0; i < 10000; i++) {
  new Timer(1000); // Все идентичны!
}

// ✅ ХОРОШО - переиспользуем Timer для одинаковых интервалов
class Timer {
  constructor(interval) {
    this.interval = interval;
    this.callbacks = [];
  }
  
  addCallback(cb) {
    this.callbacks.push(cb);
  }
}

class TimerFactory {
  static timers = new Map();
  
  static getTimer(interval) {
    if (!this.timers.has(interval)) {
      this.timers.set(interval, new Timer(interval));
    }
    return this.timers.get(interval);
  }
}

class Interval {
  constructor(interval, callback) {
    this.interval = interval;
    this.callback = callback;
    const timer = TimerFactory.getTimer(interval);
    timer.addCallback(callback);
  }
}

// Теперь вместо 10000 Timer объектов у нас только несколько
// (по одному на каждый уникальный интервал)
```

### Когда использовать?

1. **Много объектов** - нужно иметь хотя бы тысячи экземпляров
2. **Похожесть** - объекты отличаются только в few параметрах
3. **Затраты на память** - создание объектов дорого по памяти
4. **Иммутабельность** - "легкие" объекты не меняются после создания

### Примеры из реальной жизни

**В видеоиграх**: каждый пиксель на экране или каждый спрайт врага может быть flyweight объектом с переиспользуемыми данными о текстурах и моделях.

**В браузере**: DOM ноды, особенно при большом количестве элементов в списке.

**В сервере**: объекты для обработки запросов в Node.js приложении.

---

## Ключевые выводы

### 1. **Понимайте язык программирования**
- JavaScript != Java != C++
- V8 оптимизирует мономорфные структуры данных
- Не копируйте паттерны слепо из других языков

### 2. **Баланс между читаемостью и производительностью**
- Чистый код важен, но не за счет производительности на серверной части
- На клиенте производительность менее критична (обычно)
- На сервере тысячи пользователей требуют оптимизации

### 3. **GRASP раньше, чем SOLID и GoF**
- Начните с правильного распределения ответственности (GRASP)
- Потом применяйте SOLID для масштабируемости
- GoF паттерны - это инструменты, а не закон

### 4. **Профилируйте перед оптимизацией**
- Не гадайте, где узкие места
- Используйте инструменты профилирования
- Оптимизируйте только то, что действительно медленно

### 5. **Критическое мышление**
- Анализируйте рекомендации, не принимайте их на веру
- Проверяйте, работает ли паттерн в вашем конкретном языке
- Будьте готовы отступиться от рекомендаций, если есть веские причины

### 6. **Модули - лучший паттерн для JavaScript**
- Вместо Singleton используйте модули (они уже singleton-ы)
- Вместо Factory часто можно использовать функции
- Вместо классов иногда лучше использовать объекты и функции

---

## Рекомендуемые источники

1. **Gang of Four** - "Design Patterns" (классика, но для Java/C++)
2. **Robert C. Martin** - "Clean Code" (хорошо, но нужно критически читать)
3. **Craig Larman** - "Applying UML and Patterns" (GRASP паттерны)
4. **V8 документация** - особенности оптимизации объектов
5. **JavaScript мудрость** - Kyle Simpson "You Don't Know JS"

---

## Практическое резюме

**Что нужно помнить на работе:**

1. ✅ Применяйте паттерны, когда они решают реальную проблему
2. ✅ Думайте о производительности, особенно на backend
3. ✅ Профилируйте и измеряйте
4. ✅ Читайте исходный код языка (V8 для JavaScript)
5. ✅ Балансируйте между читаемостью и производительностью
6. ❌ Не копируйте паттерны из других языков автоматически
7. ❌ Не применяйте паттерны "на всякий случай"
8. ❌ Не игнорируйте производительность ради паттернов

