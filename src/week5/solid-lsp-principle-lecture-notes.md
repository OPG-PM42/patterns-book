# Принцип подстановки Барбары Лисков (Liskov Substitution Principle — LSP)

## Обзор

Принцип подстановки Барбары Лисков — это третий принцип SOLID, который связан с теорией типов и описывает правила корректного наследования и использования подтипов. Этот принцип обеспечивает возможность заменять объекты базового типа на объекты производного типа без изменения корректности программы.

---

## Формулировка принципа

### Оригинальная формулировка Барбары Лисков

Если у нас есть **Q(x)** — свойство, которое истинно относительно объекта **x**, который является экземпляром типа **T**, то **Q(y)** также должно быть истинным для объекта **y**, который является экземпляром типа **S**, где **S** является подтипом **T**.

Математическая запись:
```
Q(x), где x: T
Q(y), где y: S, и S <: T (S является подтипом T)
```

**Примечание**: Эта формулировка достаточно абстрактная и требует нескольких прочтений для полного понимания.

### Формулировка Роберта Мартина

Более понятная формулировка от Роберта Мартина гласит:

> **Функции, которые используют базовый тип, должны иметь возможность использовать подтипы этого базового типа, не зная об этом.**

Это означает, что если функция принимает аргумент типа T, то мы можем передать любой подтип этого типа T, и функция должна корректно работать.

---

## Области применения LSP

Принцип подстановки применяется не только к аргументам функций, но и к:

### 1. Аргументы функций
Если функция принимает параметр типа T, она должна корректно работать с любым подтипом T.

### 2. Свойства объектов
Если свойство класса имеет тип T, то в это свойство можно присвоить любое значение, являющееся подтипом T.

**Пример:**
```javascript
// Если у нас есть свойство типа Stream
let stream;

// Мы можем присвоить ему WritableStream (наследник Stream)
stream = new WritableStream();

// Или любой другой наследник Stream
```

### 3. Элементы коллекций
Коллекция геометрических фигур может содержать различные подтипы фигур.

**Правильно:**
```javascript
// Коллекция геометрических фигур
const figures = [];
figures.push(new Square());
figures.push(new Triangle());
figures.push(new Rhombus());
```

**Неправильно:**
```javascript
// Нельзя добавлять несовместимые типы
figures.push(new Socket());  // Сокет - не геометрическая фигура
figures.push(new Button());  // Кнопка - не геометрическая фигура
```

---

## Связь с другими принципами

### LSP и Open/Closed Principle

Принцип подстановки тесно связан с **принципом открытости/закрытости** (Open/Closed):

- Когда мы хотим подставлять подтипы, мы приходим к необходимости расширять функциональность, не изменяя базовый код
- Подтип **может только расширять** функциональность базового типа
- Подтип **не должен удалять или изменять** поведение класса-предка

### LSP и контрактное программирование

Принцип подстановки подводит нас к **контрактному программированию**:

- Потребитель опирается на **интерфейс** (контракт)
- Потребитель не должен проверять конкретный тип объекта
- Все подтипы должны соблюдать контракт базового типа

**Примеры контрактов в JavaScript:**
- **Iterable** — объекты с методом `Symbol.iterator`
- **Thenable** — объекты с методом `then`
- **Serializable** — объекты с возможностью сериализации
- **EventEmitter** — объекты с методами `on`, `emit`

---

## Примеры в JavaScript

### Пример 1: Функции и асинхронные функции

```javascript
// Обычная функция
function regularFunction() {
  console.log('Regular function');
}

// Асинхронная функция
async function asyncFunction() {
  console.log('Async function');
}

// Проверка наследования
console.log(regularFunction.__proto__);        // Function
console.log(asyncFunction.__proto__);          // AsyncFunction
console.log(asyncFunction.__proto__.__proto__); // Function

// Цепочка прототипов:
// asyncFunction -> AsyncFunction -> Function -> Object

// setTimeout ожидает Function
setTimeout(regularFunction, 1000);  // ✓ Работает

// AsyncFunction является наследником Function
setTimeout(asyncFunction, 1000);    // ✓ Работает (LSP соблюдается)

// Object не является наследником Function
setTimeout({}, 1000);                // ✗ Не работает (нет метода call)
```

**Важно**: В этом примере `AsyncFunction` является подтипом `Function`, поэтому принцип подстановки соблюдается.

---

### Пример 2: Прямоугольник и квадрат (правильная реализация)

```javascript
// Базовый класс - прямоугольник
class Rect {
  constructor(width, height) {
    this.width = width;
    this.height = height;
  }

  get area() {
    return this.width * this.height;
  }
}

// Наследник - квадрат
class Square extends Rect {
  constructor(side) {
    super(side, side); // Вызываем конструктор родителя
    this.#side = side;
  }

  #side;

  // Переопределяем геттеры для width и height
  get width() {
    return this.#side;
  }

  get height() {
    return this.#side;
  }

  // Переопределяем сеттеры - оба устанавливают side
  set width(value) {
    this.#side = value;
  }

  set height(value) {
    this.#side = value;
  }
}

// Функция, работающая с Rect
function useRect(rect) {
  rect.width = 10;
  console.log('Area:', rect.area);
}

// Использование
const r1 = new Rect(10, 5);
const r2 = new Square(10);

useRect(r1); // Area: 50
useRect(r2); // Area: 100

// ✓ LSP соблюдается - функция работает с обоими типами
```

**Ключевой момент**: Мы не пишем проверки типа (`instanceof`), функция опирается на общий интерфейс.

**Антипаттерн** (нарушение LSP):
```javascript
// ✗ Плохо - проверка типа нарушает LSP
function useRect(rect) {
  if (rect instanceof Square) {
    // Особая логика для Square
  } else if (rect instanceof Rect) {
    // Логика для Rect
  }

  // Это приводит к "лапше" при появлении новых подтипов
}
```

**Правило**: Если вы видите проверки `instanceof` в коде, задумайтесь — не нарушен ли принцип подстановки?

---

### Пример 3: Банковский счет (правильное расширение)

```javascript
// Базовый класс - банковский счет
class Account extends EventEmitter {
  constructor(username, balance) {
    super();
    this.username = username;
    this.balance = balance;
  }

  withdraw(amount) {
    this.balance -= amount;
    this.emit('withdraw', { username: this.username, amount, balance: this.balance });
  }
}

// Подтип - пользовательский счет (правильная реализация)
class UserAccount extends Account {
  withdraw(amount) {
    // Сначала вызываем базовое поведение
    super.withdraw(amount);

    // Затем РАСШИРЯЕМ функциональность
    if (this.balance < 0) {
      this.emit('debt', Math.abs(this.balance));
    }
  }
}

// Использование
const account = new UserAccount('Marcus Aurelius', 100);

account.on('debt', (amount) => {
  console.log('Debt:', amount);
});

account.on('withdraw', ({ username, amount, balance }) => {
  console.log(`${username} withdrew ${amount}, balance: ${balance}`);
});

account.withdraw(50);  // Balance: 50
account.withdraw(70);  // Balance: -20, Debt: 20
account.withdraw(-10); // Balance: -10 (начисление)

// ✓ LSP соблюдается - поведение расширено, но не изменено
```

**Результат выполнения:**
```
Marcus Aurelius withdrew 50, balance: 50
Marcus Aurelius withdrew 70, balance: -20
Debt: 20
Marcus Aurelius withdrew -10, balance: -10
```

---

### Пример 4: Нарушение LSP

```javascript
// ✗ ПЛОХО - нарушение принципа подстановки
class BrokenUserAccount extends Account {
  withdraw(amount) {
    // Проверка блокирует дальнейшие операции
    if (this.balance <= 0) {
      return; // Ничего не делаем!
    }

    super.withdraw(amount);
  }
}

// Использование
const account = new BrokenUserAccount('Marcus Aurelius', 100);

account.on('withdraw', ({ username, amount, balance }) => {
  console.log(`${username} withdrew ${amount}, balance: ${balance}`);
});

account.withdraw(50);  // Balance: 50 ✓
account.withdraw(70);  // Balance: -20 ✓
account.withdraw(-10); // НЕ ВЫПОЛНИТСЯ! ✗

// ✗ LSP нарушен - поведение полностью изменилось
```

**Проблема**: После того, как баланс стал отрицательным, метод `withdraw` перестает работать **полностью** — невозможно ни снять, ни пополнить счет. Это нарушает ожидаемое поведение базового класса.

**Результат:**
```
Marcus Aurelius withdrew 50, balance: 50
Marcus Aurelius withdrew 70, balance: -20
// Третья операция не выполнилась!
```

---

### Пример 5: CancellablePromise

```javascript
// Расширяем Promise новой функциональностью
class CancellablePromise extends Promise {
  #cancelled = false;

  cancel() {
    this.#cancelled = true;
  }

  then(onResolve, onReject) {
    // Переопределяем then с проверкой отмены
    return super.then(
      (value) => {
        if (this.#cancelled) return;
        return onResolve(value);
      },
      (error) => {
        if (this.#cancelled) return;
        return onReject(error);
      }
    );
  }
}

// Использование
const promise = new CancellablePromise((resolve) => {
  setTimeout(() => {
    resolve('Done!');
  }, 10);
});

promise.then(console.log); // Работает как обычный Promise

// Можно использовать везде, где требуется Promise
await promise; // ✓ Работает

// ✓ LSP соблюдается - можем использовать везде, где нужен Promise
```

**Важно**:
- `CancellablePromise` **расширяет** функциональность, добавляя метод `cancel()`
- Он **не ломает** контракт Thenable (наличие метода `then`)
- Его можно использовать с `await` и во всех местах, где ожидается Promise

---

## Контракты в JavaScript

### Контракт Thenable

Любой объект с методом `then` может использоваться с `await`:

```javascript
// Создаем собственный thenable объект
const thenable = () => ({
  then: (onResolve) => {
    onResolve(5); // Сразу резолвим с значением 5
  }
});

// Использование с async/await
const value = await thenable();
console.log(value); // 5

// ✓ Контракт Thenable соблюден
```

**Объяснение**: Оператор `await` завязан на контракт Thenable, а не на конкретный тип Promise. Поэтому любой объект, реализующий этот контракт, будет работать.

---

### Контракт Iterable

Любой объект с методом `Symbol.iterator` можно использовать в циклах `for...of`:

```javascript
// Реализуем контракт Iterable
const iterable = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next() {
        if (i < 3) {
          return { value: i++, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
};

// Использование
for (const item of iterable) {
  console.log(item); // 0, 1, 2
}

// Spread оператор тоже работает
const arr = [...iterable]; // [0, 1, 2]

// ✓ Контракт Iterable соблюден
```

**Важно**: Цикл `for...of` — это не цикл по массиву, а цикл по итерируемому объекту. Массив просто реализует контракт Iterable.

---

### Array-like объекты

Объекты, похожие на массивы, но не являющиеся массивами:

```javascript
// NodeList из DOM
const divs = document.querySelectorAll('div');
console.log(divs); // NodeList - array-like объект

// Конвертация в обычный массив
const divsArray = Array.from(divs);

// Или через spread (использует Iterable контракт)
const divsArray2 = [...divs];
```

**Создание собственного array-like объекта:**

```javascript
// Array-like с контрактом длины и индексов
const arrayLike1 = {
  0: 'a',
  1: 'b',
  2: 'c',
  length: 3
};

// Работает обход через индексы
for (let i = 0; i < arrayLike1.length; i++) {
  console.log(arrayLike1[i]); // a, b, c
}

// Array.from работает
const arr1 = Array.from(arrayLike1); // ['a', 'b', 'c']

// ✗ for...of НЕ работает (нет Symbol.iterator)
// for (const item of arrayLike1) {} // TypeError!
```

**Array-like с контрактом Iterable:**

```javascript
// Array-like с Symbol.iterator
const arrayLike2 = {
  0: 'a',
  1: 'b',
  2: 'c',
  length: 3,

  [Symbol.iterator]() {
    let i = 0;
    return {
      next: () => {
        if (i < this.length) {
          return { value: this[i++], done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
};

// Теперь for...of работает
for (const item of arrayLike2) {
  console.log(item); // a, b, c
}

// Spread оператор работает
const arr2 = [...arrayLike2]; // ['a', 'b', 'c']

// Array.from тоже работает
const arr3 = Array.from(arrayLike2); // ['a', 'b', 'c']
```

**Вывод**:
- `Array.from` опирается на контракт "длина + индексы" ИЛИ Iterable
- `for...of` и `...spread` опирается на контракт Iterable (`Symbol.iterator`)
- Разные операции могут требовать разные контракты

---

## Обобщенное программирование и LSP (TypeScript)

```typescript
// Определяем интерфейс
interface Dated {
  year: number;
}

// Функция, работающая с Dated объектами
function later<T extends Dated>(a: T, b: T): T {
  return a.year > b.year ? a : b;
}

// Типы, реализующие Dated
interface Person extends Dated {
  name: string;
  year: number; // год рождения
}

interface Book extends Dated {
  title: string;
  author: string;
  year: number; // год публикации
}

// Создаем объекты
const marcus: Person = {
  name: 'Marcus Aurelius',
  year: 121
};

const seneca: Person = {
  name: 'Lucius Seneca',
  year: -4
};

const meditations: Book = {
  title: 'Meditations',
  author: 'Marcus Aurelius',
  year: 170
};

const discourse: Book = {
  title: 'Discourse on the Method',
  author: 'René Descartes',
  year: 1637
};

// Использование функции later
console.log(later(meditations, discourse));  // discourse (1637 > 170)
console.log(later(marcus, seneca));          // marcus (121 > -4)

// ✓ Можно даже сравнивать разные типы!
console.log(later(marcus, meditations));     // meditations (170 > 121)

// ✓ LSP соблюдается через интерфейс Dated
```

**Объяснение**:
- Дженерик `<T extends Dated>` означает, что T должен реализовывать интерфейс Dated
- Функция `later` может принимать **любой** подтип Dated
- Контракт (наличие поля `year: number`) обеспечивает корректную работу

---

## Ключевые выводы

### 1. Основное правило LSP
**Подтипы должны быть заменяемы на базовые типы без изменения корректности программы.**

### 2. Что можно делать в подтипах
✓ **Расширять** функциональность базового типа
✓ **Добавлять** новые методы и свойства
✓ **Уточнять** поведение через переопределение с вызовом `super`

### 3. Что НЕЛЬЗЯ делать в подтипах
✗ **Удалять** или блокировать методы базового типа
✗ **Изменять** ожидаемое поведение базовых методов
✗ **Нарушать** контракт базового типа

### 4. Признаки нарушения LSP

```javascript
// ✗ Плохо - проверка типов
if (obj instanceof SpecificType) {
  // специальная логика
}

// ✗ Плохо - множественные проверки
switch (obj.constructor.name) {
  case 'TypeA': /* ... */ break;
  case 'TypeB': /* ... */ break;
}
```

Если вы пишете такой код, скорее всего LSP нарушен.

### 5. Контрактное программирование

В JavaScript контракты важнее наследования:
- **Iterable** — `Symbol.iterator`
- **Thenable** — метод `then`
- **Array-like** — `length` + числовые индексы
- **EventEmitter** — методы `on`, `emit`

Объект, реализующий контракт, может использоваться везде, где этот контракт ожидается, даже без наследования.

### 6. LSP и полиморфизм

LSP обеспечивает **полиморфизм**:
```javascript
// Одна функция работает с разными типами
function processShape(shape) {
  console.log('Area:', shape.area);
  console.log('Perimeter:', shape.perimeter);
}

processShape(new Rectangle(10, 5));
processShape(new Circle(7));
processShape(new Triangle(3, 4, 5));

// ✓ Все типы соблюдают контракт Shape
```

---

## Практические рекомендации

### 1. Опирайтесь на интерфейсы, а не на типы
```javascript
// ✓ Хорошо
function process(iterable) {
  for (const item of iterable) {
    // ...
  }
}

// ✗ Плохо
function process(array) {
  if (!Array.isArray(array)) {
    throw new Error('Expected array');
  }
  // ...
}
```

### 2. Используйте композицию вместо изменения поведения
```javascript
// ✓ Хорошо - композиция
class ValidatedAccount {
  constructor(account) {
    this.account = account;
  }

  withdraw(amount) {
    if (this.account.balance < amount) {
      throw new Error('Insufficient funds');
    }
    return this.account.withdraw(amount);
  }
}

// ✗ Плохо - изменение поведения наследника
class BrokenAccount extends Account {
  withdraw(amount) {
    if (this.balance < amount) {
      return; // Ломает контракт!
    }
    super.withdraw(amount);
  }
}
```

### 3. Документируйте контракты
```javascript
/**
 * Интерфейс Shape
 * @interface
 * @property {number} area - Площадь фигуры
 * @property {number} perimeter - Периметр фигуры
 */

/**
 * Обрабатывает геометрическую фигуру
 * @param {Shape} shape - Объект, реализующий интерфейс Shape
 */
function processShape(shape) {
  // ...
}
```

---

## Заключение

Принцип подстановки Барбары Лисков — это не просто абстрактная теория типов, а практическое руководство по созданию гибких и расширяемых систем.

**Основная идея**: Код должен работать с абстракциями (интерфейсами, контрактами), а не с конкретными реализациями. Это позволяет:

- Безопасно добавлять новые типы без изменения существующего кода
- Избегать условной логики с проверками типов
- Создавать переиспользуемые компоненты
- Обеспечивать предсказуемое поведение системы

**Помните**: Если функция опирается на контракт, а не на конкретный тип, она автоматически становится более гибкой и поддерживает LSP.

**Цитата Роберта Мартина**:
> "Функции, которые используют базовый тип, должны иметь возможность использовать подтипы этого базового типа, не зная об этом."

Эта простая формулировка открывает путь к созданию элегантного, расширяемого и поддерживаемого кода.

---

## Дополнительные материалы

### Связанные принципы SOLID
- **S** — Single Responsibility Principle (Принцип единственной ответственности)
- **O** — Open/Closed Principle (Принцип открытости/закрытости)
- **L** — **Liskov Substitution Principle** (Принцип подстановки Лисков) ← Эта тема
- **I** — Interface Segregation Principle (Принцип разделения интерфейсов)
- **D** — Dependency Inversion Principle (Принцип инверсии зависимостей)

### Ключевые термины
- **Подтип (Subtype)** — тип, наследующий или реализующий другой тип
- **Контракт (Contract)** — набор ожидаемых свойств и методов
- **Полиморфизм (Polymorphism)** — возможность работы с разными типами через общий интерфейс
- **Ковариантность (Covariance)** — сохранение отношения наследования при передаче типов
- **Контравариантность (Contravariance)** — обращение отношения наследования

### Паттерны, использующие LSP
- **Strategy** — подменяемые алгоритмы
- **Template Method** — расширение поведения через наследование
- **Decorator** — расширение функциональности без изменения интерфейса
- **Adapter** — адаптация интерфейса к ожидаемому контракту
