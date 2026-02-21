# Callback контракт в JavaScript: Подробный конспект

## Обзор

Callbacks (колбеки) представляют собой первый и наиболее фундаментальный контракт асинхронности в JavaScript и Node.js. Эта лекция детально рассматривает эволюцию от простых синхронных функций к асинхронным колбекам, объясняет контракт **callback-last-error-first**, демонстрирует решение проблемы callback hell и показывает лучшие практики работы с колбеками.

**Ключевые темы лекции:**
- Эволюция от синхронных функций к callback-паттерну
- Контракт callback-last-error-first (CLF) и его значение
- Различие между синхронными и асинхронными колбеками
- Именованные колбеки и их преимущества перед анонимными
- Эффективное решение проблемы callback hell через декомпозицию
- Систематическая обработка ошибок в асинхронном коде
- Восстановление естественного логического порядка операций

---

## 1. Что такое колбек? Эволюция от синхронности к асинхронности

### Определение

**Callback (колбек)** — это функция, которая передается в качестве аргумента в другую функцию и вызывается этой функцией для передачи результата операции или сигнализации о завершении выполнения.

Колбеки являются примером паттерна **Inversion of Control** (инверсия управления): вместо того чтобы возвращать результат через `return`, функция вызывает переданный ей колбек с результатом.

### От синхронной функции к асинхронному колбеку

#### Шаг 1: Обычная синхронная функция

```javascript
// Простая синхронная функция сложения двух чисел
function sum(a, b) {
  return a + b;
}

// Использование
const result = sum(5, 3);
console.log('Результат:', result); // Результат: 8

// Выполнение происходит синхронно:
console.log('До вызова sum');
const res = sum(10, 20);
console.log('После вызова sum, результат:', res);
console.log('Продолжаем выполнение');

// Вывод (последовательный):
// До вызова sum
// После вызова sum, результат: 30
// Продолжаем выполнение
```

**Характеристики синхронной функции:**
- Принимает параметры (в данном случае `a` и `b`)
- Выполняет вычисления
- Возвращает результат через оператор `return`
- Блокирует выполнение программы до получения результата
- Результат доступен немедленно после вызова

**Проблемы синхронного подхода:**
- Блокировка выполнения при долгих операциях (чтение файлов, сетевые запросы)
- Невозможность выполнять другие задачи во время ожидания
- Неэффективное использование ресурсов

#### Шаг 2: Простой колбек без обработки ошибок

```javascript
// Функция с колбеком - добавляем третий параметр
function sum(a, b, callback) {
  // Вместо return вызываем колбек с результатом
  const result = a + b;
  callback(result);
}

// Использование с анонимной стрелочной функцией
sum(5, 3, (result) => {
  console.log('Результат через колбек:', result);
  // Результат через колбек: 8
});

// Использование с именованной функцией
function handleSum(result) {
  console.log('Сумма равна:', result);
}

sum(10, 20, handleSum); // Сумма равна: 30

// Демонстрация асинхронного поведения
console.log('Начало');

sum(7, 13, (result) => {
  console.log('Результат:', result);
});

console.log('Конец');

// Вывод (пока синхронный):
// Начало
// Результат: 20
// Конец
```

**Что изменилось:**
- Добавлен третий аргумент — функция `callback`
- Вместо `return` результат передается как аргумент в колбек
- Вызывающий код получает результат через вызов переданной функции
- Подготовка к асинхронному выполнению (хотя пока код синхронный)

**Важное замечание:** Сам факт использования колбека не делает функцию асинхронной. Для асинхронности нужны специальные механизмы (таймеры, I/O операции, промисы и т.д.).

#### Шаг 3: Колбек с контрактом error-first

```javascript
// Функция следует контракту callback-last-error-first
function sum(a, b, callback) {
  // Первый аргумент колбека - ошибка (null, если её нет)
  // Второй аргумент - результат вычисления
  callback(null, a + b);
}

// Использование с явной обработкой ошибок
sum(5, 3, (error, result) => {
  if (error) {
    console.error('Произошла ошибка:', error.message);
    return; // Важно! Прекращаем выполнение при ошибке
  }
  console.log('Результат:', result); // Результат: 8
});

// Пример с множественными операциями
function calculate(operation, a, b, callback) {
  let result;

  switch(operation) {
    case 'add':
      result = a + b;
      break;
    case 'subtract':
      result = a - b;
      break;
    case 'multiply':
      result = a * b;
      break;
    case 'divide':
      if (b === 0) {
        callback(new Error('Division by zero'));
        return;
      }
      result = a / b;
      break;
    default:
      callback(new Error(`Unknown operation: ${operation}`));
      return;
  }

  callback(null, result);
}

// Использование
calculate('add', 10, 5, (err, res) => {
  if (err) return console.error(err.message);
  console.log('10 + 5 =', res); // 10 + 5 = 15
});

calculate('divide', 10, 0, (err, res) => {
  if (err) return console.error(err.message); // Division by zero
  console.log(res);
});
```

**Ключевые моменты контракта error-first:**
- Первый аргумент колбека **всегда** зарезервирован для ошибки
- Если операция успешна, первый аргумент равен `null` или `undefined`
- Результаты операции передаются начиная со второго аргумента
- Обработчик **обязан** проверить наличие ошибки перед использованием результата

#### Шаг 4: Передача ошибки в колбек

```javascript
// Функция с валидацией и генерацией ошибок
function divide(a, b, callback) {
  // Валидация входных данных
  if (typeof a !== 'number' || typeof b !== 'number') {
    callback(new Error('Аргументы должны быть числами'));
    return;
  }

  if (b === 0) {
    // Передаем ошибку как первый аргумент
    callback(new Error('Деление на ноль невозможно'));
    return;
  }

  // Успешный результат - первый аргумент null
  callback(null, a / b);
}

// Примеры использования
divide(10, 2, (error, result) => {
  if (error) {
    console.error('Ошибка:', error.message);
    return;
  }
  console.log('Результат деления:', result);
  // Результат деления: 5
});

divide(10, 0, (error, result) => {
  if (error) {
    console.error('Ошибка:', error.message);
    // Ошибка: Деление на ноль невозможно
    return;
  }
  console.log('Результат деления:', result);
});

divide('10', 2, (error, result) => {
  if (error) {
    console.error('Ошибка:', error.message);
    // Ошибка: Аргументы должны быть числами
    return;
  }
  console.log('Результат деления:', result);
});

// Обработка с разными стратегиями
function divideWithLogging(a, b, callback) {
  divide(a, b, (error, result) => {
    if (error) {
      // Логируем ошибку и пробрасываем дальше
      console.error(`[LOG] Ошибка при делении ${a} на ${b}:`, error.message);
      callback(error);
      return;
    }

    // Логируем успешный результат
    console.log(`[LOG] ${a} / ${b} = ${result}`);
    callback(null, result);
  });
}
```

**Типы ошибок, которые стоит обрабатывать:**
1. **Ошибки валидации** — некорректные входные данные
2. **Логические ошибки** — невозможность выполнения операции (деление на ноль)
3. **Системные ошибки** — ошибки файловой системы, сети и т.д.
4. **Ошибки парсинга** — некорректный формат данных

---

## 2. Контракт Callback-last-error-first

### Полное определение контракта

**Callback-last-error-first (CLF)** — это соглашение (convention) о том, как структурировать асинхронные функции и колбеки в экосистеме Node.js и JavaScript.

**Название контракта состоит из двух частей:**

1. **Callback-last** (колбек последним):
   - Функция с асинхронным поведением принимает колбек **последним аргументом**
   - Все параметры данных идут перед колбеком
   - Это делает API предсказуемым и единообразным

2. **Error-first** (ошибка первой):
   - Колбек получает ошибку **первым аргументом**
   - Если ошибки нет, передается `null` или `undefined`
   - Данные результата передаются начиная со второго аргумента

### Формальная структура контракта

```javascript
// Общая форма асинхронной функции с CLF контрактом
function asyncOperation(param1, param2, ..., paramN, callback) {
  // Выполнение асинхронной операции

  // При возникновении ошибки:
  if (errorCondition) {
    callback(new Error('Описание ошибки'));
    return;
  }

  // При успешном выполнении:
  callback(null, result1, result2, ..., resultN);
}

// Общая форма callback-функции
function callback(error, data1, data2, ..., dataN) {
  // Обязательная проверка ошибки
  if (error) {
    // Обработка ошибки: логирование, пробрасывание, восстановление
    console.error('Error:', error.message);
    return; // Важно! Прекращаем выполнение
  }

  // Работа с результатами
  console.log('Data:', data1, data2, ..., dataN);
}
```

### Примеры сигнатур функций с CLF

```javascript
// Чтение файла (Node.js fs module)
fs.readFile(path, encoding, callback);
// callback(error, data)

// Запись файла
fs.writeFile(path, data, encoding, callback);
// callback(error)

// Запрос к базе данных
database.query(sql, params, callback);
// callback(error, rows)

// HTTP запрос
http.get(url, options, callback);
// callback(error, response)

// Множественные результаты
someAsyncOperation(param1, param2, callback);
// callback(error, result1, result2, result3)
```

### Почему контракт CLF важен?

**Преимущества следования контракту:**

1. **Единообразие API**
   - Все асинхронные функции следуют одному паттерну
   - Не нужно каждый раз читать документацию
   - Снижается когнитивная нагрузка

2. **Обязательная обработка ошибок**
   - Первый аргумент напоминает о необходимости проверки ошибок
   - Ошибки нельзя "случайно" проигнорировать
   - Явная обработка ошибок на каждом уровне

3. **Предсказуемость кода**
   - Легко читать чужой код
   - Легко писать код, который другие поймут
   - Упрощается code review

4. **Совместимость экосистемы**
   - Стандарт для всего Node.js ecosystem
   - Все библиотеки и модули следуют одному контракту
   - Легко интегрировать разные модули

5. **Возможность создания утилит**
   - Можно писать универсальные обработчики
   - Легко промисифицировать функции
   - Упрощается тестирование

### Пример универсального обработчика

```javascript
// Универсальная функция для логирования операций
function withLogging(operationName, asyncFunction) {
  return function(...args) {
    const callback = args[args.length - 1];

    // Создаем обертку над колбеком
    const wrappedCallback = (error, ...results) => {
      if (error) {
        console.error(`[${operationName}] ERROR:`, error.message);
      } else {
        console.log(`[${operationName}] SUCCESS:`, results);
      }

      // Вызываем оригинальный колбек
      callback(error, ...results);
    };

    // Заменяем последний аргумент (колбек) на обертку
    args[args.length - 1] = wrappedCallback;

    console.log(`[${operationName}] START`);
    asyncFunction(...args);
  };
}

// Использование
const fs = require('fs');

const readFileWithLogging = withLogging('ReadFile', fs.readFile);

readFileWithLogging('./config.json', 'utf8', (error, data) => {
  if (error) return;
  console.log('File content:', data);
});

// Вывод:
// [ReadFile] START
// [ReadFile] SUCCESS: ['{"port": 3000}']
// File content: {"port": 3000}
```

### Нарушение контракта и его последствия

```javascript
// ПЛОХО: Колбек не последний аргумент
function badFunction(callback, param1, param2) {
  // Непривычный порядок - легко ошибиться
  callback(null, param1 + param2);
}

// ПЛОХО: Ошибка не первый аргумент
function badFunction2(param, callback) {
  // Нарушает контракт error-first
  callback(result, error);
}

// ХОРОШО: Следование контракту
function goodFunction(param1, param2, callback) {
  if (someError) {
    callback(new Error('Description'));
    return;
  }
  callback(null, result);
}
```

---

## 3. Синхронные и асинхронные колбеки

### Важное различие

**Критическое понимание:** Сам по себе паттерн колбеков **не гарантирует асинхронность**. Колбеки — это просто способ передачи функции в качестве аргумента. Колбек может быть вызван как синхронно, так и асинхронно.

**Асинхронность зависит от:**
- Использования Event Loop (setTimeout, setInterval)
- I/O операций (fs, network, database)
- Промисов и async/await
- Web APIs (в браузере)
- Worker threads, child processes

### Синхронные колбеки

Синхронные колбеки выполняются немедленно, в рамках текущего выполнения, до того как управление вернется к следующей строке кода.

#### Пример 1: Методы массивов

```javascript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

console.log('Начало');

// Array.prototype.filter - синхронный метод
const evenNumbers = numbers.filter((num) => {
  console.log('  Проверяем число:', num);
  return num % 2 === 0;
});

console.log('Четные числа:', evenNumbers);
console.log('---');

// Array.prototype.map - синхронный метод
const doubled = numbers.map((num) => {
  console.log('  Удваиваем:', num);
  return num * 2;
});

console.log('Удвоенные числа:', doubled);
console.log('---');

// Array.prototype.reduce - синхронный метод
const sum = numbers.reduce((acc, num) => {
  console.log(`  Аккумулятор: ${acc}, текущее: ${num}`);
  return acc + num;
}, 0);

console.log('Сумма:', sum);
console.log('---');

// Array.prototype.find - синхронный метод
const firstEven = numbers.find((num) => {
  console.log('  Ищем четное, проверяем:', num);
  return num % 2 === 0;
});

console.log('Первое четное:', firstEven);
console.log('Конец');

// Вывод будет строго последовательным
```

**Характеристики синхронных колбеков:**
- Выполняются немедленно в момент вызова
- Блокируют дальнейшее выполнение до завершения
- Не требуют Event Loop
- Используются для:
  - Итерации по коллекциям
  - Фильтрации и преобразования данных
  - Сортировки (Array.prototype.sort)
  - Валидации данных

#### Пример 2: Пользовательские синхронные колбеки

```javascript
// Функция принимает массив и колбек для обработки каждого элемента
function forEachSync(array, callback) {
  for (let i = 0; i < array.length; i++) {
    // Колбек вызывается синхронно
    callback(array[i], i, array);
  }
}

// Использование
console.log('До forEach');

forEachSync([1, 2, 3], (item, index) => {
  console.log(`  Элемент ${index}: ${item}`);
});

console.log('После forEach');

// Вывод (последовательный):
// До forEach
//   Элемент 0: 1
//   Элемент 1: 2
//   Элемент 2: 3
// После forEach
```

#### Пример 3: Обработка конфигурации

```javascript
// Синхронная обработка конфигурации
function processConfig(config, validators) {
  const errors = [];

  // Проходим по всем валидаторам синхронно
  validators.forEach((validator) => {
    const error = validator(config);
    if (error) {
      errors.push(error);
    }
  });

  return errors;
}

// Валидаторы
const validators = [
  (config) => config.port ? null : 'Port is required',
  (config) => config.host ? null : 'Host is required',
  (config) => typeof config.port === 'number' ? null : 'Port must be a number',
];

const config = { port: 3000, host: 'localhost' };

console.log('Начало валидации');
const errors = processConfig(config, validators);
console.log('Конец валидации');
console.log('Ошибки:', errors); // []
```

### Асинхронные колбеки

Асинхронные колбеки выполняются в будущем, после завершения текущего стека вызовов. Они не блокируют выполнение программы.

#### Пример 1: Таймеры

```javascript
// Асинхронные колбеки с таймерами
console.log('1. Начало программы');

setTimeout(() => {
  // Этот колбек выполнится асинхронно через 0 мс
  console.log('2. Таймер 0 мс');
}, 0);

setTimeout(() => {
  console.log('4. Таймер 100 мс');
}, 100);

console.log('3. Синхронный код');

// Вывод:
// 1. Начало программы
// 3. Синхронный код
// 2. Таймер 0 мс
// 4. Таймер 100 мс

// Обратите внимание: даже с задержкой 0 мс,
// колбек выполняется после синхронного кода!
```

#### Пример 2: Обработка массива с задержкой

```javascript
// Асинхронная итерация по массиву
const data = [
  { id: 1, name: 'Alice', role: 'Admin' },
  { id: 2, name: 'Bob', role: 'User' },
  { id: 3, name: 'Charlie', role: 'Moderator' }
];

let index = 0;

console.log('Запуск асинхронной обработки');

// Устанавливаем интервал на 10 миллисекунд
const timerId = setInterval(() => {
  // Этот колбек выполняется асинхронно каждые 10 мс
  const item = data[index];
  console.log(`  [${index + 1}] Обработка:`, item.name, '-', item.role);

  index++;

  // Когда дошли до конца массива - останавливаем таймер
  if (index >= data.length) {
    clearInterval(timerId);
    console.log('Обработка завершена');
  }
}, 10);

console.log('Таймер запущен, продолжаем выполнение...');
console.log('Эта строка выполнится ДО обработки элементов');

// Вывод:
// Запуск асинхронной обработки
// Таймер запущен, продолжаем выполнение...
// Эта строка выполнится ДО обработки элементов
//   [1] Обработка: Alice - Admin
//   [2] Обработка: Bob - User
//   [3] Обработка: Charlie - Moderator
// Обработка завершена
```

#### Пример 3: Имитация асинхронной операции

```javascript
// Функция имитирует асинхронную операцию (например, API запрос)
function fetchUserData(userId, callback) {
  console.log(`  [API] Отправка запроса для пользователя ${userId}`);

  // Имитируем задержку сети с помощью setTimeout
  setTimeout(() => {
    // Имитируем получение данных
    const userData = {
      id: userId,
      name: `User${userId}`,
      email: `user${userId}@example.com`,
      createdAt: new Date().toISOString()
    };

    console.log(`  [API] Получены данные для пользователя ${userId}`);

    // Вызываем колбек с результатом
    callback(null, userData);
  }, Math.random() * 1000); // Случайная задержка 0-1000 мс
}

// Использование
console.log('Начало');

fetchUserData(1, (error, user) => {
  if (error) {
    console.error('Ошибка:', error);
    return;
  }
  console.log('Получен пользователь:', user.name);
});

fetchUserData(2, (error, user) => {
  if (error) {
    console.error('Ошибка:', error);
    return;
  }
  console.log('Получен пользователь:', user.name);
});

console.log('Конец синхронного кода');

// Вывод (порядок может варьироваться из-за случайной задержки):
// Начало
//   [API] Отправка запроса для пользователя 1
//   [API] Отправка запроса для пользователя 2
// Конец синхронного кода
//   [API] Получены данные для пользователя 2
// Получен пользователь: User2
//   [API] Получены данные для пользователя 1
// Получен пользователь: User1
```

**Характеристики асинхронных колбеков:**
- Выполняются в будущем, не блокируя текущий код
- Используют Event Loop для планирования выполнения
- Позволяют программе продолжать работу во время ожидания
- Применяются для:
  - Операций ввода-вывода (файлы, сеть, БД)
  - Таймеров и интервалов
  - Событий пользователя (клики, нажатия клавиш)
  - Асинхронных вычислений

### Опасность смешивания синхронного и асинхронного поведения

```javascript
// ПЛОХО: Непредсказуемое поведение
function maybeAsync(useCache, callback) {
  if (useCache) {
    // Синхронный вызов колбека
    callback(null, cachedData);
  } else {
    // Асинхронный вызов колбека
    fetchData((error, data) => {
      callback(error, data);
    });
  }
}

// ХОРОШО: Всегда асинхронно
function alwaysAsync(useCache, callback) {
  if (useCache) {
    // Делаем синхронный вызов асинхронным
    process.nextTick(() => {
      callback(null, cachedData);
    });
  } else {
    fetchData((error, data) => {
      callback(error, data);
    });
  }
}
```

**Принцип:** Если функция может быть асинхронной, она должна быть **всегда** асинхронной, даже когда результат доступен немедленно.

---

## 4. Асинхронные операции ввода-вывода в Node.js

### Файловая система (fs module)

Все операции ввода-вывода в Node.js используют контракт callback-last-error-first. Модуль `fs` предоставляет как синхронные, так и асинхронные версии методов.

#### Чтение файла

```javascript
const fs = require('fs');
const path = require('path');

// Асинхронное чтение файла
function readConfigFile(callback) {
  const configPath = path.join(__dirname, 'config.json');

  // fs.readFile следует контракту CLF
  fs.readFile(configPath, 'utf8', (error, data) => {
    if (error) {
      // Обработка различных типов ошибок
      if (error.code === 'ENOENT') {
        console.error('Файл не найден:', configPath);
      } else if (error.code === 'EACCES') {
        console.error('Нет прав доступа к файлу:', configPath);
      } else {
        console.error('Ошибка чтения файла:', error.message);
      }

      callback(error);
      return;
    }

    // Парсинг JSON
    try {
      const config = JSON.parse(data);
      console.log('Конфиг успешно прочитан');
      callback(null, config);
    } catch (parseError) {
      console.error('Ошибка парсинга JSON:', parseError.message);
      callback(parseError);
    }
  });
}

// Использование
console.log('Запрос на чтение конфига...');

readConfigFile((error, config) => {
  if (error) {
    console.error('Не удалось загрузить конфиг');
    process.exit(1);
    return;
  }

  console.log('Конфигурация:', config);
  console.log('Порт:', config.port);
  console.log('Хост:', config.host);
});

console.log('Продолжаем выполнение (не ждем чтения файла)');

// Вывод:
// Запрос на чтение конфига...
// Продолжаем выполнение (не ждем чтения файла)
// Конфиг успешно прочитан
// Конфигурация: { port: 3000, host: 'localhost' }
// Порт: 3000
// Хост: localhost
```

#### Запись в файл

```javascript
const fs = require('fs');
const path = require('path');

// Асинхронная запись данных в файл
function saveData(filename, data, callback) {
  const filePath = path.join(__dirname, filename);

  // Преобразуем объект в JSON строку
  let jsonString;
  try {
    jsonString = JSON.stringify(data, null, 2); // с форматированием
  } catch (stringifyError) {
    callback(stringifyError);
    return;
  }

  // Записываем в файл
  fs.writeFile(filePath, jsonString, 'utf8', (error) => {
    if (error) {
      console.error('Ошибка записи файла:', error.message);
      callback(error);
      return;
    }

    console.log('Данные успешно записаны в', filename);
    callback(null);
  });
}

// Использование
const reportData = {
  timestamp: Date.now(),
  message: 'Hello, World!',
  status: 'success',
  items: [1, 2, 3, 4, 5]
};

saveData('report.json', reportData, (error) => {
  if (error) {
    console.error('Не удалось сохранить отчет');
    return;
  }

  console.log('Отчет сохранен');
});
```

#### Проверка существования и создание директории

```javascript
const fs = require('fs');
const path = require('path');

// Создание директории, если она не существует
function ensureDirectoryExists(dirPath, callback) {
  // Проверяем существование
  fs.access(dirPath, fs.constants.F_OK, (error) => {
    if (error) {
      // Директория не существует - создаем
      console.log('Директория не существует, создаем:', dirPath);

      fs.mkdir(dirPath, { recursive: true }, (mkdirError) => {
        if (mkdirError) {
          console.error('Ошибка создания директории:', mkdirError.message);
          callback(mkdirError);
          return;
        }

        console.log('Директория создана');
        callback(null);
      });
    } else {
      // Директория существует
      console.log('Директория уже существует:', dirPath);
      callback(null);
    }
  });
}

// Использование
const logsDir = path.join(__dirname, 'logs');

ensureDirectoryExists(logsDir, (error) => {
  if (error) {
    console.error('Не удалось подготовить директорию для логов');
    return;
  }

  // Теперь можем безопасно записывать файлы в директорию
  const logFile = path.join(logsDir, 'app.log');
  fs.writeFile(logFile, 'Application started\n', 'utf8', (writeError) => {
    if (writeError) {
      console.error('Ошибка записи лога:', writeError.message);
      return;
    }
    console.log('Лог записан');
  });
});
```

#### Чтение содержимого директории

```javascript
const fs = require('fs');
const path = require('path');

// Получение списка файлов в директории
function listFiles(dirPath, callback) {
  fs.readdir(dirPath, (error, files) => {
    if (error) {
      console.error('Ошибка чтения директории:', error.message);
      callback(error);
      return;
    }

    console.log(`Найдено файлов: ${files.length}`);

    // Фильтруем только файлы (не директории)
    const fileStats = [];
    let pending = files.length;

    if (pending === 0) {
      callback(null, []);
      return;
    }

    files.forEach((file) => {
      const filePath = path.join(dirPath, file);

      fs.stat(filePath, (statError, stats) => {
        if (!statError && stats.isFile()) {
          fileStats.push({
            name: file,
            size: stats.size,
            modified: stats.mtime
          });
        }

        pending--;

        if (pending === 0) {
          callback(null, fileStats);
        }
      });
    });
  });
}

// Использование
listFiles(__dirname, (error, files) => {
  if (error) {
    console.error('Не удалось получить список файлов');
    return;
  }

  console.log('Файлы в директории:');
  files.forEach((file) => {
    console.log(`  ${file.name} - ${file.size} байт`);
  });
});
```

### Важные моменты работы с fs

**Параметры колбека в разных операциях:**

- `fs.readFile(path, encoding, callback)` → `callback(error, data)`
- `fs.writeFile(path, data, encoding, callback)` → `callback(error)`
- `fs.readdir(path, callback)` → `callback(error, files)`
- `fs.stat(path, callback)` → `callback(error, stats)`
- `fs.mkdir(path, options, callback)` → `callback(error)`
- `fs.unlink(path, callback)` → `callback(error)`

**Общие паттерны:**
- Колбек всегда последний аргумент
- Первый параметр колбека — ошибка
- Для операций чтения — второй параметр содержит данные
- Для операций записи/удаления — колбек получает только ошибку

---

## 5. Именованные колбеки: лучшая практика

### Проблема анонимных колбеков

Анонимные стрелочные функции удобны для простых случаев, но создают проблемы:

```javascript
const fs = require('fs');

// ПРОБЛЕМА: Множественные анонимные колбеки
fs.readFile('./config.json', 'utf8', (error, data) => {
  if (error) {
    console.error(error);
    return;
  }

  const config = JSON.parse(data);

  fs.readFile(config.dataFile, 'utf8', (error2, data2) => {
    if (error2) {
      console.error(error2);
      return;
    }

    processData(data2, (error3, result) => {
      if (error3) {
        console.error(error3);
        return;
      }

      console.log(result);
    });
  });
});
```

**Проблемы этого кода:**
1. **Плохие stack traces** — при ошибке трудно понять, где именно проблема
2. **Сложность отладки** — невозможно поставить breakpoint на конкретную функцию
3. **Невозможность переиспользования** — анонимные функции нельзя вызвать повторно
4. **Сложность тестирования** — нельзя протестировать отдельные части
5. **Плохая читаемость** — код превращается в "пирамиду"

### Решение: именованные функции

**Преимущества именованных колбеков:**
- Понятные stack traces с названиями функций
- Каждую функцию можно экспортировать и переиспользовать
- Легко писать unit-тесты для каждой функции
- Код самодокументируется через имена функций
- Упрощается отладка и профилирование

#### Пример 1: Базовое использование

```javascript
const fs = require('fs');

// Именованная функция-обработчик
function handleFileRead(error, data) {
  if (error) {
    console.error('Ошибка чтения файла:', error.message);
    return;
  }
  console.log('Содержимое файла:', data);
}

// Использование
fs.readFile('./data.txt', 'utf8', handleFileRead);
```

**Stack trace с именованной функцией:**
```
Error: ENOENT: no such file or directory
    at Object.handleFileRead (example.js:5:15)
    at FSReqCallback.readFileCallback (fs.js:280:3)
```

**Stack trace с анонимной функцией:**
```
Error: ENOENT: no such file or directory
    at Object.<anonymous> (example.js:5:15)
    at FSReqCallback.readFileCallback (fs.js:280:3)
```

### Паттерн с замыканием

Часто нужно передать в колбек дополнительные параметры. Для этого используют замыкания.

#### Пример: Замыкание vs bind

```javascript
const fs = require('fs');

// Функция для вывода содержимого файла с префиксом
function printFileContent(prefix, error, data) {
  if (error) {
    console.error(`[${prefix}] Ошибка:`, error.message);
    return;
  }
  console.log(`[${prefix}] Содержимое:`, data);
}

// ВАРИАНТ 1: Использование bind (старый подход)
// Связываем первый аргумент (prefix), оставляя два других свободными
fs.readFile('./file1.txt', 'utf8', printFileContent.bind(null, 'FILE1'));
fs.readFile('./file2.txt', 'utf8', printFileContent.bind(null, 'FILE2'));

// Проблемы bind:
// - Нужно помнить про context (первый аргумент null)
// - Неочевидный порядок аргументов
// - Сложнее читается

// ВАРИАНТ 2: Использование замыкания (современный подход)
function printFileContent(prefix) {
  // Возвращаем функцию с нужной сигнатурой
  return function(error, data) {
    if (error) {
      console.error(`[${prefix}] Ошибка:`, error.message);
      return;
    }
    console.log(`[${prefix}] Содержимое:`, data);
  };
}

// Вызов функции возвращает готовый колбек
fs.readFile('./file1.txt', 'utf8', printFileContent('FILE1'));
fs.readFile('./file2.txt', 'utf8', printFileContent('FILE2'));
fs.readFile('./file3.txt', 'utf8', printFileContent('FILE3'));

// Преимущества замыкания:
// - Явное разделение параметров конфигурации и runtime параметров
// - Чистый и читаемый синтаксис
// - Не нужно думать про контекст
// - Более функциональный подход
```

#### Пример: Создание обработчиков с контекстом

```javascript
const fs = require('fs');

// Фабрика обработчиков с логированием
function createFileHandler(fileName, onSuccess) {
  return function(error, data) {
    if (error) {
      console.error(`[${fileName}] Ошибка чтения:`, error.code);
      // Можно добавить fallback логику
      if (error.code === 'ENOENT') {
        console.log(`[${fileName}] Используем значения по умолчанию`);
        onSuccess('default content');
      }
      return;
    }

    console.log(`[${fileName}] Файл прочитан, размер: ${data.length} байт`);
    onSuccess(data);
  };
}

// Использование
function processConfig(data) {
  console.log('Обработка конфига:', data);
}

function processData(data) {
  console.log('Обработка данных:', data);
}

fs.readFile('./config.json', 'utf8', createFileHandler('config.json', processConfig));
fs.readFile('./data.json', 'utf8', createFileHandler('data.json', processData));
```

#### Пример: Параметризация обработки ошибок

```javascript
const fs = require('fs');

// Создаем обработчик с разными стратегиями обработки ошибок
function createReader(options = {}) {
  const {
    fileName = 'unknown',
    onError = 'log',    // 'log', 'throw', 'ignore'
    defaultValue = null,
    encoding = 'utf8'
  } = options;

  return function(error, data) {
    if (error) {
      switch(onError) {
        case 'throw':
          throw error;
        case 'ignore':
          break;
        case 'log':
        default:
          console.error(`[${fileName}] Error:`, error.message);
      }

      // Используем значение по умолчанию при ошибке
      if (defaultValue !== null) {
        console.log(`[${fileName}] Using default value`);
        return processData(defaultValue);
      }
      return;
    }

    console.log(`[${fileName}] Success`);
    processData(data);
  };
}

// Использование с разными стратегиями
fs.readFile('./critical.json', 'utf8', createReader({
  fileName: 'critical.json',
  onError: 'throw'  // Критичный файл - прерываем при ошибке
}));

fs.readFile('./optional.json', 'utf8', createReader({
  fileName: 'optional.json',
  onError: 'ignore',  // Опциональный файл - игнорируем ошибку
  defaultValue: '{}'
}));

fs.readFile('./config.json', 'utf8', createReader({
  fileName: 'config.json',
  onError: 'log',  // Обычный файл - логируем ошибку
  defaultValue: '{"port": 3000}'
}));
```

---

## 6. Адаптация существующих API к контракту CLF

### Проблема: setTimeout и setInterval

Функции таймеров в JavaScript были разработаны до стандартизации контракта CLF и имеют другую сигнатуру:

```javascript
// setTimeout принимает колбек ПЕРВЫМ аргументом (не соответствует CLF)
setTimeout(callback, delay, ...args);

// setInterval также принимает колбек ПЕРВЫМ
setInterval(callback, interval, ...args);

// Это несовместимо с паттерном callback-last
```

### Решение: Адаптеры

Создаем обертки, которые меняют порядок аргументов:

```javascript
// Адаптер для setTimeout - приводим к CLF
function delay(timeout, callback) {
  setTimeout(callback, timeout);
}

// Адаптер для setInterval - приводим к CLF
function repeat(interval, callback) {
  return setInterval(callback, interval);
}

// Теперь можно использовать с CLF контрактом
delay(1000, () => {
  console.log('Прошла 1 секунда');
});

const timerId = repeat(500, () => {
  console.log('Тик');
});

// Остановка интервала через 3 секунды
delay(3000, () => {
  clearInterval(timerId);
  console.log('Интервал остановлен');
});
```

#### Расширенный адаптер с error-first

```javascript
// Адаптер с поддержкой error-first (для совместимости)
function delayWithCallback(timeout, callback) {
  // Валидация
  if (typeof timeout !== 'number' || timeout < 0) {
    // Асинхронно вызываем с ошибкой
    process.nextTick(() => {
      callback(new Error('Timeout must be a positive number'));
    });
    return;
  }

  if (typeof callback !== 'function') {
    throw new TypeError('Callback must be a function');
  }

  // Запускаем таймер
  const timerId = setTimeout(() => {
    // Вызываем колбек без ошибки
    callback(null);
  }, timeout);

  // Возвращаем функцию для отмены
  return () => clearTimeout(timerId);
}

// Использование
const cancel = delayWithCallback(2000, (error) => {
  if (error) {
    console.error('Ошибка:', error.message);
    return;
  }
  console.log('Таймер сработал');
});

// Можем отменить таймер
// cancel();
```

### Промисификация таймеров в Node.js

В современном Node.js есть промисифицированные версии таймеров:

```javascript
// Node.js v16+ предоставляет промисифицированные таймеры
const { setTimeout, setInterval } = require('timers/promises');

// setTimeout возвращает Promise
async function example() {
  console.log('Начало');

  // Ожидание 1 секунды
  await setTimeout(1000);
  console.log('Прошла 1 секунда');

  // Ожидание еще 2 секунды с возвратом значения
  const value = await setTimeout(2000, 'completed');
  console.log('Прошло еще 2 секунды, результат:', value);
}

example();

// setInterval для асинхронной итерации
async function intervalExample() {
  const interval = setInterval(500);
  let count = 0;

  // for await...of для асинхронной итерации
  for await (const timestamp of interval) {
    console.log(`Тик ${count}:`, new Date(timestamp).toISOString());
    count++;

    if (count >= 5) {
      break; // Прерываем цикл
    }
  }

  console.log('Завершено');
}

intervalExample();
```

**Важно понимать:** Промисы используют колбеки внутри себя. Promise — это абстракция над колбеками, но не замена им. Большинство контрактов асинхронности в JavaScript построены на колбеках как на базовом примитиве.

### Адаптер для приведения любой функции к CLF

```javascript
// Универсальный адаптер
function toCLF(fn, callbackPosition = 0) {
  return function(...args) {
    const callback = args.pop(); // Последний аргумент - колбек

    // Вставляем колбек в нужную позицию оригинальной функции
    args.splice(callbackPosition, 0, callback);

    return fn(...args);
  };
}

// Использование
const delayC LF = toCLF(setTimeout, 0); // callback на позиции 0

delayCLF(1000, () => {
  console.log('Таймер сработал');
});

// Другой пример: функция с колбеком в середине
function processData(data, callback, options) {
  // callback на позиции 1
  callback(null, data.toUpperCase());
}

const processDataCLF = toCLF(processData, 1);

processDataCLF('hello', { format: 'upper' }, (error, result) => {
  console.log(result); // HELLO
});
```

---

## 7. Callback Hell: миф и реальность

### Что такое Callback Hell?

**Callback Hell (Пирамида погибели, Ад колбеков)** — это антипаттерн, при котором множественные вложенные колбеки создают "пирамиду" кода, растущую вправо.

**Важное понимание:** Callback Hell — это НЕ проблема самого паттерна колбеков, а результат **плохой организации кода**. При правильном подходе проблемы не возникает.

#### Пример: Классический Callback Hell

```javascript
const fs = require('fs');

// ПЛОХО: Глубокая вложенность, "пирамида погибели"
fs.readFile('./config.json', 'utf8', (err1, config) => {
  if (err1) {
    console.error('Ошибка чтения конфига:', err1);
    return;
  }

  const parsedConfig = JSON.parse(config);

  fs.readFile(parsedConfig.database, 'utf8', (err2, dbConfig) => {
    if (err2) {
      console.error('Ошибка чтения DB конфига:', err2);
      return;
    }

    const db = connectToDatabase(JSON.parse(dbConfig));

    db.query('SELECT * FROM users', (err3, users) => {
      if (err3) {
        console.error('Ошибка запроса:', err3);
        return;
      }

      users.forEach(user => {
        fetchUserDetails(user.id, (err4, details) => {
          if (err4) {
            console.error('Ошибка получения деталей:', err4);
            return;
          }

          generateReport(details, (err5, report) => {
            if (err5) {
              console.error('Ошибка генерации отчета:', err5);
              return;
            }

            saveReport(report, (err6) => {
              if (err6) {
                console.error('Ошибка сохранения:', err6);
                return;
              }

              console.log('Успех!');
            });
          });
        });
      });
    });
  });
});
```

**Проблемы этого кода:**

1. **"Елочка" кода** — вложенность растет вправо
2. **Плохая читаемость** — сложно понять логику
3. **Игнорирование ошибок** — некоторые ошибки только логируются
4. **Дублирование логики** — повторяющийся код обработки ошибок
5. **Невозможность тестирования** — нельзя тестировать отдельные шаги
6. **Смешение ответственностей** — всё в одной функции
7. **Сложность модификации** — трудно добавить новые шаги

### Решение 1: Именованные функции (Декомпозиция)

**Главный принцип:** Каждому колбеку дать имя и вынести в отдельную функцию.

```javascript
const fs = require('fs');

// ХОРОШО: Плоская структура с именованными функциями

// Шаг 6: Финальная обработка
function handleReportSaved(error) {
  if (error) {
    console.error('[ERROR] Не удалось сохранить отчет:', error.message);
    process.exit(1);
    return;
  }

  console.log('[SUCCESS] Отчет успешно сохранен');
  process.exit(0);
}

// Шаг 5: Сохранение отчета
function saveReportFile(error, report) {
  if (error) {
    console.error('[ERROR] Ошибка генерации отчета:', error.message);
    handleReportSaved(error);
    return;
  }

  const reportPath = './reports/output.json';
  const reportData = JSON.stringify(report, null, 2);

  fs.writeFile(reportPath, reportData, 'utf8', handleReportSaved);
}

// Шаг 4: Генерация отчета
function generateUserReport(error, userDetails) {
  if (error) {
    console.error('[ERROR] Ошибка получения деталей пользователя:', error.message);
    saveReportFile(error);
    return;
  }

  // Логика генерации отчета
  const report = {
    timestamp: new Date().toISOString(),
    user: userDetails,
    summary: `Report for ${userDetails.name}`
  };

  saveReportFile(null, report);
}

// Шаг 3: Получение деталей пользователя
function fetchFirstUserDetails(error, users) {
  if (error) {
    console.error('[ERROR] Ошибка запроса к БД:', error.message);
    generateUserReport(error);
    return;
  }

  if (users.length === 0) {
    generateUserReport(new Error('No users found'));
    return;
  }

  // Для примера берем первого пользователя
  const firstUser = users[0];

  // Имитация получения деталей
  setTimeout(() => {
    const details = { ...firstUser, fetchedAt: Date.now() };
    generateUserReport(null, details);
  }, 100);
}

// Шаг 2: Запрос к базе данных
function queryUsers(error, dbConfig) {
  if (error) {
    console.error('[ERROR] Ошибка чтения конфига БД:', error.message);
    fetchFirstUserDetails(error);
    return;
  }

  try {
    const parsed = JSON.parse(dbConfig);
    console.log('[INFO] Подключение к БД:', parsed.host);

    // Имитация запроса к БД
    setTimeout(() => {
      const users = [
        { id: 1, name: 'Alice' },
        { id: 2, name: 'Bob' }
      ];
      fetchFirstUserDetails(null, users);
    }, 100);
  } catch (parseError) {
    fetchFirstUserDetails(parseError);
  }
}

// Шаг 1: Чтение конфигурации БД
function readDatabaseConfig(error, config) {
  if (error) {
    console.error('[ERROR] Ошибка чтения основного конфига:', error.message);
    queryUsers(error);
    return;
  }

  try {
    const parsedConfig = JSON.parse(config);
    const dbConfigPath = parsedConfig.database || './db-config.json';

    fs.readFile(dbConfigPath, 'utf8', queryUsers);
  } catch (parseError) {
    queryUsers(parseError);
  }
}

// Точка входа
console.log('[START] Начало обработки');
fs.readFile('./config.json', 'utf8', readDatabaseConfig);
```

**Преимущества этого подхода:**

1. **Плоская структура** — нет глубокой вложенности
2. **Именованные функции** — каждый шаг имеет понятное имя
3. **Единая ответственность** — каждая функция делает одну вещь
4. **Тестируемость** — можно тестировать каждую функцию отдельно
5. **Читаемость** — легко понять последовательность операций
6. **Переиспользование** — функции можно экспортировать
7. **Обработка ошибок** — систематическая обработка на каждом шаге

**Но есть новая проблема:** Порядок объявления функций **обратный** порядку выполнения!

- Выполнение: readConfig → readDB → queryUsers → fetchDetails → generateReport → saveReport
- Объявление: saveReport → generateReport → fetchDetails → queryUsers → readDB → readConfig

Это неудобно для чтения и понимания логики.

### Решение 2: Восстановление логического порядка

Чтобы код читался в естественном порядке выполнения, используем объект или класс.

#### Вариант A: Использование объекта

```javascript
const fs = require('fs');

// Все шаги в логическом порядке выполнения
const pipeline = {};

// Шаг 1: Чтение конфигурации
pipeline.start = function() {
  console.log('[STEP 1] Чтение конфигурации');
  fs.readFile('./config.json', 'utf8', pipeline.processConfig);
};

// Шаг 2: Обработка конфигурации
pipeline.processConfig = function(error, config) {
  if (error) {
    console.error('[STEP 2] Ошибка:', error.message);
    pipeline.handleError(error);
    return;
  }

  try {
    const parsedConfig = JSON.parse(config);
    console.log('[STEP 2] Конфиг прочитан');

    fs.readFile(parsedConfig.database, 'utf8', pipeline.queryDatabase);
  } catch (parseError) {
    pipeline.handleError(parseError);
  }
};

// Шаг 3: Запрос к базе данных
pipeline.queryDatabase = function(error, dbConfig) {
  if (error) {
    console.error('[STEP 3] Ошибка:', error.message);
    pipeline.handleError(error);
    return;
  }

  console.log('[STEP 3] Выполнение запроса к БД');

  // Имитация запроса
  setTimeout(() => {
    const data = [{ id: 1, value: 'data' }];
    pipeline.callAPI(null, data);
  }, 100);
};

// Шаг 4: Вызов API
pipeline.callAPI = function(error, dbData) {
  if (error) {
    console.error('[STEP 4] Ошибка:', error.message);
    pipeline.handleError(error);
    return;
  }

  console.log('[STEP 4] Вызов внешнего API');

  // Имитация API запроса
  setTimeout(() => {
    const enrichedData = { ...dbData[0], enriched: true };
    pipeline.generateReport(null, enrichedData);
  }, 100);
};

// Шаг 5: Генерация отчета
pipeline.generateReport = function(error, apiData) {
  if (error) {
    console.error('[STEP 5] Ошибка:', error.message);
    pipeline.handleError(error);
    return;
  }

  console.log('[STEP 5] Генерация отчета');

  const report = {
    timestamp: new Date().toISOString(),
    data: apiData
  };

  pipeline.checkSuccess(null, report);
};

// Шаг 6: Проверка успешности
pipeline.checkSuccess = function(error, report) {
  if (error) {
    console.error('[STEP 6] Ошибка:', error.message);
    pipeline.handleError(error);
    return;
  }

  console.log('[STEP 6] Успех!');
  console.log('Итоговый отчет:', report);
};

// Обработчик ошибок
pipeline.handleError = function(error) {
  console.error('[FATAL] Критическая ошибка:', error.message);
  process.exit(1);
};

// Запуск pipeline
pipeline.start();
```

**Преимущества:**
- Шаги объявлены в порядке выполнения
- Код читается сверху вниз
- Логика понятна и последовательна
- Можно добавлять новые шаги без путаницы

#### Вариант B: Использование класса

```javascript
const fs = require('fs');

// Класс инкапсулирует всю логику обработки
class DataProcessor {
  constructor(configPath) {
    this.configPath = configPath;
    this.config = null;
    this.dbData = null;
    this.apiData = null;
  }

  // Точка входа
  start() {
    console.log('[START] Начало обработки');
    this.readConfig();
  }

  // Шаг 1: Чтение конфигурации
  readConfig() {
    console.log('[STEP 1] Чтение конфигурации');
    fs.readFile(this.configPath, 'utf8', this.handleConfig.bind(this));
  }

  // Шаг 2: Обработка конфигурации
  handleConfig(error, data) {
    if (error) {
      return this.handleError('Reading config', error);
    }

    try {
      this.config = JSON.parse(data);
      console.log('[STEP 2] Конфиг загружен:', this.config);
      this.queryDatabase();
    } catch (parseError) {
      this.handleError('Parsing config', parseError);
    }
  }

  // Шаг 3: Запрос к БД
  queryDatabase() {
    console.log('[STEP 3] Запрос к БД');

    // Имитация асинхронного запроса
    setTimeout(() => {
      this.dbData = [
        { id: 1, name: 'Item 1' },
        { id: 2, name: 'Item 2' }
      ];
      this.callAPI();
    }, 100);
  }

  // Шаг 4: Вызов API
  callAPI() {
    console.log('[STEP 4] Вызов API');

    // Имитация API запроса
    setTimeout(() => {
      this.apiData = {
        items: this.dbData,
        metadata: { source: 'API', version: '1.0' }
      };
      this.generateReport();
    }, 100);
  }

  // Шаг 5: Генерация отчета
  generateReport() {
    console.log('[STEP 5] Генерация отчета');

    const report = {
      timestamp: new Date().toISOString(),
      config: this.config,
      data: this.apiData,
      status: 'completed'
    };

    this.saveReport(report);
  }

  // Шаг 6: Сохранение отчета
  saveReport(report) {
    console.log('[STEP 6] Сохранение отчета');

    const reportPath = './report.json';
    const reportData = JSON.stringify(report, null, 2);

    fs.writeFile(reportPath, reportData, 'utf8', (error) => {
      if (error) {
        return this.handleError('Saving report', error);
      }

      this.finish(report);
    });
  }

  // Завершение
  finish(report) {
    console.log('[SUCCESS] Обработка завершена успешно!');
    console.log('Отчет сохранен, количество элементов:', report.data.items.length);
  }

  // Обработка ошибок
  handleError(step, error) {
    console.error(`[ERROR] Ошибка на шаге "${step}":`, error.message);
    process.exit(1);
  }
}

// Использование
const processor = new DataProcessor('./config.json');
processor.start();
```

**Преимущества класса:**

1. **Инкапсуляция состояния** — храним config, dbData, apiData в свойствах
2. **Логический порядок** — методы объявлены в порядке выполнения
3. **Автоматическая видимость** — методы видят друг друга через `this`
4. **Переиспользование** — можно создать несколько экземпляров
5. **Расширяемость** — легко добавлять новые методы
6. **Тестируемость** — можно тестировать каждый метод отдельно

**Важное замечание:** При использовании методов класса как колбеков нужно не забывать про `bind(this)`, иначе контекст будет потерян.

---

## 8. Систематическая обработка ошибок

### Золотое правило

**НИКОГДА не игнорируйте ошибки в колбеках!**

Ошибка — это не исключительная ситуация, а нормальная часть работы программы. Задача программиста — предусмотреть и обработать все возможные ошибки.

### Антипаттерны обработки ошибок

```javascript
const fs = require('fs');

// ПЛОХО #1: Полное игнорирование ошибки
fs.readFile('./file.txt', 'utf8', (error, data) => {
  // Ошибка не проверяется - ОПАСНО!
  console.log(data); // Может быть undefined
  const parsed = JSON.parse(data); // Может упасть
});

// ПЛОХО #2: Проверка без действия
fs.readFile('./file.txt', 'utf8', (error, data) => {
  if (error) {
    // Ничего не делаем - ошибка проглатывается
  }
  console.log(data); // Все равно используем data
});

// ПЛОХО #3: Только логирование без прекращения
fs.readFile('./file.txt', 'utf8', (error, data) => {
  if (error) {
    console.error(error);
    // Забыли return - продолжаем выполнение
  }
  const result = data.toUpperCase(); // Упадет, если data undefined
  console.log(result);
});

// ХОРОШО: Полная обработка
fs.readFile('./file.txt', 'utf8', (error, data) => {
  if (error) {
    console.error('Ошибка чтения файла:', error.message);
    // Логируем в систему логирования
    logger.error('File read failed', { file: './file.txt', error });
    // Передаем ошибку дальше или прекращаем выполнение
    return; // ОБЯЗАТЕЛЬНО!
  }

  // Безопасно используем data
  console.log('Содержимое:', data);
});
```

### Паттерны обработки ошибок

#### Паттерн 1: Логирование и прокидывание

```javascript
const fs = require('fs');

function readConfigFile(callback) {
  fs.readFile('./config.json', 'utf8', (error, data) => {
    if (error) {
      // Логируем для отладки
      console.error('[readConfigFile] Ошибка:', error.message);

      // Передаем ошибку дальше по цепочке
      callback(error);
      return;
    }

    try {
      const config = JSON.parse(data);
      callback(null, config);
    } catch (parseError) {
      console.error('[readConfigFile] Ошибка парсинга:', parseError.message);
      callback(parseError);
    }
  });
}

// Использование
readConfigFile((error, config) => {
  if (error) {
    console.error('Не удалось загрузить конфиг');
    process.exit(1);
    return;
  }

  console.log('Конфиг загружен:', config);
});
```

**Когда использовать:** В промежуточных функциях, где ошибка должна всплыть на уровень выше.

#### Паттерн 2: Обогащение ошибки контекстом

```javascript
function fetchUserData(userId, callback) {
  const filename = `./data/users/${userId}.json`;

  fs.readFile(filename, 'utf8', (error, data) => {
    if (error) {
      // Создаем новую ошибку с дополнительной информацией
      const enrichedError = new Error(
        `Failed to load user ${userId}: ${error.message}`
      );

      // Сохраняем оригинальную ошибку
      enrichedError.originalError = error;
      enrichedError.userId = userId;
      enrichedError.code = error.code; // ENOENT, EACCES, etc.

      callback(enrichedError);
      return;
    }

    try {
      const userData = JSON.parse(data);
      callback(null, userData);
    } catch (parseError) {
      const enrichedError = new Error(
        `Invalid JSON for user ${userId}: ${parseError.message}`
      );
      enrichedError.originalError = parseError;
      enrichedError.userId = userId;
      enrichedError.rawData = data; // Для отладки

      callback(enrichedError);
    }
  });
}

// Использование
fetchUserData(42, (error, user) => {
  if (error) {
    console.error('Error:', error.message);
    console.error('User ID:', error.userId);
    console.error('Error code:', error.code);
    console.error('Original error:', error.originalError);
    return;
  }

  console.log('User loaded:', user);
});
```

**Когда использовать:** Когда нужно добавить контекст выполнения для упрощения отладки.

#### Паттерн 3: Fallback значения

```javascript
function loadConfigWithDefaults(callback) {
  const defaultConfig = {
    port: 3000,
    host: 'localhost',
    env: 'development',
    debug: false
  };

  fs.readFile('./config.json', 'utf8', (error, data) => {
    if (error) {
      // Если файл не найден - используем значения по умолчанию
      if (error.code === 'ENOENT') {
        console.warn('Config file not found, using defaults');
        callback(null, defaultConfig);
        return;
      }

      // Для других ошибок - прокидываем дальше
      callback(error);
      return;
    }

    try {
      const userConfig = JSON.parse(data);

      // Объединяем с дефолтными значениями
      const config = { ...defaultConfig, ...userConfig };

      callback(null, config);
    } catch (parseError) {
      console.warn('Invalid config JSON, using defaults');
      callback(null, defaultConfig);
    }
  });
}
```

**Когда использовать:** Для опциональных конфигураций, где можно использовать разумные значения по умолчанию.

#### Паттерн 4: Retry логика

```javascript
function readFileWithRetry(filepath, options, callback) {
  const maxRetries = options.maxRetries || 3;
  const retryDelay = options.retryDelay || 1000;
  let attempts = 0;

  function attempt() {
    attempts++;
    console.log(`Attempt ${attempts}/${maxRetries}`);

    fs.readFile(filepath, 'utf8', (error, data) => {
      if (error) {
        // Если еще есть попытки и ошибка временная
        if (attempts < maxRetries && isRetryableError(error)) {
          console.warn(`Retry in ${retryDelay}ms...`);
          setTimeout(attempt, retryDelay);
          return;
        }

        // Исчерпали попытки или ошибка не временная
        const finalError = new Error(
          `Failed after ${attempts} attempts: ${error.message}`
        );
        finalError.attempts = attempts;
        finalError.originalError = error;

        callback(finalError);
        return;
      }

      // Успех
      callback(null, data);
    });
  }

  function isRetryableError(error) {
    // Повторяем только для временных ошибок
    return error.code === 'EBUSY' ||
           error.code === 'EAGAIN' ||
           error.code === 'ETIMEDOUT';
  }

  attempt();
}

// Использование
readFileWithRetry('./file.txt', { maxRetries: 5, retryDelay: 2000 }, (error, data) => {
  if (error) {
    console.error('Failed after retries:', error.message);
    console.error('Total attempts:', error.attempts);
    return;
  }

  console.log('File loaded:', data);
});
```

**Когда использовать:** Для операций, которые могут временно не удаваться (сетевые запросы, занятые файлы).

#### Паттерн 5: Централизованная обработка ошибок

```javascript
// Централизованный обработчик ошибок
class ErrorHandler {
  constructor(logger) {
    this.logger = logger;
  }

  handle(context, error, callback) {
    // Логирование
    this.logger.error(`[${context}] Error occurred`, {
      message: error.message,
      code: error.code,
      stack: error.stack
    });

    // Определяем тип ошибки и стратегию обработки
    if (error.code === 'ENOENT') {
      this.logger.warn(`[${context}] File not found - using fallback`);
      // Можем вернуть fallback значение
      callback(null, this.getFallbackValue(context));
      return;
    }

    if (error.code === 'EACCES') {
      this.logger.error(`[${context}] Permission denied - critical error`);
      // Критическая ошибка - останавливаем процесс
      process.exit(1);
    }

    // По умолчанию - прокидываем ошибку
    callback(error);
  }

  getFallbackValue(context) {
    // Fallback значения для разных контекстов
    const fallbacks = {
      'config': { port: 3000, host: 'localhost' },
      'users': [],
      'settings': {}
    };

    return fallbacks[context] || null;
  }
}

// Использование
const errorHandler = new ErrorHandler(console);

function readConfig(callback) {
  fs.readFile('./config.json', 'utf8', (error, data) => {
    if (error) {
      errorHandler.handle('config', error, callback);
      return;
    }

    try {
      const config = JSON.parse(data);
      callback(null, config);
    } catch (parseError) {
      errorHandler.handle('config', parseError, callback);
    }
  });
}
```

### Комплексный пример с полной обработкой ошибок

```javascript
const fs = require('fs');

// Полная цепочка с обработкой ошибок на каждом шаге

// Финальный обработчик
function handleCompletion(error) {
  if (error) {
    console.error('\n=== OPERATION FAILED ===');
    console.error('Error:', error.message);
    console.error('Code:', error.code);
    console.error('========================\n');
    process.exit(1);
    return;
  }

  console.log('\n=== OPERATION SUCCEEDED ===');
  console.log('All steps completed successfully');
  console.log('===========================\n');
  process.exit(0);
}

// Шаг 4: Сохранение отчета
function saveReport(error, report) {
  if (error) {
    console.error('[Step 4] Report generation failed:', error.message);
    handleCompletion(error);
    return;
  }

  console.log('[Step 4] Saving report...');

  fs.writeFile('./output/report.json', JSON.stringify(report, null, 2), 'utf8', (writeError) => {
    if (writeError) {
      const enrichedError = new Error(`Failed to save report: ${writeError.message}`);
      enrichedError.originalError = writeError;
      handleCompletion(enrichedError);
      return;
    }

    console.log('[Step 4] Report saved successfully');
    handleCompletion(null);
  });
}

// Шаг 3: Генерация отчета
function generateReport(error, apiData) {
  if (error) {
    console.error('[Step 3] API call failed:', error.message);
    saveReport(error);
    return;
  }

  console.log('[Step 3] Generating report...');

  try {
    const report = {
      timestamp: new Date().toISOString(),
      data: apiData,
      summary: `Processed ${apiData.length} records`,
      status: 'completed'
    };

    console.log('[Step 3] Report generated');
    saveReport(null, report);
  } catch (err) {
    const enrichedError = new Error(`Report generation error: ${err.message}`);
    enrichedError.originalError = err;
    saveReport(enrichedError);
  }
}

// Шаг 2: Вызов API
function callAPI(error, dbData) {
  if (error) {
    console.error('[Step 2] Database query failed:', error.message);
    generateReport(error);
    return;
  }

  console.log('[Step 2] Calling external API...');

  // Имитация API вызова с возможностью ошибки
  setTimeout(() => {
    const success = Math.random() > 0.2; // 80% успеха

    if (!success) {
      const apiError = new Error('API request timeout');
      apiError.code = 'ETIMEDOUT';
      generateReport(apiError);
      return;
    }

    const enrichedData = dbData.map(item => ({
      ...item,
      enriched: true,
      apiTimestamp: Date.now()
    }));

    console.log('[Step 2] API call successful');
    generateReport(null, enrichedData);
  }, 500);
}

// Шаг 1: Запрос к БД
function queryDatabase(error, config) {
  if (error) {
    console.error('[Step 1] Config loading failed:', error.message);
    callAPI(error);
    return;
  }

  console.log('[Step 1] Querying database...');

  // Имитация запроса к БД
  setTimeout(() => {
    const dbData = [
      { id: 1, value: 'Record 1' },
      { id: 2, value: 'Record 2' },
      { id: 3, value: 'Record 3' }
    ];

    console.log('[Step 1] Database query successful');
    callAPI(null, dbData);
  }, 300);
}

// Точка входа: Чтение конфигурации
function startProcess() {
  console.log('\n=== STARTING PROCESS ===\n');
  console.log('[Step 0] Loading configuration...');

  fs.readFile('./config.json', 'utf8', (error, data) => {
    if (error) {
      // Обрабатываем разные типы ошибок по-разному
      if (error.code === 'ENOENT') {
        console.warn('[Step 0] Config not found, using defaults');
        const defaultConfig = { database: 'default_db' };
        queryDatabase(null, defaultConfig);
        return;
      }

      const enrichedError = new Error(`Config load error: ${error.message}`);
      enrichedError.code = error.code;
      queryDatabase(enrichedError);
      return;
    }

    try {
      const config = JSON.parse(data);
      console.log('[Step 0] Configuration loaded');
      queryDatabase(null, config);
    } catch (parseError) {
      const enrichedError = new Error(`Config parse error: ${parseError.message}`);
      queryDatabase(enrichedError);
    }
  });
}

// Запуск
startProcess();
```

**Ключевые принципы обработки ошибок:**

1. **Проверяйте каждый колбек** — `if (error)` первым делом
2. **Всегда используйте `return`** после обработки ошибки
3. **Пробрасывайте или обрабатывайте** — не проглатывайте ошибки
4. **Добавляйте контекст** — обогащайте ошибки информацией
5. **Логируйте системно** — используйте logger, а не только console
6. **Различайте типы ошибок** — обрабатывайте по-разному
7. **Используйте try-catch** — для синхронных операций в асинхронном коде

---

## 9. Ключевые выводы и best practices

### Контракт Callback-last-error-first

**Резюме:**
- Колбек передается **последним** аргументом
- Ошибка передается **первым** аргументом в колбек
- Это стандарт для всей экосистемы Node.js
- Обеспечивает единообразие и предсказуемость API

**Следуйте контракту:**
```javascript
// ПРАВИЛЬНО
function asyncOp(param1, param2, callback) {
  callback(error, result);
}

// НЕПРАВИЛЬНО
function asyncOp(callback, param1, param2) { // колбек не последний
  callback(result, error); // ошибка не первая
}
```

### Callback Hell — это миф

**Важное понимание:**

Callback Hell НЕ является неизбежной проблемой колбеков. Это результат:
- Плохой организации кода
- Неиспользования именованных функций
- Отсутствия декомпозиции
- Нарушения принципа единственной ответственности

**При правильном подходе:**
- Код легко читается
- Каждая функция имеет четкую ответственность
- Ошибки обрабатываются систематически
- Код легко тестировать и поддерживать

### Именованные колбеки — обязательная практика

**Преимущества:**

1. **Читаемость** — код самодокументируется
2. **Отладка** — понятные stack traces
3. **Тестируемость** — каждую функцию можно тестировать отдельно
4. **Переиспользование** — функции можно экспортировать
5. **Производительность** — движок может лучше оптимизировать

```javascript
// ПЛОХО
fs.readFile('./file.txt', 'utf8', (error, data) => {
  if (error) return console.error(error);
  process(data, (err, result) => {
    if (err) return console.error(err);
    save(result, (e) => {
      if (e) return console.error(e);
      console.log('Done');
    });
  });
});

// ХОРОШО
function handleFileRead(error, data) {
  if (error) return console.error('Read error:', error);
  process(data, handleProcess);
}

function handleProcess(error, result) {
  if (error) return console.error('Process error:', error);
  save(result, handleSave);
}

function handleSave(error) {
  if (error) return console.error('Save error:', error);
  console.log('Done');
}

fs.readFile('./file.txt', 'utf8', handleFileRead);
```

### Обработка ошибок — критически важна

**Золотые правила:**

1. **Никогда не игнорируйте** первый аргумент error-first колбеков
2. **Всегда проверяйте** ошибку перед использованием результата
3. **Обязательно используйте return** после обработки ошибки
4. **Логируйте или пробрасывайте** — не проглатывайте ошибки
5. **Добавляйте контекст** — обогащайте ошибки для отладки

```javascript
// Правильная обработка
function operation(callback) {
  asyncTask((error, data) => {
    if (error) {
      logger.error('Task failed', { error: error.message });
      callback(error); // Пробрасываем
      return; // Обязательно!
    }

    // Безопасно используем data
    callback(null, processData(data));
  });
}
```

### Колбеки как фундамент асинхронности

**Ключевое понимание:**

Колбеки — это **базовый примитив** асинхронности в JavaScript. Все остальные контракты используют колбеки:

- **Promises** — используют колбеки в `.then()` и `.catch()`
- **async/await** — синтаксический сахар над промисами
- **Observables** — используют колбеки для подписчиков
- **Event Emitters** — колбеки как обработчики событий
- **Async Iterators** — колбеки в реализации

**Поэтому понимание колбеков критически важно для:**
- Работы с любым асинхронным кодом
- Понимания более сложных паттернов
- Отладки и оптимизации
- Создания собственных асинхронных абстракций

### Практические рекомендации

#### При написании функций с колбеками:

1. **Следуйте CLF** — callback-last-error-first
2. **Документируйте** — особенно структуру ошибок и результатов
3. **Вызывайте колбек ровно один раз** — избегайте множественных вызовов
4. **Делайте асинхронность явной** — если может быть async, должно быть всегда async
5. **Используйте try-catch** — для синхронных операций в асинхронном контексте

```javascript
function goodAsyncFunction(param, callback) {
  // Валидация
  if (!param) {
    process.nextTick(() => {
      callback(new Error('param is required'));
    });
    return;
  }

  // Асинхронная операция
  asyncOperation(param, (error, result) => {
    if (error) {
      callback(error);
      return;
    }

    // Синхронная обработка в try-catch
    try {
      const processed = processResult(result);
      callback(null, processed);
    } catch (err) {
      callback(err);
    }
  });
}
```

#### При использовании колбеков:

1. **Всегда обрабатывайте ошибки** — первым делом проверяйте `error`
2. **Используйте именованные функции** — для сложных цепочек
3. **Не создавайте глубокие вложенности** — декомпозируйте код
4. **Группируйте связанные операции** — в классы или модули
5. **Добавляйте логирование** — для отладки асинхронных операций

#### При рефакторинге:

1. **Начните с именования** — дайте имена анонимным функциям
2. **Извлеките логику** — каждая функция = одна ответственность
3. **Добавьте обработку ошибок** — на каждый уровень
4. **Используйте классы/объекты** — для восстановления порядка
5. **Рассмотрите промисы** — для нового кода

---

## 10. Дополнительные материалы и связь с другими паттернами

### Эволюция асинхронных паттернов

Колбеки — это первый шаг в эволюции асинхронного программирования в JavaScript:

```
Callbacks (ES5)
    ↓
Thenable (переходный паттерн)
    ↓
Promises (ES6/ES2015)
    ↓
async/await (ES2017)
    ↓
AsyncIterators (ES2018)
    ↓
Top-level await (ES2022)
```

Каждый следующий паттерн решает проблемы предыдущего, но использует его как основу.

### Связь с другими паттернами

#### Thenable

**Thenable** — предшественник промисов, объект с методом `.then()`:

```javascript
// Колбек
function fetchData(callback) {
  setTimeout(() => {
    callback(null, 'data');
  }, 1000);
}

// Thenable (упрощенная версия)
function fetchDataThenable() {
  return {
    then(onSuccess, onError) {
      setTimeout(() => {
        onSuccess('data');
      }, 1000);
    }
  };
}

// Использование
fetchDataThenable().then(
  data => console.log(data),
  error => console.error(error)
);
```

#### Promise

**Promise** — стандартизированный способ работы с асинхронностью:

```javascript
// Промисификация callback-функции
function promisify(fn) {
  return function(...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (error, result) => {
        if (error) {
          reject(error);
        } else {
          resolve(result);
        }
      });
    });
  };
}

// Использование
const readFilePromise = promisify(fs.readFile);

readFilePromise('./file.txt', 'utf8')
  .then(data => console.log(data))
  .catch(error => console.error(error));
```

#### Async/Await

**Async/await** — синтаксический сахар над промисами:

```javascript
// Callback
fs.readFile('./file.txt', 'utf8', (error, data) => {
  if (error) return console.error(error);
  console.log(data);
});

// Promise
readFilePromise('./file.txt', 'utf8')
  .then(data => console.log(data))
  .catch(error => console.error(error));

// Async/await
async function readFile() {
  try {
    const data = await readFilePromise('./file.txt', 'utf8');
    console.log(data);
  } catch (error) {
    console.error(error);
  }
}
```

### Node.js util.promisify

Node.js предоставляет встроенную утилиту для промисификации:

```javascript
const util = require('util');
const fs = require('fs');

// Промисификация всего модуля
const readFilePromise = util.promisify(fs.readFile);
const writeFilePromise = util.promisify(fs.writeFile);

// Использование с async/await
async function processFile() {
  try {
    const data = await readFilePromise('./input.txt', 'utf8');
    const processed = data.toUpperCase();
    await writeFilePromise('./output.txt', processed, 'utf8');
    console.log('File processed');
  } catch (error) {
    console.error('Error:', error.message);
  }
}

processFile();
```

### Для самостоятельного изучения

**Практические задания:**

1. Напишите 3-4 функции с контрактом CLF для работы с данными
2. Создайте цепочку из 5 асинхронных операций с именованными колбеками
3. Рефакторите пример callback hell в плоскую структуру
4. Реализуйте retry-логику для асинхронной операции
5. Создайте универсальный обработчик ошибок для колбеков

**Изучите исходный код:**

- Node.js `fs` module — примеры CLF в действии
- Express.js middleware — использование колбеков для цепочки обработки
- Async.js library — утилиты для работы с колбеками

**Следующие темы:**

- **Thenable** — переходный паттерн между callbacks и promises
- **Promise API** — современный стандарт асинхронности
- **Iterator & AsyncIterator** — для работы с последовательностями
- **Disposable** — управление ресурсами с автоматическим освобождением

---

## Итоговое резюме

### Что мы изучили

1. **Эволюция от sync к async** — как функции превращаются в колбеки
2. **Контракт CLF** — callback-last-error-first как стандарт Node.js
3. **Синхронные vs асинхронные колбеки** — важное различие
4. **Работа с fs модулем** — практические примеры I/O операций
5. **Именованные колбеки** — решение проблем читаемости и тестируемости
6. **Адаптация API** — приведение к единому контракту
7. **Решение callback hell** — через декомпозицию и именование
8. **Обработка ошибок** — систематический подход на каждом уровне

### Ключевые takeaways

- **Колбеки — фундамент** асинхронности в JavaScript
- **CLF контракт** обеспечивает единообразие API
- **Callback hell — миф** при правильной организации кода
- **Именованные функции** критически важны для качества кода
- **Обработка ошибок** должна быть на каждом уровне
- **Все асинхронные паттерны** используют колбеки как основу

### Что дальше

Колбеки — это база. Понимая их, вы готовы к изучению:
- Промисов и их комбинаторов
- Async/await синтаксиса
- Асинхронных итераторов
- Event-driven архитектуры
- Reactive programming с Observables

---

**Автор лекции:** Тимур Шемсединов
**Курс:** Асинхронное программирование в Node.js
**Неделя:** 2 - Native features in language and platforms
**Ссылка на видео:** https://youtu.be/vcOGCWL-eZc

---

**Примечание:** Все примеры кода доступны в репозитории курса. Практикуйтесь, экспериментируйте и не бойтесь колбеков — они ваши друзья в асинхронном программировании!
