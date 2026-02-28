# Паттерн «Команда» (Command) — Часть 2: Реализация на JavaScript

## Введение

Паттерн **Команда** (GoF — Gang of Four) позволяет описывать операции декларативно, в виде объектов-экземпляров. Это особенно полезно когда мы разрабатываем:

- текстовые или графические редакторы (история действий, отмена/повтор);
- электронные таблицы;
- банковские системы, где операции нужно записывать в лог и применять или откатывать по одной.

Паттерн тесно связан с принципами **CQS/CQRS** и **Event Sourcing**.

В этой лекции мы пройдём путь от классической реализации с классами до идиоматичного JavaScript-стиля, где классов минимум, а данные хранятся в простых структурах.

---

## Проблема

Нам нужно:

1. Выполнять операции над объектами (например, зачислять и списывать деньги с банковского счёта).
2. Хранить историю всех выполненных операций.
3. Иметь возможность **отменять (undo)** операции.
4. Иметь возможность **сериализовать** команды — записывать их в лог, базу данных или передавать на другой сервис.

Если операцию описывает просто функция, её сложно хранить, передавать и отменять. Паттерн Команда решает эту проблему, превращая каждую операцию в объект с чётко определённым интерфейсом.

---

## Решение

Паттерн вводит следующие роли:

| Роль | Описание | В примере |
|---|---|---|
| **Command** | Абстрактный класс или интерфейс команды | `AccountCommand` |
| **ConcreteCommand** | Конкретная команда с реализацией | `Withdraw`, `Income` |
| **Receiver / Target** | Объект, над которым выполняется команда | `Account` |
| **Invoker** | Объект, который хранит и вызывает команды | `Bank` |

В JavaScript классы можно сохранять в переменную, поэтому вместо дерева наследования для стратегий выбора типа команды часто достаточно простой коллекции (Map или объекта).

---

## Реализация

### Пример 1. Базовая версия: выполнение операций с историей

```javascript
// Абстрактный класс команды (только структура)
class AccountCommand {
  constructor(account, amount) {
    this.account = account;
    this.amount = amount;
  }
  execute() {
    throw new Error('Метод execute() должен быть реализован');
  }
}

// Конкретная команда: списание
class Withdraw extends AccountCommand {
  execute() {
    this.account.balance -= this.amount;
  }
}

// Конкретная команда: зачисление
class Income extends AccountCommand {
  execute() {
    this.account.balance += this.amount;
  }
}

// Receiver: банковский счёт
class Account {
  constructor(name) {
    this.name = name;
    this.balance = 0;
  }
}

// Invoker: банк хранит историю команд
class Bank {
  constructor() {
    // Map: ключ — время операции, значение — команда
    this.commands = new Map();
  }

  operate(account, amount) {
    // Выбираем тип команды: списание (< 0) или зачисление (> 0)
    const CommandClass = amount < 0 ? Withdraw : Income;
    const command = new CommandClass(account, Math.abs(amount));
    command.execute();
    this.commands.set(Date.now(), command);
  }

  showOperations() {
    console.table(
      [...this.commands.entries()].map(([time, cmd]) => ({
        time: new Date(time).toISOString(),
        account: cmd.account.name,
        type: cmd.constructor.name,
        amount: cmd.amount,
      }))
    );
  }
}

// Клиентский код
const bank = new Bank();
const marcus = new Account('Марк Аврелий');
const antoninus = new Account('Антонин Пий');

bank.operate(marcus, 1000);
bank.operate(marcus, -50);
bank.operate(antoninus, 500);
bank.operate(antoninus, -100);
bank.operate(antoninus, 150);

bank.showOperations();
console.log('Баланс Марка:', marcus.balance);       // 950
console.log('Баланс Антонина:', antoninus.balance); // 550
```

> Обратите внимание: в бухгалтерии все суммы традиционно хранятся как положительные числа, а тип операции (списание/зачисление) выносится в отдельное поле.

---

### Пример 2. Добавляем Undo — отмена операций

```javascript
class AccountCommand {
  constructor(account, amount) {
    this.account = account;
    this.amount = amount;
  }
}

class Withdraw extends AccountCommand {
  execute() { this.account.balance -= this.amount; }
  undo()    { this.account.balance += this.amount; }
}

class Income extends AccountCommand {
  execute() { this.account.balance += this.amount; }
  undo()    { this.account.balance -= this.amount; }
}

class Account {
  constructor(name) {
    this.name = name;
    this.balance = 0;
  }
}

class Bank {
  constructor() {
    this.commands = [];
  }

  operate(account, amount) {
    const CommandClass = amount < 0 ? Withdraw : Income;
    const command = new CommandClass(account, Math.abs(amount));
    command.execute();
    this.commands.push(command);
  }

  // Отмена последних N операций
  undoLast(count = 1) {
    for (let i = 0; i < count; i++) {
      const command = this.commands.pop();
      if (command) command.undo();
    }
  }

  showOperations() {
    console.table(
      this.commands.map((cmd) => ({
        account: cmd.account.name,
        type: cmd.constructor.name,
        amount: cmd.amount,
      }))
    );
  }
}

const bank = new Bank();
const marcus = new Account('Марк Аврелий');
const antoninus = new Account('Антонин Пий');

bank.operate(marcus, 1000);
bank.operate(marcus, -50);
bank.operate(antoninus, 500);
bank.operate(antoninus, -100);
bank.operate(antoninus, 150);

console.log('--- До отмены ---');
bank.showOperations();
console.log('Баланс Марка:', marcus.balance);       // 950
console.log('Баланс Антонина:', antoninus.balance); // 550

bank.undoLast(2);

console.log('--- После отмены 2 операций ---');
bank.showOperations();
console.log('Баланс Марка:', marcus.balance);
console.log('Баланс Антонина:', antoninus.balance);
```

Операции `execute` и `undo` — зеркальные: что одна прибавляет, другая отнимает.

---

### Пример 3. Анемичная команда: отделяем данные от логики

Чтобы команды можно было легко **сериализовать** (JSON, лог, БД), убираем из них методы `execute`/`undo`. Команда становится чистой структурой данных (**anemic object**).

```javascript
// Анемичная команда — только данные, никакой логики
function createCommand(operation, accountName, amount) {
  return { operation, accountName, amount };
}

// Коллекция обработчиков операций (аналог стратегий)
const operations = {
  withdraw: {
    execute: (account, amount) => { account.balance -= amount; },
    undo:    (account, amount) => { account.balance += amount; },
  },
  income: {
    execute: (account, amount) => { account.balance += amount; },
    undo:    (account, amount) => { account.balance -= amount; },
  },
};

class Account {
  constructor(name) {
    this.name = name;
    this.balance = 0;
  }
}

class Bank {
  constructor() {
    this.accounts = new Map();
    this.commands = [];
  }

  addAccount(account) {
    this.accounts.set(account.name, account);
  }

  getAccount(name) {
    return this.accounts.get(name);
  }

  operate(command) {
    const { operation, accountName, amount } = command;
    const handler = operations[operation];
    if (!handler) throw new Error(`Неизвестная операция: ${operation}`);

    const account = this.getAccount(accountName);
    if (!account) throw new Error(`Аккаунт не найден: ${accountName}`);

    handler.execute(account, amount);
    this.commands.push(command);
  }

  undoLast(count = 1) {
    for (let i = 0; i < count; i++) {
      const command = this.commands.pop();
      if (!command) break;
      const { operation, accountName, amount } = command;
      const account = this.getAccount(accountName);
      operations[operation].undo(account, amount);
    }
  }

  showOperations() {
    // Команды — просто массив объектов, console.table работает идеально
    console.table(this.commands);
  }
}

// Клиентский код
const bank = new Bank();
const marcus = new Account('Марк Аврелий');
const antoninus = new Account('Антонин Пий');
bank.addAccount(marcus);
bank.addAccount(antoninus);

bank.operate(createCommand('income',   'Марк Аврелий',   1000));
bank.operate(createCommand('withdraw', 'Марк Аврелий',     50));
bank.operate(createCommand('income',   'Антонин Пий',     500));
bank.operate(createCommand('withdraw', 'Антонин Пий',     100));
bank.operate(createCommand('income',   'Антонин Пий',     150));

console.log('--- Все операции ---');
bank.showOperations();
console.log('Баланс Марка:', marcus.balance);       // 950
console.log('Баланс Антонина:', antoninus.balance); // 550

bank.undoLast(2);

console.log('--- После отмены 2 операций ---');
bank.showOperations();
```

Теперь команды можно легко сохранять как JSON и воспроизводить на другом сервисе — это шаг в сторону **Event Sourcing**.

---

### Пример 4. Валидация команд: проверка перед выполнением

```javascript
const operations = {
  withdraw: {
    check:   (account, amount) => account.balance >= amount,
    execute: (account, amount) => { account.balance -= amount; },
    undo:    (account, amount) => { account.balance += amount; },
  },
  income: {
    check:   (_account, _amount) => true,
    execute: (account, amount) => { account.balance += amount; },
    undo:    (account, amount) => { account.balance -= amount; },
  },
};

class Bank {
  constructor() {
    this.accounts = new Map();
    this.commands = [];
  }

  addAccount(account) {
    this.accounts.set(account.name, account);
  }

  getAccount(name) {
    return this.accounts.get(name);
  }

  operate({ operation, accountName, amount }) {
    const handler = operations[operation];
    if (!handler) throw new Error(`Неизвестная операция: ${operation}`);

    const account = this.getAccount(accountName);
    if (!account) throw new Error(`Аккаунт не найден: ${accountName}`);

    const allowed = handler.check(account, amount);
    if (!allowed) {
      throw new Error(
        JSON.stringify({
          message: `Операция "${operation}" запрещена для аккаунта "${accountName}"`,
          balance: account.balance,
          requestedAmount: amount,
        })
      );
    }

    handler.execute(account, amount);
    this.commands.push({ operation, accountName, amount });
  }
}

// Демонстрация ошибки при недостатке средств
const bank = new Bank();
const marcus = new Account('Марк Аврелий');
bank.addAccount(marcus);

bank.operate({ operation: 'income',   accountName: 'Марк Аврелий', amount: 500 });
try {
  bank.operate({ operation: 'withdraw', accountName: 'Марк Аврелий', amount: 10000 }); // Ошибка!
} catch (e) {
  console.error('Ошибка операции:', e.message);
}
console.log('Баланс Марка:', marcus.balance); // 500 — операция не прошла
```

---

### Пример 5. JavaScript Way: минимум классов

В JavaScript нет нужды создавать классы там, где достаточно обычных функций и объектов. Итоговая «JS-way» версия:

```javascript
// Коллекция операций — объект с обработчиками
const operations = {
  withdraw: {
    check:   (account, amount) => account.balance >= amount,
    execute: (account, amount) => { account.balance -= amount; },
    undo:    (account, amount) => { account.balance += amount; },
  },
  income: {
    check:   () => true,
    execute: (account, amount) => { account.balance += amount; },
    undo:    (account, amount) => { account.balance -= amount; },
  },
};

// Аккаунты — обычный Map, без класса
const accs = new Map();

function addAccount(name) {
  accs.set(name, { name, balance: 0 });
}

function getAccount(name) {
  return accs.get(name);
}

// История команд — массив простых объектов
const commands = [];

// Invoker — функции вместо методов класса Bank
function operate(operation, accountName, amount) {
  const handler = operations[operation];
  if (!handler) throw new Error(`Неизвестная операция: ${operation}`);

  const account = getAccount(accountName);
  if (!account) throw new Error(`Аккаунт не найден: ${accountName}`);

  if (!handler.check(account, amount)) {
    throw new Error(
      JSON.stringify({ message: 'Операция запрещена', accountName, operation, amount })
    );
  }

  handler.execute(account, amount);
  commands.push({ operation, accountName, amount });
}

function undoLast(count = 1) {
  for (let i = 0; i < count; i++) {
    const cmd = commands.pop();
    if (!cmd) break;
    const account = getAccount(cmd.accountName);
    operations[cmd.operation].undo(account, cmd.amount);
  }
}

// Использование
addAccount('Марк Аврелий');
addAccount('Антонин Пий');

operate('income',   'Марк Аврелий',  1000);
operate('withdraw', 'Марк Аврелий',    50);
operate('income',   'Антонин Пий',    500);
operate('withdraw', 'Антонин Пий',    100);
operate('income',   'Антонин Пий',    150);

console.log('--- Все операции ---');
console.table(commands);
console.log('Баланс Марка:', getAccount('Марк Аврелий').balance);   // 950
console.log('Баланс Антонина:', getAccount('Антонин Пий').balance); // 550

undoLast(2);

console.log('--- После отмены 2 операций ---');
console.table(commands);
console.log('Баланс Марка:', getAccount('Марк Аврелий').balance);
console.log('Баланс Антонина:', getAccount('Антонин Пий').balance);
```

Паттерн остался тем же самым — изменился лишь способ его выражения. В JavaScript функции являются объектами первого класса, а коллекции (Map, Array) — удобным инструментом для хранения команд и их истории без избыточных классов.

---

## Применение

Паттерн Команда применяют когда:

- Нужна **история операций** с возможностью отмены и повтора (текстовые/графические редакторы, электронные таблицы).
- Нужно **логировать операции** — записывать их в файл, базу данных или очередь сообщений.
- Нужно **передавать операции** между сервисами и применять их в другом контексте (микросервисы, очереди задач).
- Нужна **транзакционность** — возможность откатить группу операций при ошибке.
- Реализуется **Event Sourcing** — состояние системы восстанавливается повторным применением сохранённых команд.

Связанные паттерны и принципы:

- **CQS (Command Query Separation)** — разделение команд (изменяют состояние) и запросов (читают состояние).
- **CQRS** — архитектурное развитие CQS.
- **Event Sourcing** — хранение истории событий вместо текущего состояния.
- **Strategy** — схожая структура, но Стратегия описывает алгоритм, а Команда — операцию с историей.

---

## Итог

В ходе лекции мы прошли путь эволюции паттерна Команда в шесть шагов:

| Шаг | Что добавили |
|-----|-------------|
| 1 | Базовые команды с `execute`, история операций |
| 2 | Метод `undo` — отмена последних N операций |
| 3 | Анемичные команды (чистые структуры данных), логика вынесена в коллекцию обработчиков |
| 4 | Коллекция аккаунтов в банке, разрыв прямой ссылки команды на объект |
| 5 | Валидация команд (`check`) и выброс ошибки с деталями в JSON |
| 6 | JavaScript Way: убраны лишние классы, всё заменено функциями и коллекциями |

**Ключевой вывод:** паттерн — это не структура классов, а решаемая задача. В JavaScript за счёт функций первого класса и гибких коллекций реализация получается значительно компактнее канонического объектно-ориентированного варианта, но при этом решает ту же задачу и сохраняет все преимущества паттерна.
