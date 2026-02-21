# Thenable контракт (promise-like) в JavaScript

## Обзор

Thenable — это абстракция, которая исторически предшествовала появлению Promise в JavaScript. Это упрощенный promise-like интерфейс, который можно создавать вручную и который обладает многими возможностями промисов. Несмотря на существование нативных Promise, thenable контракт остается полезным инструментом в современной разработке.

**Ключевые отличия от Promise:**
- Может выполняться многократно (Promise резолвится только один раз)
- Более простая реализация
- Полная совместимость с async/await синтаксисом
- Подходит для создания специализированных абстракций (query builders, ленивые вычисления)

---

## Что такое Thenable?

**Thenable** — это любой объект или класс, который имеет метод `then`. Это минимальный контракт, необходимый для работы с асинхронным кодом в JavaScript.

### Минимальная структура thenable

```javascript
const thenable = {
  then(onFulfilled) {
    // Асинхронная операция
    setTimeout(() => {
      onFulfilled('результат');
    }, 1000);
  }
};
```

**Важно:** Объект считается thenable, если у него есть метод `then`, принимающий callback-функцию.

---

## Базовый пример использования

### Простейший thenable объект

```javascript
// Создаем thenable объект
const delay = {
  then(onFulfilled) {
    setTimeout(() => {
      onFulfilled('Выполнено через 1 секунду');
    }, 1000);
  }
};

// Использование с async/await
(async () => {
  const a = await delay;
  console.log(a); // "Выполнено через 1 секунду"

  const b = await delay;
  console.log(b); // Снова ждет 1 секунду и выводит то же

  const c = await delay;
  console.log(c); // И снова ждет...

  const d = await delay;
  console.log(d); // Каждый await запускает операцию заново!
})();
```

**Ключевое отличие от Promise:** Один и тот же thenable объект можно использовать многократно. Каждый вызов `await` запускает операцию заново, в отличие от Promise, который резолвится только один раз.

---

## Thenable через функцию-фабрику

Более практичный подход — создание thenable через фабричную функцию.

### Пример: получение данных пользователя

```javascript
// Фабричная функция, возвращающая thenable
function getPerson(id) {
  return {
    then(onFulfilled) {
      // Имитация асинхронной операции (запрос к API/БД)
      setTimeout(() => {
        const person = {
          id: id,
          name: `User ${id}`,
          email: `user${id}@example.com`
        };
        onFulfilled(person);
      }, 1000);
    }
  };
}

// Использование
(async () => {
  const user1 = await getPerson(10);
  console.log(user1); // { id: 10, name: 'User 10', email: 'user10@example.com' }

  const user2 = await getPerson(25);
  console.log(user2); // { id: 25, name: 'User 25', email: 'user25@example.com' }
})();
```

**Преимущество:** Каждый вызов `getPerson(id)` создает новый thenable с разными параметрами, что делает код более похожим на работу с Promise.

---

## Thenable через классы

Класс позволяет создавать более сложные thenable объекты с внутренним состоянием.

### Базовая реализация класса

```javascript
class Result {
  constructor() {
    this.callback = null;
  }

  // Метод then сохраняет callback
  then(onFulfilled) {
    this.callback = onFulfilled;
    return this;
  }

  // Метод для вызова callback при готовности данных
  ready(data) {
    if (this.callback) {
      this.callback(data);
    }
  }
}

// Пример использования
const result = new Result();

// Регистрируем обработчик
result.then((data) => {
  console.log('Получены данные:', data);
});

// Позже, когда данные готовы
setTimeout(() => {
  result.ready({ status: 'success', value: 42 });
}, 2000);
```

---

## Практический пример: обертка для доступа к данным

### Класс Security для работы с пользователями

```javascript
class Result {
  constructor() {
    this.callback = null;
  }

  then(onFulfilled) {
    this.callback = onFulfilled;
    return this;
  }

  ready(data) {
    if (this.callback) {
      this.callback(data);
    }
  }
}

class Security {
  // Метод получения пользователя по ID
  getPerson(id) {
    const res = new Result();

    // Имитация асинхронного запроса к БД
    setTimeout(() => {
      // В реальном приложении здесь был бы запрос к базе данных
      const person = {
        id: id,
        name: `Person ${id}`
      };

      // Когда данные готовы, вызываем ready
      res.ready(person);
    }, 1000);

    return res;
  }
}

// Использование
const security = new Security();

(async () => {
  const person = await security.getPerson(10);
  console.log(person); // { id: 10, name: 'Person 10' }
})();
```

**Применение:** Этот паттерн удобен для создания query builders, ORM систем и других абстракций доступа к данным.

---

## Обработка ошибок в Thenable

Метод `then` может принимать два аргумента: `onFulfilled` и `onRejected`.

### Пример с обработкой ошибок

```javascript
function getNumbers() {
  const numbers = [10, 20, 30];
  let index = 0;

  return {
    then(onFulfilled, onRejected) {
      setTimeout(() => {
        if (index < numbers.length) {
          // Если элементы еще есть, возвращаем следующий
          onFulfilled(numbers[index++]);
        } else {
          // Если элементы закончились, генерируем ошибку
          const error = new Error('Больше чисел нет');
          onRejected(error);
        }
      }, 100);
    }
  };
}

// Использование
(async () => {
  const next = getNumbers();

  try {
    // Первые 3 итерации успешны
    for (let i = 0; i < 5; i++) {
      const num = await next;
      console.log(`Итерация ${i + 1}:`, num);
    }
  } catch (error) {
    // На 4-й итерации попадаем сюда
    console.error('Ошибка:', error.message); // "Больше чисел нет"
  }
})();

// Вывод:
// Итерация 1: 10
// Итерация 2: 20
// Итерация 3: 30
// Ошибка: Больше чисел нет
```

**Важно:** Thenable может быть использован многократно, что позволяет реализовывать итераторы и генераторы данных.

---

## Чейнинг (цепочки вызовов)

Thenable поддерживает цепочки вызовов, возвращая `this` из метода `then`.

### Простой чейнинг

```javascript
function getNumbers() {
  const numbers = [10, 20, 30];
  let index = 0;

  return {
    then(onFulfilled, onRejected) {
      setTimeout(() => {
        if (index < numbers.length) {
          onFulfilled(numbers[index++]);
        } else {
          onRejected(new Error('Больше чисел нет'));
        }
      }, 100);

      // Возвращаем this для поддержки чейнинга
      return this;
    }
  };
}

// Функции-обработчики
const onSuccess = (data) => console.log('Получено:', data);
const onError = (error) => console.error('Ошибка:', error.message);

// Использование с чейнингом
getNumbers()
  .then(onSuccess, onError)
  .then(onSuccess, onError)
  .then(onSuccess, onError)
  .then(onSuccess, onError)
  .then(onSuccess, onError);

// Вывод:
// Получено: 10
// Получено: 20
// Получено: 30
// Ошибка: Больше чисел нет
// Ошибка: Больше чисел нет
```

---

## Продвинутый чейнинг с созданием цепочки объектов

Для более сложных сценариев можно создавать цепочку связанных thenable объектов.

### Реализация рекурсивного thenable

```javascript
class Thenable {
  constructor() {
    this.onSuccess = null;
    this.next = null; // Ссылка на следующий thenable в цепочке
  }

  then(onSuccess) {
    // Создаем новый экземпляр и связываем его с текущим
    this.next = new Thenable();
    this.onSuccess = onSuccess;
    return this.next;
  }

  resolve(value) {
    // Если есть обработчик успеха
    if (this.onSuccess) {
      const result = this.onSuccess(value);

      // Если обработчик вернул значение и есть следующий в цепочке
      if (result !== undefined && this.next) {
        // Проверяем, что у next есть метод then (соблюдение контракта)
        if (typeof this.next.then === 'function') {
          // Продолжаем цепочку
          this.next.resolve(result);
        }
      }
    }
  }
}

// Пример использования
const th = new Thenable();

th.then((x) => {
  console.log('Шаг 1:', x);
  return x * 2;
}).then((x) => {
  console.log('Шаг 2:', x);
  return x + 10;
}).then((x) => {
  console.log('Шаг 3:', x);
  return x / 2;
});

th.resolve(5);

// Вывод:
// Шаг 1: 5
// Шаг 2: 10
// Шаг 3: 20
```

**Принцип работы:** Создается односвязный список thenable объектов, где каждый узел хранит ссылку на следующий (`next`).

---

## Практический пример: обертка для fs.readFile

Thenable можно использовать для оборачивания callback-based API в promise-like интерфейс.

### Обертка для чтения файлов

```javascript
const fs = require('fs');

class Thenable {
  constructor() {
    this.onSuccess = null;
    this.next = null;
  }

  then(onSuccess) {
    this.next = new Thenable();
    this.onSuccess = onSuccess;
    return this.next;
  }

  resolve(value) {
    if (this.onSuccess) {
      const result = this.onSuccess(value);
      if (result !== undefined && this.next && typeof this.next.then === 'function') {
        this.next.resolve(result);
      }
    }
  }
}

// Функция чтения файла, возвращающая thenable
function readFile(filename) {
  const thenable = new Thenable();

  // Используем классический callback API Node.js
  fs.readFile(filename, 'utf8', (error, data) => {
    if (error) {
      console.error('Ошибка чтения файла:', error);
    } else {
      thenable.resolve(data);
    }
  });

  return thenable;
}

// Последовательное чтение файлов
readFile('./file1.txt')
  .then((data1) => {
    console.log('Файл 1:', data1);
    return readFile('./file2.txt');
  })
  .then((data2) => {
    console.log('Файл 2:', data2);
    return readFile('./file3.txt');
  })
  .then((data3) => {
    console.log('Файл 3:', data3);
  });
```

**Преимущество:** Преобразование callback-based кода в цепочку вызовов без использования нативных Promise.

---

## Query Builder на основе Thenable

Одно из самых практичных применений thenable — создание fluent API для построения запросов.

### Реализация простого Query Builder

```javascript
class Query {
  constructor() {
    this.table = '';
    this.conditions = [];
    this.orderField = '';
    this.limitValue = null;
  }

  // Указываем таблицу
  from(tableName) {
    this.table = tableName;
    return this; // Возвращаем this для чейнинга
  }

  // Добавляем условие WHERE
  where(condition) {
    this.conditions.push(condition);
    return this;
  }

  // Добавляем сортировку
  order(field) {
    this.orderField = field;
    return this;
  }

  // Добавляем ограничение
  limit(count) {
    this.limitValue = count;
    return this;
  }

  // Метод then делает объект thenable
  then(onFulfilled) {
    // Формируем SQL запрос
    let sql = `SELECT * FROM ${this.table}`;

    if (this.conditions.length > 0) {
      sql += ` WHERE ${this.conditions.join(' AND ')}`;
    }

    if (this.orderField) {
      sql += ` ORDER BY ${this.orderField}`;
    }

    if (this.limitValue !== null) {
      sql += ` LIMIT ${this.limitValue}`;
    }

    // Имитация асинхронного запроса к БД
    setTimeout(() => {
      console.log('Выполняется SQL:', sql);

      // Имитация результата из БД
      const result = [
        { id: 1, name: 'Alice', age: 30 },
        { id: 2, name: 'Bob', age: 25 }
      ];

      onFulfilled(result);
    }, 100);
  }
}

// Использование Query Builder
(async () => {
  const result = await new Query()
    .from('users')
    .where('age > 18')
    .order('name')
    .limit(10);

  console.log('Результат:', result);
})();

// Вывод:
// Выполняется SQL: SELECT * FROM users WHERE age > 18 ORDER BY name LIMIT 10
// Результат: [
//   { id: 1, name: 'Alice', age: 30 },
//   { id: 2, name: 'Bob', age: 25 }
// ]
```

**Важные моменты:**
- Методы `from`, `where`, `order`, `limit` возвращают `this` для поддержки fluent API
- Метод `then` делает объект thenable, позволяя использовать его с `await`
- SQL запрос собирается только при вызове `then` (ленивое вычисление)
- Легко расширить до реального взаимодействия с базой данных

### Расширенная версия с реальным подключением к БД

```javascript
class DatabaseQuery {
  constructor(db) {
    this.db = db; // Подключение к БД
    this.table = '';
    this.conditions = [];
    this.orderField = '';
    this.limitValue = null;
  }

  from(tableName) {
    this.table = tableName;
    return this;
  }

  where(condition) {
    this.conditions.push(condition);
    return this;
  }

  order(field) {
    this.orderField = field;
    return this;
  }

  limit(count) {
    this.limitValue = count;
    return this;
  }

  then(onFulfilled, onRejected) {
    // Формируем SQL
    let sql = `SELECT * FROM ${this.table}`;

    if (this.conditions.length > 0) {
      sql += ` WHERE ${this.conditions.join(' AND ')}`;
    }

    if (this.orderField) {
      sql += ` ORDER BY ${this.orderField}`;
    }

    if (this.limitValue !== null) {
      sql += ` LIMIT ${this.limitValue}`;
    }

    // Выполняем реальный запрос к БД
    this.db.query(sql, (error, results) => {
      if (error) {
        if (onRejected) {
          onRejected(error);
        }
      } else {
        onFulfilled(results);
      }
    });
  }
}

// Использование с реальной БД
// const db = require('mysql').createConnection({...});
// const result = await new DatabaseQuery(db)
//   .from('users')
//   .where('active = 1')
//   .order('created_at DESC')
//   .limit(20);
```

---

## Сравнение Thenable и Promise

| Характеристика | Thenable | Promise |
|---------------|----------|---------|
| Многократное выполнение | Да, каждый await запускает заново | Нет, резолвится только один раз |
| Минимальный API | Только метод `then` | then, catch, finally + статические методы |
| Совместимость с async/await | Полная | Полная |
| Обработка ошибок | Через второй аргумент `then` | Через `catch` или второй аргумент `then` |
| Стандартизация | Не стандартизирован | Часть спецификации ES6+ |
| Сложность реализации | Простая | Более сложная (state machine) |
| Цепочки вызовов | Требует ручной реализации | Встроенная поддержка |

---

## Когда использовать Thenable

### Подходящие сценарии:

1. **Query Builders** — создание fluent API для построения запросов
2. **Ленивые вычисления** — отложенное выполнение операций
3. **Итераторы** — многократное получение данных
4. **Обертки для legacy API** — адаптация callback-based кода
5. **Кастомные асинхронные абстракции** — специализированные контракты

### Когда лучше использовать Promise:

1. Операция должна выполниться только один раз
2. Нужна стандартная обработка ошибок через `catch`
3. Требуются методы `Promise.all`, `Promise.race`, и т.д.
4. Важна совместимость и стандартизация

---

## Ключевые выводы

1. **Thenable — это минимальный контракт**: любой объект с методом `then` может использоваться с async/await

2. **Многократное использование**: в отличие от Promise, thenable может выполняться многократно

3. **Гибкость реализации**: можно создавать как простые объекты, так и сложные классы

4. **Чейнинг**: поддерживается через возврат `this` или создание цепочки объектов

5. **Обработка ошибок**: через второй аргумент `then(onFulfilled, onRejected)`

6. **Практическое применение**: query builders, обертки для callback API, ленивые вычисления

7. **Исторический контекст**: thenable предшествовал Promise и остается полезным инструментом

---

## Дополнительные ресурсы

**Связанные темы для изучения:**
- Promise API и внутреннее устройство
- Async/await синтаксис
- Генераторы и итераторы
- Паттерн Builder
- Fluent Interface
- Обработка асинхронности в JavaScript

**Практические задания:**
1. Реализовать thenable для HTTP запросов
2. Создать query builder для нескольких типов БД
3. Написать thenable-обертку для событийной модели
4. Реализовать систему middleware на основе thenable
5. Создать ленивый вычислитель выражений с thenable интерфейсом
