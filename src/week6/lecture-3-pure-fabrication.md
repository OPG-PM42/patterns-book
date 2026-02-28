# Лекция 3. GRASP — Pure Fabrication (Чистая выдумка)

## Введение

**Pure Fabrication** (Чистая выдумка) — один из девяти принципов GRASP. Он отвечает
на вопрос: что делать, если нам нужна программная абстракция, которая **не соответствует
ни одной сущности предметной области**, но при этом значительно снижает зацепление
(Low Coupling) и улучшает связность (High Cohesion) кода?

Ответ: **придумываем её**. Отсюда и название — "чистая выдумка".

Примеры чистых выдумок, которые мы уже изучали:
- `Promise` — нет такой вещи в реальном мире
- `EventEmitter` — тоже придуман для удобства программирования
- `Socket` — абстракция над сетевым соединением
- Слой доступа к данным (Repository, Loader, Storage) — не объект предметной области

---

## 3.1 Проблема: знание о слое данных внутри доменного класса

Рассмотрим типичную ошибку — когда класс предметной области "знает" о структуре
базы данных.

```javascript
// Пример 1. ПЛОХОЙ КОД — Person знает о базе данных (высокое зацепление)

// Симуляция объекта соединения с базой данных
const db = {
  async query(sql, params) {
    // В реальности здесь был бы вызов pg, mysql2 и т.д.
    console.log(`Выполняем SQL: ${sql}`, params);
    // Возвращаем симулированные данные
    return { name: 'Рене Декарт', age: 54, address: 'Нидерланды' };
  },
};

// ПРОБЛЕМА: класс Person сам умеет загружать себя из БД
class Person {
  constructor(data) {
    // Person знает о структуре записи из базы данных
    this.name = data.name;
    this.age = data.age;
    this.address = data.address;
  }

  // ПЛОХО: статический метод загрузки прямо в доменном классе
  // Person не должен знать о SQL и структуре таблиц
  static async load(db, id) {
    const data = await db.query('SELECT name, age, address FROM persons WHERE id = $1', [id]);
    // Person знает, что поля называются именно name, age, address
    return new Person(data);
  }
}

// Использование
async function main() {
  const person = await Person.load(db, 101);
  console.log(person);
}

main();
// Проблемы:
// 1. Person привязан к структуре БД (изменение схемы = изменение Person)
// 2. Person нарушает Single Responsibility Principle
// 3. Сложно тестировать — нужна реальная БД или мок
```

---

## 3.2 Шаг 1: Разделение Person и логики загрузки

Первое улучшение — вынести метод загрузки из класса `Person` в отдельный класс.

```javascript
// Пример 2. Разделение: Person — только предметная область, Loader — чистая выдумка

// Симуляция базы данных
const db = {
  async query(sql, params) {
    console.log(`SQL: ${sql}`, params);
    return { name: 'Рене Декарт', age: 54, address: 'Нидерланды' };
  },
};

// Чистый доменный класс — знает только о своих полях предметной области
class Person {
  constructor({ name, age, address }) {
    this.name = name;
    this.age = age;
    this.address = address;
  }

  // Методы предметной области (если нужны)
  greet() {
    return `Меня зовут ${this.name}`;
  }
}

// ЧИСТАЯ ВЫДУМКА: Loader — класс, которого нет в предметной области,
// но он нужен нам для организации кода
// Loader знает о базе данных, Person — нет
class PersonLoader {
  constructor(db) {
    this.db = db;    // ссылка на соединение с БД (Dependency Injection)
  }

  // Загружает Person из БД по идентификатору
  async load(id) {
    const data = await this.db.query(
      'SELECT name, age, address FROM persons WHERE id = $1',
      [id],
    );
    // Loader создаёт Person из данных БД
    return new Person(data);
  }
}

// Использование
async function main() {
  const loader = new PersonLoader(db);
  const person = await loader.load(101);
  console.log(person.greet());
  // Вывод: SQL: SELECT name, age, address FROM persons WHERE id = $1 [ 101 ]
  //        Меня зовут Рене Декарт
}

main();
```

---

## 3.3 Шаг 2: Универсальный Loader через замыкания

Ещё лучше — сделать Loader универсальным, не привязанным к конкретному классу Person.
Здесь мы используем метапрограммирование JavaScript: создаём класс "на лету".

```javascript
// Пример 3. Универсальный Loader — ещё одна чистая выдумка
// Loader не знает о Person, он работает с любой сущностью

const db = {
  async query(sql, params) {
    console.log(`SQL: ${sql}`, params);
    // Симулируем данные из разных таблиц
    const mockData = {
      persons: { name: 'Рене Декарт', age: 54, address: 'Нидерланды' },
      products: { title: 'Книга по философии', price: 999 },
    };
    return mockData[params[1]] ?? {};
  },
};

// Фабричная функция: принимает соединение с БД и имя сущности,
// возвращает асинхронную функцию загрузки
// Это замыкание — вместо класса для краткости
const createLoader = (db, entityName) => {
  // Создаём анонимный класс и даём ему имя через Object.defineProperty
  // Это метапрограммирование: имя класса формируется динамически
  const EntityClass = class {};
  Object.defineProperty(EntityClass, 'name', {
    value: entityName,
    writable: false,
  });

  // SQL-запрос тоже формируется по имени сущности
  const sql = `SELECT * FROM ${entityName} WHERE id = $1`;

  // Возвращаем функцию загрузки — замыкание над db, EntityClass и sql
  return async (id) => {
    const data = await db.query(sql, [id, entityName]);
    // Создаём экземпляр нужного класса и копируем в него поля из БД
    return Object.assign(new EntityClass(), data);
  };
};

// Использование: создаём загрузчики для разных сущностей
async function main() {
  const loadPerson = createLoader(db, 'persons');
  const loadProduct = createLoader(db, 'products');

  const person = await loadPerson(101);
  console.log('Загружен:', person.constructor.name, person);
  // Вывод: Загружен: persons { name: 'Рене Декарт', age: 54, address: 'Нидерланды' }

  const product = await loadProduct(42);
  console.log('Загружен:', product.constructor.name, product);
  // Вывод: Загружен: products { title: 'Книга по философии', price: 999 }
}

main();
```

---

## 3.4 Чистая выдумка для работы с файлами: класс Storage

Ещё один классический пример Pure Fabrication — слой хранения данных в файловой системе.
Сначала покажем плохой вариант, затем правильный.

```javascript
// Пример 4. ПЛОХОЙ КОД — Person сам умеет сохраняться на диск

import { writeFileSync, readFileSync, existsSync } from 'node:fs';

// ПРОБЛЕМА: Person знает о файловой системе, сериализации, путях к файлам
class Person {
  constructor(name, age, fileName) {
    this.name = name;
    this.age = age;
    // fileName — это деталь реализации, которая не должна быть в Person
    // В предметной области у Person нет "имени файла"
    Object.defineProperty(this, 'fileName', {
      value: fileName,
      enumerable: false,   // не попадёт в JSON.stringify
      writable: false,
    });
  }

  // ПЛОХО: Person знает о сериализации и записи в файл
  save() {
    const json = JSON.stringify(this);
    writeFileSync(this.fileName, json, 'utf-8');
    console.log(`Сохранено в ${this.fileName}`);
  }

  // ПЛОХО: Person умеет читать себя из файла
  static load(fileName) {
    const json = readFileSync(fileName, 'utf-8');
    const data = JSON.parse(json);
    return Object.assign(new Person('', 0, fileName), data);
  }
}

// Использование
const person = new Person('Рене Декарт', 54, './descartes.json');
person.save();

const loaded = Person.load('./descartes.json');
console.log(loaded);
// Проблемы те же: Person нарушает SRP, тяжело тестировать, тяжело переиспользовать
```

```javascript
// Пример 5. ХОРОШИЙ КОД — Storage как чистая выдумка (Pure Fabrication)

import { writeFileSync, readFileSync, mkdirSync, existsSync } from 'node:fs';
import { join } from 'node:path';

// Чистый доменный класс — только поля предметной области
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }

  introduce() {
    return `Я ${this.name}, мне ${this.age} лет`;
  }
}

// ЧИСТАЯ ВЫДУМКА: Storage
// Этот класс не существует в предметной области, но он нам нужен
// для организации работы с файловой системой
// Преимущества:
// - Person ничего не знает о файлах
// - Storage можно заменить (например, на DatabaseStorage) без изменения Person
// - Storage легко тестировать изолированно
class Storage {
  constructor(directory) {
    this.directory = directory;
    // Создаём директорию, если её нет
    if (!existsSync(directory)) {
      mkdirSync(directory, { recursive: true });
    }
  }

  // Формируем путь к файлу по идентификатору
  _getFilePath(id) {
    return join(this.directory, `${id}.json`);
  }

  // Сохраняем произвольный объект под указанным идентификатором
  save(id, entity) {
    const filePath = this._getFilePath(id);
    const json = JSON.stringify(entity, null, 2);
    writeFileSync(filePath, json, 'utf-8');
    console.log(`Сохранено: ${filePath}`);
  }

  // Загружаем объект по идентификатору и "превращаем" его в экземпляр нужного класса
  load(id, EntityClass) {
    const filePath = this._getFilePath(id);
    if (!existsSync(filePath)) {
      throw new Error(`Запись с id=${id} не найдена`);
    }
    const json = readFileSync(filePath, 'utf-8');
    const data = JSON.parse(json);
    // Object.assign копирует все поля из data в новый экземпляр EntityClass
    return Object.assign(new EntityClass('', 0), data);
  }
}

// Использование — Storage и Person слабо связаны (Low Coupling)
async function main() {
  const storage = new Storage('./data');

  // Создаём Person без каких-либо знаний о хранении
  const person = new Person('Рене Декарт', 54);

  // Storage сохраняет его — Person об этом не знает
  storage.save(100, person);

  // Storage загружает и восстанавливает объект
  const loaded = storage.load(100, Person);
  console.log(loaded.introduce());
  // Вывод: Я Рене Декарт, мне 54 лет
}

main();
```

---

## 3.5 Несколько уровней чистой выдумки

В реальных приложениях чистые выдумки образуют **слои**. Каждый слой скрывает
детали от следующего.

```javascript
// Пример 6. Три слоя чистой выдумки: Domain -> Repository -> Driver

// ==========================================
// Слой 1: Предметная область (Domain)
// Только бизнес-логика, никаких деталей хранения
// ==========================================

class Product {
  constructor(title, price, category) {
    this.title = title;
    this.price = price;
    this.category = category;
  }

  isExpensive() {
    return this.price > 1000;
  }

  applyDiscount(percent) {
    return new Product(
      this.title,
      this.price * (1 - percent / 100),
      this.category,
    );
  }
}

// ==========================================
// Слой 2: Репозиторий (ЧИСТАЯ ВЫДУМКА #1)
// Абстракция над хранилищем — не знает, это файл, БД или память
// Знает только о Product
// ==========================================

class ProductRepository {
  constructor(driver) {
    // driver — ещё одна чистая выдумка ниже уровнем
    this.driver = driver;
  }

  async findById(id) {
    const data = await this.driver.read('products', id);
    if (!data) return null;
    return Object.assign(new Product('', 0, ''), data);
  }

  async save(id, product) {
    await this.driver.write('products', id, { ...product });
  }

  async findExpensive() {
    const all = await this.driver.readAll('products');
    return all
      .map((data) => Object.assign(new Product('', 0, ''), data))
      .filter((p) => p.isExpensive());
  }
}

// ==========================================
// Слой 3: Драйвер хранения (ЧИСТАЯ ВЫДУМКА #2)
// Знает о конкретном способе хранения (в памяти, файл, БД)
// Не знает о Product — работает с сырыми данными
// ==========================================

class InMemoryDriver {
  constructor() {
    // Простое хранилище в памяти: { 'products': { 1: {...}, 2: {...} } }
    this.store = new Map();
  }

  async read(collection, id) {
    const coll = this.store.get(collection);
    return coll ? coll.get(id) ?? null : null;
  }

  async write(collection, id, data) {
    if (!this.store.has(collection)) {
      this.store.set(collection, new Map());
    }
    this.store.get(collection).set(id, data);
  }

  async readAll(collection) {
    const coll = this.store.get(collection);
    return coll ? [...coll.values()] : [];
  }
}

// ==========================================
// Пример использования
// ==========================================

async function main() {
  // Собираем слои через Dependency Injection
  const driver = new InMemoryDriver();           // чистая выдумка #2
  const repo = new ProductRepository(driver);    // чистая выдумка #1

  // Работаем с предметной областью — про слои хранения не думаем
  const book = new Product('Паттерны проектирования', 1500, 'books');
  const pen = new Product('Ручка', 50, 'stationery');

  await repo.save(1, book);
  await repo.save(2, pen);

  // Загружаем по id
  const loaded = await repo.findById(1);
  console.log(`Загружен: ${loaded.title}, цена: ${loaded.price}`);
  // Вывод: Загружен: Паттерны проектирования, цена: 1500

  // Используем метод предметной области
  console.log(`Дорогой товар? ${loaded.isExpensive()}`);
  // Вывод: Дорогой товар? true

  // Поиск дорогих товаров
  const expensive = await repo.findExpensive();
  console.log('Дорогие товары:', expensive.map((p) => p.title));
  // Вывод: Дорогие товары: [ 'Паттерны проектирования' ]
}

main();
```

---

## 3.6 Стратегия как расширение чистой выдумки

Чистая выдумка (Loader, Storage, Repository) может быть расширена до **паттерна Стратегия**:
базовый класс определяет интерфейс, конкретные подклассы — реализации.

```javascript
// Пример 7. Repository как стратегия: файловое и in-memory хранилище

import { writeFileSync, readFileSync, readdirSync, existsSync, mkdirSync } from 'node:fs';
import { join } from 'node:path';

// ==========================================
// Базовый класс — определяет интерфейс (контракт)
// ==========================================

class BaseRepository {
  // Абстрактный метод — подклассы обязаны переопределить
  async save(id, entity) {
    throw new Error('save() не реализован');
  }

  async findById(id) {
    throw new Error('findById() не реализован');
  }

  async findAll() {
    throw new Error('findAll() не реализован');
  }
}

// ==========================================
// Стратегия 1: хранение в памяти (для тестов)
// ==========================================

class MemoryRepository extends BaseRepository {
  constructor() {
    super();
    this.data = new Map();
  }

  async save(id, entity) {
    // Сохраняем копию, чтобы избежать мутации оригинала
    this.data.set(id, { ...entity });
  }

  async findById(id) {
    return this.data.get(id) ?? null;
  }

  async findAll() {
    return [...this.data.values()];
  }
}

// ==========================================
// Стратегия 2: хранение в файлах (для продакшена)
// ==========================================

class FileRepository extends BaseRepository {
  constructor(directory) {
    super();
    this.directory = directory;
    if (!existsSync(directory)) {
      mkdirSync(directory, { recursive: true });
    }
  }

  _filePath(id) {
    return join(this.directory, `${id}.json`);
  }

  async save(id, entity) {
    writeFileSync(this._filePath(id), JSON.stringify(entity, null, 2), 'utf-8');
  }

  async findById(id) {
    const path = this._filePath(id);
    if (!existsSync(path)) return null;
    return JSON.parse(readFileSync(path, 'utf-8'));
  }

  async findAll() {
    if (!existsSync(this.directory)) return [];
    return readdirSync(this.directory)
      .filter((f) => f.endsWith('.json'))
      .map((f) => JSON.parse(readFileSync(join(this.directory, f), 'utf-8')));
  }
}

// ==========================================
// Предметная область — не зависит от реализации хранилища
// ==========================================

class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
}

// ==========================================
// Функция сервисного слоя — работает с любой стратегией хранилища
// ==========================================

async function runDemo(repository) {
  await repository.save(1, new Person('Рене Декарт', 54));
  await repository.save(2, new Person('Иммануил Кант', 79));

  const descartes = await repository.findById(1);
  console.log('Найден:', descartes);

  const all = await repository.findAll();
  console.log('Все записи:', all.length, 'штук');
}

// Запускаем с разными стратегиями хранения
async function main() {
  console.log('--- In-Memory хранилище ---');
  await runDemo(new MemoryRepository());

  console.log('\n--- Файловое хранилище ---');
  await runDemo(new FileRepository('./tmp-persons'));
}

main();
// Код сервисного слоя (runDemo) одинаков для обеих стратегий!
// Pure Fabrication + Strategy = мощная комбинация
```

---

## 3.7 Итог: почему Pure Fabrication важна

```javascript
// Пример 8. Краткая демонстрация эффекта от применения Pure Fabrication

// БЕЗ Pure Fabrication:
// User --знает о--> Database
// User --знает о--> FileSystem
// User --знает о--> NetworkAPI
// Высокое зацепление (High Coupling) — изменение БД ломает User

// С Pure Fabrication:
// User --использует--> UserRepository (чистая выдумка)
// UserRepository --использует--> Driver (чистая выдумка)
// Driver --работает с--> Database / FileSystem / NetworkAPI
// Каждый слой знает только о следующем уровне абстракции

// Ключевые выгоды Pure Fabrication:
// 1. Low Coupling — User не зависит от конкретной БД
// 2. High Cohesion — каждый класс делает одно дело
// 3. Testability — можно подменить Driver на MemoryDriver в тестах
// 4. Reusability — Repository можно использовать с разными Entity

// Примеры Pure Fabrication в экосистеме Node.js:
const examples = [
  'EventEmitter    — нет такого в предметной области, но упрощает события',
  'Promise         — абстракция над асинхронностью, не объект реального мира',
  'Readable/Writable Stream — абстракция над потоком данных',
  'Repository      — нет в предметной области, но нужен для Low Coupling',
  'Logger          — не бизнес-логика, но нужен везде',
  'ConnectionPool  — техническая абстракция над пулом соединений',
  'Middleware      — концепция из Express, нет в HTTP-спецификации',
];

examples.forEach((ex) => console.log('-', ex));
```

---

## Заключение

Три темы, разобранные в лекциях этой недели, тесно связаны между собой:

1. **EventTarget / EventEmitter** — конкретные реализации контракта событий в JavaScript.
   Понимание разницы между ними критично для написания кода, совместимого с браузером
   и Node.js одновременно.

2. **Observer / Observable** — паттерн, который лежит в основе EventEmitter, RxJS
   и других реактивных библиотек. Реализация "с нуля" позволяет понять, что происходит
   внутри готовых инструментов.

3. **Pure Fabrication** — принцип GRASP, объясняющий, почему мы создаём абстракции
   вроде Repository, Loader, Storage. Это не усложнение ради усложнения, а сознательное
   снижение зацепления между слоями приложения.

**Практический совет:** Реализуйте каждый из примеров самостоятельно, добавьте
`console.log` в ключевые места, напишите тесты. Только через практику приходит
настоящее понимание этих концепций.
