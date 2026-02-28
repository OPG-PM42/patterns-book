# Паттерн Command (Команда) — Часть 1

> Поведенческий паттерн из классической книги «Банды четырёх» (GoF).
> Лекция на TypeScript.

---

## Введение

Паттерн **Command** («Команда») — это поведенческий паттерн проектирования из книги *Design Patterns* («Банды четырёх»). Он превращает запросы или простые операции в объекты-данные (команды), которые можно передавать, хранить, откатывать и воспроизводить.

Паттерн широко применяется в самых разных языках и платформах — от Delphi и C++ Builder до современного TypeScript. Его принципы универсальны и не зависят от конкретного языка программирования.

---

## Проблема

В системах часто встречается следующая ситуация: есть **отправитель** (Invoker) — тот, кто хочет вызвать какую-то операцию, и есть **получатель** (Receiver) — тот, кто должен её выполнить.

```
Invoker ──вызывает──► Receiver
```

Самый простой способ — вызывать методы друг у друга напрямую или передавать события. Но это создаёт **сильную связанность** (coupling): Invoker знает о конкретном классе Receiver и наоборот.

Проблемы сильной связанности:

- Трудно заменить одну реализацию другой без изменения обеих сторон.
- Невозможно поставить операцию в очередь или записать историю вызовов.
- Нельзя легко реализовать отмену (undo) и повтор (redo) операций.
- Сложно передать операцию в другую систему или сохранить на диск.

---

## Решение

Паттерн **Command** предлагает выделить описание операции в отдельную **структуру данных** — объект-команду (Command object). Это так называемый **анемичный объект** (Anemic Object) или **DTO** (Data Transfer Object): он хранит данные о том, *что* нужно сделать, но не знает *как*.

```
Invoker ──создаёт──► Command ──передаёт──► Receiver
```

Теперь:
- Invoker знает только о структуре Command.
- Receiver знает только о структуре Command.
- Invoker и Receiver **не знают друг о друге**.

Это аналогично разделению через интерфейсы: все стороны знают про контракт (интерфейс или структуру данных), но не зависят напрямую друг от друга.

### Что открывает этот паттерн

Сериализация команд в объекты открывает путь к целому ряду мощных архитектурных решений:

| Паттерн / Приём | Что даёт |
|---|---|
| **CQS** (Command Query Separation) | Разделение команд (изменений) и запросов (чтений) |
| **CQRS** (Command Query Responsibility Segregation) | Разделение слоёв чтения и записи, доступ к БД |
| **Event Sourcing** | Хранение истории всех изменений, масштабируемость |
| **Saga** | Откат распределённых транзакций |

### Практические возможности

- **Логирование**: автоматическая запись всех действий в системе.
- **Undo / Redo**: движение по истории вперёд и назад.
- **Очередь команд**: постановка операций в очередь для асинхронной обработки.
- **Передача по сети**: отправка команды в другую систему для выполнения там.
- **Сохранение на диск / в БД**: персистентность истории операций.
- **Синхронизация систем**: команды можно накатить на другую базу данных.

### Недостатки

- Увеличивается количество классов и структур в коде.
- Незначительно растёт потребление памяти и процессора.

Тем не менее польза от паттерна в подходящих задачах значительно превышает его стоимость.

---

## Реализация

Рассмотрим два варианта реализации на TypeScript: классический (ООП с абстрактным классом) и современный (через интерфейсы и анемичные структуры).

### Вариант 1. Классический ООП-подход

В этом варианте используется абстрактный класс `Command` с методами `exec()` и `undo()`, от которого наследуются конкретные команды.

```typescript
// --- Модель данных ---

class BankAccount {
  constructor(
    public name: string,
    public balance: number
  ) {}
}

// --- Абстрактная команда ---

abstract class Command {
  constructor(
    public account: BankAccount,
    public amount: number,
    public operation: string
  ) {}

  abstract exec(): void;
  abstract undo(): void;
}

// --- Конкретная команда: пополнение счёта ---

class IncomeCommand extends Command {
  constructor(account: BankAccount, amount: number) {
    super(account, amount, 'income');
  }

  exec(): void {
    this.account.balance += this.amount;
    console.log(`[+] ${this.account.name}: +${this.amount} => ${this.account.balance}`);
  }

  undo(): void {
    this.account.balance -= this.amount;
    console.log(`[undo+] ${this.account.name}: -${this.amount} => ${this.account.balance}`);
  }
}

// --- Конкретная команда: списание со счёта ---

class WithdrawCommand extends Command {
  constructor(account: BankAccount, amount: number) {
    super(account, amount, 'withdraw');
  }

  exec(): void {
    this.account.balance -= this.amount;
    console.log(`[-] ${this.account.name}: -${this.amount} => ${this.account.balance}`);
  }

  undo(): void {
    this.account.balance += this.amount;
    console.log(`[undo-] ${this.account.name}: +${this.amount} => ${this.account.balance}`);
  }
}

// --- Инвокер: Банк хранит историю команд ---

class Bank {
  private history: Command[] = [];

  operate(account: BankAccount, value: number): void {
    // Выбираем класс команды в зависимости от знака суммы
    const CommandClass = value > 0 ? IncomeCommand : WithdrawCommand;
    const command = new CommandClass(account, Math.abs(value));
    command.exec();
    this.history.push(command);
  }

  undo(steps: number): void {
    for (let i = 0; i < steps; i++) {
      const command = this.history.pop();
      if (command) command.undo();
    }
  }

  showOperations(): void {
    console.log('\n--- История операций ---');
    console.table(
      this.history.map((c) => ({
        account: c.account.name,
        operation: c.operation,
        amount: c.amount,
        balance: c.account.balance,
      }))
    );
  }
}

// --- Использование ---

const bank = new Bank();
const alice = new BankAccount('Alice', 1000);
const bob = new BankAccount('Bob', 500);

bank.operate(alice, 200);   // Alice: +200
bank.operate(alice, -50);   // Alice: -50
bank.operate(bob, 300);     // Bob: +300
bank.operate(bob, -100);    // Bob: -100
bank.operate(alice, 150);   // Alice: +150

bank.showOperations();      // 5 операций

bank.undo(2);               // откатываем 2 последних

bank.showOperations();      // 3 операции
```

---

### Вариант 2. Современный TypeScript-подход (анемичные команды + Record)

В этом варианте команды являются **чистыми структурами данных** (анемичными объектами), а поведение хранится отдельно — в коллекции обработчиков. Это более гибкий и идиоматичный подход для TypeScript.

```typescript
// --- Тип аккаунта ---

type Account = {
  name: string;
  balance: number;
};

// --- Коллекция аккаунтов ---

type Accounts = Map<string, Account>;

const accounts: Accounts = new Map();

function addAccount(name: string, initialBalance: number): void {
  accounts.set(name, { name, balance: initialBalance });
}

// --- Анемичная команда (только данные, никакого поведения) ---

type OperationType = 'withdraw' | 'income';

interface Command {
  account: string;  // идентификатор аккаунта
  amount: number;   // абсолютное значение суммы
  type: OperationType;
}

// --- Интерфейс операции (поведение) ---

interface Operation {
  exec(command: Command): void;
  undo(command: Command): void;
}

// --- Коллекция обработчиков: поведение хранится отдельно ---

type Operations = Record<OperationType, Operation>;

const operations: Operations = {
  income: {
    exec(command: Command): void {
      const acc = accounts.get(command.account);
      if (!acc) throw new Error(`Account not found: ${command.account}`);
      acc.balance += command.amount;
      console.log(`[+] ${acc.name}: +${command.amount} => ${acc.balance}`);
    },
    undo(command: Command): void {
      const acc = accounts.get(command.account);
      if (!acc) throw new Error(`Account not found: ${command.account}`);
      acc.balance -= command.amount;
      console.log(`[undo+] ${acc.name}: -${command.amount} => ${acc.balance}`);
    },
  },

  withdraw: {
    exec(command: Command): void {
      const acc = accounts.get(command.account);
      if (!acc) throw new Error(`Account not found: ${command.account}`);
      acc.balance -= command.amount;
      console.log(`[-] ${acc.name}: -${command.amount} => ${acc.balance}`);
    },
    undo(command: Command): void {
      const acc = accounts.get(command.account);
      if (!acc) throw new Error(`Account not found: ${command.account}`);
      acc.balance += command.amount;
      console.log(`[undo-] ${acc.name}: +${command.amount} => ${acc.balance}`);
    },
  },
};

// --- Банк: хранит историю анемичных команд ---

class Bank {
  private history: Command[] = [];

  operate(accountName: string, value: number): void {
    // Определяем тип операции по знаку суммы
    const type: OperationType = value >= 0 ? 'income' : 'withdraw';
    const command: Command = {
      account: accountName,
      amount: Math.abs(value),
      type,
    };

    // Выполняем операцию через коллекцию обработчиков
    operations[type].exec(command);

    // Сохраняем анемичную команду в историю
    this.history.push(command);
  }

  undo(steps: number): void {
    for (let i = 0; i < steps; i++) {
      const command = this.history.pop();
      if (command) {
        operations[command.type].undo(command);
      }
    }
  }

  showOperations(): void {
    console.log('\n--- История операций ---');
    console.table(this.history);
  }
}

// --- Использование ---

addAccount('Alice', 1000);
addAccount('Bob', 500);

const bank = new Bank();

bank.operate('Alice', 200);   // Alice: +200
bank.operate('Alice', -50);   // Alice: -50
bank.operate('Bob', 300);     // Bob: +300
bank.operate('Bob', -100);    // Bob: -100
bank.operate('Alice', 150);   // Alice: +150

bank.showOperations();        // 5 команд в истории

bank.undo(2);                 // откатываем 2 операции

bank.showOperations();        // 3 команды в истории
```

---

## Применение

Паттерн Command уместен в следующих ситуациях:

1. **Нужна история операций** — когда требуется знать, что было сделано в системе (аудит, логирование).
2. **Нужен Undo/Redo** — редакторы текста, графические редакторы, IDE.
3. **Асинхронная обработка** — команды ставятся в очередь (message queue) и выполняются позднее.
4. **Распределённые системы** — команды сериализуются и передаются между сервисами (микросервисы, event-driven архитектура).
5. **Синхронизация данных** — одна и та же история команд накатывается на несколько баз данных.
6. **Транзакции с откатом** — реализация паттерна Saga для распределённых транзакций.

### Отличие двух вариантов реализации

| Критерий | Классический (абстрактный класс) | Современный (анемичные объекты) |
|---|---|---|
| Поведение | Внутри команды | Отдельно в коллекции обработчиков |
| Расширяемость | Новый класс-наследник | Новый ключ в `Record` |
| Сериализация | Затруднена (классы) | Легко (plain objects) |
| Зависимость от ресивера | Команда знает о BankAccount | Команда знает только `string` id |
| TypeScript-идиоматичность | Хорошо | Очень хорошо |

---

## Итог

Паттерн **Command** решает проблему связанности между отправителем и получателем операции, вводя промежуточную структуру данных — **команду**.

Ключевые идеи:

- Команда — это **данные**, а не поведение. Поведение хранится отдельно.
- Обе стороны (Invoker и Receiver) знают только о **структуре команды**, но не друг о друге.
- История команд позволяет реализовать **undo/redo**, **логирование**, **очереди**, **синхронизацию систем**.
- В TypeScript предпочтительнее использовать **анемичные объекты** вместо классов — это упрощает сериализацию и передачу данных.

Этот паттерн является фундаментом для таких архитектурных стилей как CQS, CQRS, Event Sourcing и Saga, которые широко используются в современных распределённых системах.

---

> Следующая часть лекции: реализация паттерна Command на JavaScript на том же примере.
