# Promise контракт в JavaScript — Полный конспект

## Оглавление
1. [Введение в Promise](#введение-в-promise)
2. [Конструктор Promise](#конструктор-promise)
3. [Состояния Promise](#состояния-promise)
4. [Методы экземпляра Promise](#методы-экземпляра-promise)
5. [Цепочки Promise (Promise Chaining)](#цепочки-promise-promise-chaining)
6. [Практическое применение: работа с файлами](#практическое-применение-работа-с-файлами)
7. [Статические методы класса Promise](#статические-методы-класса-promise)
8. [Обработка ошибок](#обработка-ошибок)
9. [Best Practices и важные моменты](#best-practices-и-важные-моменты)

---

## Введение в Promise

**Promise (промис)** — это объект, который символизирует результат отложенной (асинхронной) операции. Промис представляет собой контейнер со значением, которое будет доступно в будущем.

### Основные характеристики:
- Представляет результат асинхронной операции
- Имеет встроенные методы для обработки результатов
- Позволяет избежать "callback hell" (ад колбэков)
- Поддерживает цепочные вызовы (chaining)

---

## Конструктор Promise

### Синтаксис и работа конструктора

Конструктор Promise принимает функцию с двумя аргументами: **resolve** и **reject**. Это так называемый **"открытый конструктор"** (revealing constructor pattern).

```javascript
const promise = new Promise((resolve, reject) => {
  // Асинхронная операция
  // Вызовите resolve(value) при успехе
  // Вызовите reject(error) при ошибке
});
```

### Пример с успешным выполнением

```javascript
const successPromise = new Promise((resolve, reject) => {
  // Когда промис готов "развиться" (перейти в финальное состояние),
  // вызывается одна из функций: resolve или reject
  resolve(100);
});

// Просмотр промиса в консоли
console.dir(successPromise);
// Promise { <fulfilled>: 100 }
```

**Важно:** Значение `100` попадет в промис как результат его выполнения и будет в нем храниться.

### Пример с отложенным выполнением

```javascript
const delayedPromise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve(100);
  }, 1000);
});
```

### Пример с ошибкой

```javascript
const errorPromise = new Promise((resolve, reject) => {
  // В reject можно передавать любое значение (value)
  reject(new Error('Что-то пошло не так'));

  // Или даже просто число
  // reject(200);

  // Или undefined
  // reject();
});
```

**Примечание:** В `reject` можно передавать не только объекты Error, но и любые другие значения, включая `undefined`. Это не считается неправильным.

### Немедленное разрешение промиса

```javascript
const immediatePromise = new Promise((resolve) => {
  resolve('Готово!');
});

console.log(immediatePromise);
// Promise { <fulfilled>: 'Готово!' }
```

---

## Состояния Promise

Промис может находиться в одном из **трех состояний**:

### 1. **Pending (ожидание)**
```javascript
const pendingPromise = new Promise((resolve, reject) => {
  // Промис создан, но resolve/reject еще не вызваны
});

console.dir(pendingPromise);
// Promise { <pending> }
```

### 2. **Fulfilled (выполнен успешно)**
```javascript
const fulfilledPromise = new Promise((resolve) => {
  resolve('Успех!');
});

console.dir(fulfilledPromise);
// Promise { <fulfilled>: 'Успех!' }
```

### 3. **Rejected (отклонен с ошибкой)**
```javascript
const rejectedPromise = new Promise((resolve, reject) => {
  reject(new Error('Ошибка!'));
});

console.dir(rejectedPromise);
// Promise { <rejected>: Error: Ошибка! }
```

### Диаграмма переходов состояний

```
        ┌─────────────┐
        │   Pending   │ (начальное состояние)
        └──────┬──────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
┌──────────┐      ┌──────────┐
│Fulfilled │      │ Rejected │
│(успех)   │      │ (ошибка) │
└──────────┘      └──────────┘
```

**Важно:** После того как промис перешел в состояние `fulfilled` или `rejected`, он больше не может изменить свое состояние.

---

## Методы экземпляра Promise

### Метод `.then()`

Метод `.then()` принимает **два аргумента** — две функции:
- Первая функция обрабатывает успешное выполнение (resolve)
- Вторая функция обрабатывает ошибку (reject) — опциональна

```javascript
const promise = new Promise((resolve, reject) => {
  resolve(100);
});

promise.then(
  (value) => {
    console.log('Успех:', value); // Успех: 100
  },
  (error) => {
    console.error('Ошибка:', error);
  }
);
```

### Получение значения из промиса

```javascript
const promise = Promise.resolve('Данные');

// Внутреннее значение промиса недоступно напрямую
console.log(promise); // Promise { <fulfilled>: 'Данные' }

// Правильный способ получения значения
promise.then((data) => {
  console.log(data); // Данные
});

// Можно передать console.log напрямую
promise.then(console.log); // Данные
```

**Важный момент:** У промиса нет публичного свойства для доступа к значению. Консоль "магическим образом" добывает это значение для отладки, но программно получить значение можно только через `.then()`.

### Метод `.catch()`

Метод `.catch()` используется для обработки ошибок. Он эквивалентен `.then(null, errorHandler)`.

```javascript
const errorPromise = new Promise((resolve, reject) => {
  reject(new Error('Произошла ошибка'));
});

errorPromise.catch((error) => {
  console.error('Поймали ошибку:', error);
});

// Это то же самое, что:
errorPromise.then(null, (error) => {
  console.error('Поймали ошибку:', error);
});
```

### Метод `.finally()`

Метод `.finally()` выполняется **всегда**, независимо от того, выполнился промис успешно или с ошибкой.

```javascript
fetchData()
  .then((data) => {
    console.log('Данные получены:', data);
  })
  .catch((error) => {
    console.error('Ошибка:', error);
  })
  .finally(() => {
    console.log('Операция завершена');
    // Здесь можно освободить ресурсы:
    // - закрыть файловые дескрипторы
    // - освободить память
    // - скрыть индикатор загрузки
  });
```

### Два аргумента в .then() vs .catch()

```javascript
const promise = Promise.reject('Ошибка!');

// Вариант 1: Второй аргумент в then
promise.then(
  (value) => console.log('Успех:', value),
  (error) => console.error('Ошибка в then:', error) // Сюда попадет
);

// Вариант 2: Отдельный catch
promise
  .then((value) => console.log('Успех:', value))
  .catch((error) => console.error('Ошибка в catch:', error)); // И сюда попадет
```

**Важное отличие:** Если используются оба подхода одновременно, ошибка попадет во **второй аргумент `.then()`**, а `.catch()` ее не получит. Однако если ошибка возникнет **внутри первого аргумента** `.then()`, она попадет в `.catch()`.

```javascript
const promise = Promise.resolve('OK');

promise
  .then(
    (value) => {
      console.log(value);
      throw new Error('Ошибка внутри then!');
    },
    (error) => {
      console.log('Второй аргумент then:', error); // Не выполнится
    }
  )
  .catch((error) => {
    console.error('Catch перехватил:', error); // Сюда попадет ошибка из then
  });
```

---

## Цепочки Promise (Promise Chaining)

Одна из самых мощных возможностей промисов — **цепочные вызовы**. Каждый `.then()` возвращает новый промис.

### Базовая цепочка

```javascript
Promise.resolve(5)
  .then((value) => {
    console.log('Первый then:', value); // 5
    return value * 2;
  })
  .then((value) => {
    console.log('Второй then:', value); // 10
    return value * 2;
  })
  .then((value) => {
    console.log('Третий then:', value); // 20
  });
```

### Цепочка с обработкой ошибок

```javascript
Promise.resolve(100)
  .then((value) => {
    console.log('Шаг 1:', value);
    if (value > 50) {
      throw new Error('Значение слишком большое!');
    }
    return value * 2;
  })
  .then((value) => {
    console.log('Шаг 2:', value); // Не выполнится
  })
  .catch((error) => {
    console.error('Ошибка перехвачена:', error);
    return 0; // Можно вернуть значение для продолжения цепочки
  })
  .then((value) => {
    console.log('Продолжение после ошибки:', value); // 0
  });
```

### Цепочка .catch()

```javascript
Promise.reject('Первая ошибка')
  .catch((error) => {
    console.error('Первый catch:', error);
    throw new Error('Вторая ошибка'); // Генерируем новую ошибку
  })
  .catch((error) => {
    console.error('Второй catch:', error); // Ловит ошибку из первого catch
  });
```

### Несколько .then() после .catch()

```javascript
fetchData()
  .then(processData)
  .catch(handleError)
  .then(continueProcessing)  // Выполнится даже после catch
  .catch(handleFinalError);  // Ловит ошибки из continueProcessing
```

---

## Практическое применение: работа с файлами

### Преобразование callback-функции в промис

Промисы отлично подходят для работы с асинхронными операциями: таймерами, сетевыми запросами, взаимодействием с файловой системой, процессами и потоками.

#### Исходная функция с callback

```javascript
const fs = require('fs');

// Callback-стиль
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Ошибка чтения:', err);
    return;
  }
  console.log('Содержимое:', data);
});
```

#### Обертка в промис

```javascript
const fs = require('fs');

// Создаем функцию, возвращающую промис
const readFilePromise = (filename, encoding) => {
  return new Promise((resolve, reject) => {
    fs.readFile(filename, encoding, (err, data) => {
      if (err) {
        reject(err); // При ошибке вызываем reject
        return;
      }
      // При успехе преобразуем данные и вызываем resolve
      resolve(data.toString());
    });
  });
};

// Использование
readFilePromise('file.txt', 'utf8')
  .then((data) => {
    console.log('Файл прочитан:', data);
  })
  .catch((error) => {
    console.error('Ошибка чтения файла:', error);
  });
```

### Встроенная поддержка промисов в Node.js

```javascript
const fs = require('fs').promises;
// или
const fs = require('fs/promises');

// Теперь readFile возвращает промис
fs.readFile('file.txt', 'utf8')
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error(error);
  });
```

### Последовательное чтение нескольких файлов

```javascript
const fs = require('fs').promises;

// Читаем файлы последовательно
fs.readFile('file1.txt', 'utf8')
  .then((data1) => {
    console.log('Файл 1:', data1);
    // Возвращаем промис для чтения второго файла
    return fs.readFile('file2.txt', 'utf8');
  })
  .then((data2) => {
    console.log('Файл 2:', data2);
    return fs.readFile('file3.txt', 'utf8');
  })
  .then((data3) => {
    console.log('Файл 3:', data3);
  })
  .catch((error) => {
    console.error('Ошибка при чтении файлов:', error);
  });
```

**Проблема:** При таком подходе все ошибки попадут в один `.catch()`, и мы не сможем точно определить, на каком файле произошла ошибка.

### Обработка ошибок для каждого файла отдельно

```javascript
const fs = require('fs').promises;

fs.readFile('file1.txt', 'utf8')
  .then(
    (data) => {
      console.log('Файл 1:', data);
      return fs.readFile('file2.txt', 'utf8');
    },
    (err1) => {
      console.error('Ошибка чтения file1.txt:', err1);
      // Можно вернуть значение по умолчанию или продолжить цепочку
      throw err1; // Или пробросить ошибку дальше
    }
  )
  .then(
    (data) => {
      console.log('Файл 2:', data);
      return fs.readFile('file3.txt', 'utf8');
    },
    (err2) => {
      console.error('Ошибка чтения file2.txt:', err2);
    }
  )
  .then(
    (data) => {
      console.log('Файл 3:', data);
    },
    (err3) => {
      console.error('Ошибка чтения file3.txt:', err3);
    }
  );
```

**Важно понимать:** В цепочке промисов ошибка от конкретного промиса попадает в обработчик ошибок (второй аргумент `.then()` или `.catch()`), который идет непосредственно после этого промиса.

---

## Статические методы класса Promise

### Promise.resolve()

Создает немедленно разрешенный промис с указанным значением.

```javascript
const promise1 = Promise.resolve(42);

promise1.then((value) => {
  console.log(value); // 42
});

// Эквивалентно:
const promise2 = new Promise((resolve) => {
  resolve(42);
});
```

### Promise.reject()

Создает немедленно отклоненный промис с указанной ошибкой.

```javascript
const errorPromise = Promise.reject(new Error('Что-то пошло не так'));

errorPromise.catch((error) => {
  console.error(error); // Error: Что-то пошло не так
});

// Эквивалентно:
const errorPromise2 = new Promise((resolve, reject) => {
  reject(new Error('Что-то пошло не так'));
});
```

### Promise.all()

Ожидает завершения **всех** промисов в массиве. Возвращает массив результатов в том же порядке, что и входные промисы.

```javascript
const promise1 = fetch('http://localhost:1000/person');
const promise2 = fetch('http://localhost:1000/company');
const promise3 = fetch('http://localhost:1000/city');

Promise.all([promise1, promise2, promise3])
  .then((values) => {
    console.log('Все запросы выполнены:', values);
    // values - это массив [response1, response2, response3]
    // Порядок соответствует порядку промисов в массиве
    const [personData, companyData, cityData] = values;
  })
  .catch((error) => {
    console.error('Ошибка в одном из запросов:', error);
    // Если хоть один промис отклонен, весь Promise.all завершается с ошибкой
  });
```

**Важные особенности Promise.all():**
- Возвращает массив результатов в том же порядке, что и промисы
- Если **хотя бы один** промис отклонен, весь `Promise.all()` завершается с ошибкой
- Невозможно точно определить, какой именно промис вызвал ошибку
- Все промисы выполняются **параллельно**

### Promise.allSettled()

Ждет завершения **всех** промисов (успешных или нет) и возвращает массив результатов с информацией о статусе каждого.

```javascript
const promise1 = fetch('http://localhost:1000/person');
const promise2 = fetch('http://localhost:1000/company');
const promise3 = fetch('http://localhost:1000/city');

Promise.allSettled([promise1, promise2, promise3])
  .then((results) => {
    console.log('Все промисы завершены:', results);

    results.forEach((result, index) => {
      if (result.status === 'fulfilled') {
        console.log(`Промис ${index} успешен:`, result.value);
      } else {
        console.log(`Промис ${index} отклонен:`, result.reason);
      }
    });
  });

// Пример результата:
// [
//   { status: 'fulfilled', value: Response {...} },
//   { status: 'rejected', reason: Error: Network error },
//   { status: 'fulfilled', value: Response {...} }
// ]
```

**Структура результата:**
```javascript
// Для успешного промиса:
{ status: 'fulfilled', value: /* результат */ }

// Для отклоненного промиса:
{ status: 'rejected', reason: /* ошибка */ }
```

**Отличия от Promise.all():**
- Не завершается при первой ошибке
- Всегда выполняется успешно (попадает в `.then()`, а не в `.catch()`)
- Предоставляет подробную информацию о каждом промисе
- Идеален для ситуаций, когда нужны результаты всех операций, даже если некоторые не удались

### Promise.race()

Возвращает результат **первого завершенного** промиса (успешного или с ошибкой).

```javascript
const promise1 = fetch('http://api1.example.com/currency');
const promise2 = fetch('http://api2.example.com/currency');
const promise3 = fetch('http://api3.example.com/currency');

Promise.race([promise1, promise2, promise3])
  .then((result) => {
    console.log('Первый ответивший сервер:', result);
  })
  .catch((error) => {
    console.error('Первый промис завершился с ошибкой:', error);
  });
```

**Особенности Promise.race():**
- Возвращает результат первого **любого** завершенного промиса
- Если первый промис отклонен, весь `Promise.race()` отклоняется
- Полезно для таймаутов или выбора самого быстрого источника данных

#### Пример с таймаутом:

```javascript
const dataFetch = fetch('http://slow-api.example.com/data');
const timeout = new Promise((_, reject) => {
  setTimeout(() => reject(new Error('Timeout')), 5000);
});

Promise.race([dataFetch, timeout])
  .then((data) => console.log('Данные получены:', data))
  .catch((error) => console.error('Ошибка или таймаут:', error));
```

### Promise.any()

Ожидает **первого успешного** промиса. Игнорирует отклоненные промисы до тех пор, пока не найдет успешный.

```javascript
const promise1 = fetch('http://api1.example.com/currency'); // Может упасть
const promise2 = fetch('http://api2.example.com/currency'); // Может упасть
const promise3 = fetch('http://api3.example.com/currency'); // Может упасть

Promise.any([promise1, promise2, promise3])
  .then((result) => {
    console.log('Первый успешный результат:', result);
  })
  .catch((error) => {
    console.error('Все промисы отклонены:', error);
    // AggregateError: All promises were rejected
  });
```

**Ключевые отличия Promise.any() от Promise.race():**

| Характеристика | Promise.race() | Promise.any() |
|----------------|----------------|---------------|
| Что возвращает | Первый **любой** результат | Первый **успешный** результат |
| При первой ошибке | Отклоняется сразу | Продолжает ждать успешных |
| Когда отклоняется | При первом отклонении | Когда **все** промисы отклонены |
| Тип ошибки | Ошибка от промиса | AggregateError |

**Пример сравнения:**

```javascript
const fast_fail = Promise.reject('Быстрая ошибка');
const slow_success = new Promise((resolve) => {
  setTimeout(() => resolve('Медленный успех'), 1000);
});

// Promise.race() - вернет ошибку сразу
Promise.race([fast_fail, slow_success])
  .then((result) => console.log('Race успех:', result))
  .catch((error) => console.error('Race ошибка:', error));
  // Вывод: Race ошибка: Быстрая ошибка

// Promise.any() - дождется успешного результата
Promise.any([fast_fail, slow_success])
  .then((result) => console.log('Any успех:', result))
  .catch((error) => console.error('Any ошибка:', error));
  // Вывод (через 1 сек): Any успех: Медленный успех
```

**Практическое применение Promise.any():**
- Запросы к нескольким зеркалам API
- Получение данных из нескольких источников (курс валют, погода)
- Fallback стратегии при недоступности основного сервиса

---

## Обработка ошибок

### Цепочка обработки ошибок

```javascript
Promise.resolve('Начало')
  .then((value) => {
    console.log(value);
    throw new Error('Ошибка в первом then');
  })
  .catch((error) => {
    console.error('Первый catch:', error);
    return 'Восстановлено';
  })
  .then((value) => {
    console.log('После восстановления:', value);
    throw new Error('Вторая ошибка');
  })
  .catch((error) => {
    console.error('Второй catch:', error);
  });
```

### Проброс ошибок

```javascript
fetchUser(userId)
  .then((user) => {
    if (!user.active) {
      throw new Error('Пользователь неактивен');
    }
    return user;
  })
  .then((user) => fetchUserPosts(user.id))
  .catch((error) => {
    // Обработка и повторный проброс
    console.error('Ошибка:', error);
    throw error; // Проброс ошибки дальше по цепочке
  })
  .then((posts) => {
    // Не выполнится из-за ошибки выше
    console.log('Посты:', posts);
  })
  .catch((error) => {
    // Финальная обработка ошибки
    console.error('Финальная обработка:', error);
  });
```

### Поток управления в промисах

```javascript
// Визуализация потока управления
Promise.resolve('Старт')
  .then(onSuccess1, onError1)  // ← Успех идет сюда
  .then(onSuccess2, onError2)  // ← Результат onSuccess1 или onError1
  .catch(onError3)             // ← Ошибки из onSuccess2
  .finally(cleanup);           // ← Выполнится всегда
```

**Важные правила:**
1. Ошибка передается в ближайший обработчик ошибок (второй аргумент `.then()` или `.catch()`)
2. Если обработчика ошибок нет, ошибка передается дальше по цепочке
3. Возврат значения из обработчика ошибок "восстанавливает" промис (дальше идут успешные обработчики)
4. Повторный `throw` в обработчике ошибок передает ошибку дальше

---

## Best Practices и важные моменты

### 1. Всегда обрабатывайте ошибки

```javascript
// Плохо - необработанное отклонение промиса
fetchData().then(processData);

// Хорошо
fetchData()
  .then(processData)
  .catch(handleError);
```

### 2. Возвращайте промисы из .then()

```javascript
// Плохо - вложенные промисы
fetchUser().then((user) => {
  fetchPosts(user.id).then((posts) => {
    console.log(posts);
  });
});

// Хорошо - цепочка
fetchUser()
  .then((user) => fetchPosts(user.id))
  .then((posts) => console.log(posts))
  .catch(handleError);
```

### 3. Используйте Promise.all() для параллельных операций

```javascript
// Плохо - последовательное выполнение (медленно)
fetchUser()
  .then((user) => fetchPosts())
  .then((posts) => fetchComments());

// Хорошо - параллельное выполнение (быстро)
Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments()
]).then(([user, posts, comments]) => {
  console.log(user, posts, comments);
});
```

### 4. Не забывайте про .finally()

```javascript
showLoader();

fetchData()
  .then(processData)
  .catch(handleError)
  .finally(() => {
    hideLoader(); // Выполнится всегда
  });
```

### 5. Используйте правильный статический метод

```javascript
// Когда нужны ВСЕ результаты и критична любая ошибка
Promise.all([...]);

// Когда нужны все результаты, но ошибки не критичны
Promise.allSettled([...]);

// Когда нужен первый ЛЮБОЙ результат
Promise.race([...]);

// Когда нужен первый УСПЕШНЫЙ результат
Promise.any([...]);
```

### 6. Избегайте "заглушек" в reject

```javascript
// Плохо
new Promise((resolve, reject) => {
  if (error) reject(); // undefined
});

// Хорошо
new Promise((resolve, reject) => {
  if (error) reject(new Error('Описание ошибки'));
});
```

### 7. Понимайте разницу между вторым аргументом .then() и .catch()

```javascript
promise
  .then(
    onSuccess,
    onError1  // Не поймает ошибки из onSuccess
  )
  .catch(onError2); // Поймает ошибки из onSuccess

// Предпочтительнее использовать .catch() для лучшей читаемости
promise
  .then(onSuccess)
  .catch(onError);
```

### 8. Можно передавать функции напрямую

```javascript
// Эти варианты эквивалентны:
promise.then((x) => console.log(x));
promise.then(console.log);

// Для ошибок:
promise.catch((err) => console.error(err));
promise.catch(console.error);
```

---

## Дополнительные примеры и паттерны

### Паттерн: Retry (повторная попытка)

```javascript
function fetchWithRetry(url, retries = 3) {
  return fetch(url).catch((error) => {
    if (retries > 0) {
      console.log(`Осталось попыток: ${retries}`);
      return fetchWithRetry(url, retries - 1);
    }
    throw error;
  });
}

fetchWithRetry('http://unstable-api.com/data')
  .then((data) => console.log('Данные:', data))
  .catch((error) => console.error('Не удалось получить данные:', error));
```

### Паттерн: Последовательное выполнение из массива

```javascript
const files = ['file1.txt', 'file2.txt', 'file3.txt'];

// Последовательное чтение файлов
files.reduce((promise, file) => {
  return promise.then((results) => {
    return fs.readFile(file, 'utf8').then((data) => {
      results.push(data);
      return results;
    });
  });
}, Promise.resolve([])).then((allData) => {
  console.log('Все файлы прочитаны:', allData);
});
```

### Паттерн: Промисификация callback-функций

```javascript
// Универсальная функция промисификации
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}

// Использование
const fs = require('fs');
const readFilePromise = promisify(fs.readFile);

readFilePromise('file.txt', 'utf8')
  .then(console.log)
  .catch(console.error);
```

---

## Ключевые выводы

1. **Promise** — это объект для работы с асинхронными операциями
2. Промис имеет **три состояния**: pending, fulfilled, rejected
3. **Конструктор** принимает функцию с `resolve` и `reject`
4. **Методы экземпляра**: `.then()`, `.catch()`, `.finally()`
5. **Статические методы**: `.resolve()`, `.reject()`, `.all()`, `.allSettled()`, `.race()`, `.any()`
6. Промисы поддерживают **цепочки вызовов** для последовательных операций
7. Обработка ошибок идет по цепочке до ближайшего обработчика
8. Всегда используйте `.catch()` для обработки ошибок
9. Выбирайте правильный статический метод для параллельных операций

---

## Практические задания для закрепления

### Задание 1: Базовая работа с промисами
Создайте промис, который разрешается через 2 секунды со значением "Готово" и обработайте результат.

### Задание 2: Цепочка промисов
Создайте цепочку из трех промисов, где каждый умножает предыдущее значение на 2, начиная с 5.

### Задание 3: Обработка ошибок
Создайте промис, который случайным образом разрешается или отклоняется, и правильно обработайте оба случая.

### Задание 4: Promise.all()
Создайте три промиса с разным временем выполнения и дождитесь их всех с помощью `Promise.all()`.

### Задание 5: Практическая задача
Промисифицируйте функцию `setTimeout` и используйте ее для создания задержек в цепочке операций.

---

## Дополнительные ресурсы

- [MDN: Promise](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [Promises/A+ спецификация](https://promisesaplus.com/)
- [Node.js: Промисификация](https://nodejs.org/api/util.html#utilpromisifyoriginal)
- [JavaScript.info: Промисы](https://learn.javascript.ru/promise)

---

**Конец конспекта**

*Конспект создан на основе лекции "Promise contract in JavaScript — Async 2024"*
