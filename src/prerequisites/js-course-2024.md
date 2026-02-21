# JavaScript Basics

## 1. Введение

Это покрывает **необходимый минимум** синтаксиса JavaScript — примерно 20-25% языка, которые позволят писать практически значимый код. JavaScript присутствует везде: на фронтенде, бэкенде, в мобильной разработке. При этом он является одним из самых сложных языков из-за накопившихся за историю синтаксических конструкций.

### Настройка среды

```bash
# Проверка версии Git
git -v

# Проверка версии Node.js
node -v
```

Рекомендуемые инструменты:
- **ОС**: Linux (Fedora, Ubuntu), macOS, FreeBSD (Windows тоже поддерживается)
- **Браузер**: Chrome или Chromium
- **Редактор кода**: любой с подсветкой синтаксиса

---

## 2. Идентификаторы: переменные и константы

### Переменные (let)

Переменные объявляются ключевым словом `let` и могут изменять своё значение.

```javascript
// Объявление переменной
let amount = 0;

// Изменение значения
amount = 100;

// Составное присваивание
amount += 50; // amount теперь 150
```

### Константы (const)

Константы объявляются ключевым словом `const` и не могут быть переприсвоены.

```javascript
// Глобальная константа (SCREAMING_SNAKE_CASE)
const MAX_PURCHASE = 2000;

// Локальная константа (camelCase)
const groupName = 'Electronics';
```

### Стили именования

| Стиль | Использование | Пример |
|-------|---------------|--------|
| SCREAMING_SNAKE_CASE | Глобальные константы | `MAX_PURCHASE`, `API_URL` |
| camelCase | Переменные, функции, параметры | `groupName`, `calculateTotal` |
| PascalCase | Классы, конструкторы | `UserService`, `ProductModel` |

### Хорошие и плохие имена

```javascript
// ✅ Хорошо
const prices = [100, 200, 300];
const groupName = 'Electronics';
const calculateTotal = (items) => { /* ... */ };

// ❌ Плохо
const numbers = [100, 200, 300];     // Непонятно, что это цены
const g_n = 'Electronics';           // Сокращение непонятно
const itogo2 = 500;                  // Нумерация — плохая практика
const ERCODE = 404;                  // Аббревиатура без смысла
```

### Правила объявления

```javascript
// ✅ Хорошо — каждая переменная на отдельной строке
let i = 0;
let counter = 0;
const year = 2024;
const eventName = 'Conference';

// ❌ Плохо — несколько переменных в одной строке
let i = 0, counter = 0, year = 2024, eventName = 'Conference';
```

---

## 3. Типы данных

### Примитивные типы

JavaScript имеет 7 примитивных типов данных:

```javascript
// 1. Number — числа (целые и с плавающей точкой)
const integer = 42;
const float = 3.14;
const infinity = Infinity;
const notANumber = NaN;

// 2. BigInt — большие целые числа
const bigNumber = 9007199254740991n;
const anotherBig = BigInt('123456789012345678901234567890');

// 3. String — строки
const singleQuotes = 'Hello';
const doubleQuotes = "World";
const templateLiteral = `Hello, ${name}!`;

// 4. Boolean — логический тип
const isActive = true;
const isDeleted = false;

// 5. undefined — значение не задано
let notDefined;
console.log(notDefined); // undefined

// 6. null — намеренное отсутствие значения
const emptyObject = null;

// 7. Symbol — уникальный идентификатор
const sym = Symbol('description');
```

### Оператор typeof

```javascript
typeof 42;           // 'number'
typeof 'hello';      // 'string'
typeof true;         // 'boolean'
typeof undefined;    // 'undefined'
typeof null;         // 'object' (историческая ошибка!)
typeof {};           // 'object'
typeof [];           // 'object'
typeof function(){}; // 'function'
typeof Symbol();     // 'symbol'
typeof 42n;          // 'bigint'
```

### Особенности типов

```javascript
// undefined vs null
// undefined — для примитивных типов, когда значение не задано
// null — для объектов, когда объект намеренно пуст

let count;          // undefined (примитив не задан)
let user = null;    // null (объект намеренно пуст)

// NaN — Not a Number
const result = 'hello' / 2;  // NaN
isNaN(result);               // true

// Infinity
const inf = 1 / 0;           // Infinity
const negInf = -1 / 0;       // -Infinity
```

---

## 4. Функции и область видимости

### Стрелочные функции (Arrow Functions)

Основной синтаксис, используемый в современном JavaScript:

```javascript
// Полная форма
const sum = (a, b) => {
  return a + b;
};

// Краткая форма (неявный return)
const sum = (a, b) => a + b;

// Один параметр — скобки можно опустить
const double = x => x * 2;

// Без параметров
const greet = () => 'Hello!';
```

### Function Declaration vs Function Expression

```javascript
// Function Declaration — поднимается (hoisting)
function multiply(a, b) {
  return a * b;
}

// Function Expression — не поднимается
const divide = function(a, b) {
  return a / b;
};

// Вызов до объявления
console.log(multiply(2, 3)); // 6 — работает (hoisting)
console.log(divide(6, 2));   // Ошибка! (не поднимается)
```

### Параметры и аргументы

```javascript
// Параметры — внутри функции
// Аргументы — при вызове функции

const greet = (name, greeting) => {
  return `${greeting}, ${name}!`;
};

// При вызове передаём аргументы
greet('Alice', 'Hello'); // 'Hello, Alice!'
```

### Параметры по умолчанию

```javascript
const createUser = (name, age = 18, role = 'user') => {
  return { name, age, role };
};

createUser('Alice');              // { name: 'Alice', age: 18, role: 'user' }
createUser('Bob', 25);            // { name: 'Bob', age: 25, role: 'user' }
createUser('Admin', 30, 'admin'); // { name: 'Admin', age: 30, role: 'admin' }
```

### Rest-параметры

```javascript
// Собрать все аргументы в массив
const sum = (...numbers) => {
  return numbers.reduce((acc, n) => acc + n, 0);
};

sum(1, 2, 3);       // 6
sum(1, 2, 3, 4, 5); // 15

// Комбинация обычных и rest-параметров
const greetAll = (greeting, ...names) => {
  return names.map(name => `${greeting}, ${name}!`);
};

greetAll('Hello', 'Alice', 'Bob', 'Charlie');
// ['Hello, Alice!', 'Hello, Bob!', 'Hello, Charlie!']
```

### Область видимости (Scope)

```javascript
const globalVar = 'global';

const outerFunction = () => {
  const outerVar = 'outer';
  
  const innerFunction = () => {
    const innerVar = 'inner';
    
    // Доступ ко всем переменным
    console.log(globalVar); // 'global'
    console.log(outerVar);  // 'outer'
    console.log(innerVar);  // 'inner'
  };
  
  innerFunction();
  // console.log(innerVar); // Ошибка! innerVar недоступна
};

outerFunction();
// console.log(outerVar); // Ошибка! outerVar недоступна
```

### Блочная область видимости

```javascript
let level = 1;
console.log(level); // 1

{
  let level = 2; // Новая переменная в блоке
  console.log(level); // 2
  
  {
    let level = 3;
    console.log(level); // 3
  }
  
  console.log(level); // 2
}

console.log(level); // 1
```

---

## 5. Условия и ветвление

### if / else if / else

```javascript
const checkAge = (age) => {
  if (age < 0) {
    return 'Некорректный возраст';
  } else if (age < 18) {
    return 'Несовершеннолетний';
  } else if (age < 65) {
    return 'Взрослый';
  } else {
    return 'Пенсионер';
  }
};

checkAge(25); // 'Взрослый'
```

### Тернарный оператор

```javascript
// condition ? valueIfTrue : valueIfFalse
const status = age >= 18 ? 'adult' : 'minor';

// Вложенные тернарные операторы (не рекомендуется злоупотреблять)
const category = age < 13 ? 'child' 
               : age < 20 ? 'teenager' 
               : 'adult';
```

### switch / case

```javascript
const getDayName = (dayNumber) => {
  switch (dayNumber) {
    case 1:
      return 'Понедельник';
    case 2:
      return 'Вторник';
    case 3:
      return 'Среда';
    case 4:
      return 'Четверг';
    case 5:
      return 'Пятница';
    case 6:
    case 7:
      return 'Выходной';
    default:
      return 'Неизвестный день';
  }
};
```

### Логические операторы

```javascript
// && — И (возвращает первое ложное или последнее значение)
const result1 = true && 'hello';  // 'hello'
const result2 = false && 'hello'; // false
const result3 = 'a' && 'b' && 'c'; // 'c'

// || — ИЛИ (возвращает первое истинное или последнее значение)
const result4 = false || 'default'; // 'default'
const result5 = 'value' || 'default'; // 'value'

// ?? — Nullish coalescing (только для null/undefined)
const result6 = null ?? 'default';      // 'default'
const result7 = undefined ?? 'default'; // 'default'
const result8 = 0 ?? 'default';         // 0 (0 не null/undefined)
const result9 = '' ?? 'default';        // '' (пустая строка не null/undefined)
```

### Сравнение значений

```javascript
// === — строгое равенство (без приведения типов)
5 === 5;     // true
5 === '5';   // false
null === undefined; // false

// == — нестрогое равенство (с приведением типов) — НЕ РЕКОМЕНДУЕТСЯ
5 == '5';    // true (строка приводится к числу)
null == undefined; // true

// Всегда используйте === и !==
```

---

## 6. Циклы

### for

```javascript
// Классический for
for (let i = 0; i < 5; i++) {
  console.log(i); // 0, 1, 2, 3, 4
}

// Обратный цикл
for (let i = 4; i >= 0; i--) {
  console.log(i); // 4, 3, 2, 1, 0
}

// С шагом
for (let i = 0; i < 10; i += 2) {
  console.log(i); // 0, 2, 4, 6, 8
}
```

### for...of (итерация по значениям)

```javascript
const fruits = ['apple', 'banana', 'orange'];

for (const fruit of fruits) {
  console.log(fruit);
}
// 'apple', 'banana', 'orange'

// Для строк
for (const char of 'Hello') {
  console.log(char);
}
// 'H', 'e', 'l', 'l', 'o'

// Для Map
const map = new Map([['a', 1], ['b', 2]]);
for (const [key, value] of map) {
  console.log(key, value);
}
// 'a' 1, 'b' 2
```

### for...in (итерация по ключам)

```javascript
const user = { name: 'Alice', age: 25, city: 'Minsk' };

for (const key in user) {
  console.log(`${key}: ${user[key]}`);
}
// 'name: Alice', 'age: 25', 'city: Minsk'

// ⚠️ Не используйте for...in для массивов!
const arr = ['a', 'b', 'c'];
for (const index in arr) {
  console.log(index); // '0', '1', '2' — это строки!
}
```

### while и do...while

```javascript
// while — проверка перед итерацией
let count = 0;
while (count < 3) {
  console.log(count);
  count++;
}
// 0, 1, 2

// do...while — проверка после итерации (минимум 1 выполнение)
let num = 0;
do {
  console.log(num);
  num++;
} while (num < 3);
// 0, 1, 2
```

### break и continue

```javascript
// break — выход из цикла
for (let i = 0; i < 10; i++) {
  if (i === 5) break;
  console.log(i);
}
// 0, 1, 2, 3, 4

// continue — пропуск итерации
for (let i = 0; i < 5; i++) {
  if (i === 2) continue;
  console.log(i);
}
// 0, 1, 3, 4

// Метки для вложенных циклов
outer: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) break outer;
    console.log(i, j);
  }
}
// 0 0, 0 1, 0 2, 1 0
```

---

## 7. Коллекции: массивы и объекты

### Массивы (Array)

```javascript
// Создание массивов
const empty = [];
const numbers = [1, 2, 3, 4, 5];
const mixed = [1, 'hello', true, null, { key: 'value' }];

// Доступ по индексу
numbers[0]; // 1
numbers[2]; // 3
numbers[numbers.length - 1]; // 5 (последний элемент)

// Изменение элементов
numbers[0] = 10;
numbers.push(6);    // Добавить в конец
numbers.pop();      // Удалить с конца
numbers.unshift(0); // Добавить в начало
numbers.shift();    // Удалить с начала
```

### Методы массивов

```javascript
const numbers = [1, 2, 3, 4, 5];

// map — преобразование каждого элемента
const doubled = numbers.map(n => n * 2);
// [2, 4, 6, 8, 10]

// filter — фильтрация по условию
const even = numbers.filter(n => n % 2 === 0);
// [2, 4]

// find — поиск первого подходящего
const found = numbers.find(n => n > 3);
// 4

// findIndex — индекс первого подходящего
const index = numbers.findIndex(n => n > 3);
// 3

// some — есть ли хотя бы один подходящий
const hasEven = numbers.some(n => n % 2 === 0);
// true

// every — все ли подходят
const allPositive = numbers.every(n => n > 0);
// true

// reduce — свёртка массива
const sum = numbers.reduce((acc, n) => acc + n, 0);
// 15

// includes — содержит ли элемент
numbers.includes(3); // true
numbers.includes(10); // false
```

### Деструктуризация массивов

```javascript
const coordinates = [10, 20, 30];

// Извлечение значений
const [x, y, z] = coordinates;
console.log(x, y, z); // 10 20 30

// Пропуск элементов
const [first, , third] = coordinates;
console.log(first, third); // 10 30

// Rest-оператор
const [head, ...tail] = [1, 2, 3, 4, 5];
console.log(head); // 1
console.log(tail); // [2, 3, 4, 5]

// Значения по умолчанию
const [a, b, c = 0] = [1, 2];
console.log(c); // 0
```

### Объекты (Object)

```javascript
// Создание объектов
const user = {
  name: 'Alice',
  age: 25,
  email: 'alice@example.com',
  isActive: true
};

// Доступ к свойствам
user.name;       // 'Alice'
user['age'];     // 25

// Динамический доступ
const key = 'email';
user[key];       // 'alice@example.com'

// Изменение и добавление свойств
user.age = 26;
user.city = 'Minsk';

// Удаление свойств
delete user.isActive;
```

### Методы объектов

```javascript
const user = { name: 'Alice', age: 25, city: 'Minsk' };

// Object.keys — массив ключей
Object.keys(user); // ['name', 'age', 'city']

// Object.values — массив значений
Object.values(user); // ['Alice', 25, 'Minsk']

// Object.entries — массив пар [ключ, значение]
Object.entries(user); // [['name', 'Alice'], ['age', 25], ['city', 'Minsk']]

// Object.assign — копирование/слияние
const copy = Object.assign({}, user);
const merged = Object.assign({}, user, { role: 'admin' });

// Spread-оператор (современный способ)
const copy2 = { ...user };
const merged2 = { ...user, role: 'admin' };
```

### Деструктуризация объектов

```javascript
const user = {
  name: 'Alice',
  age: 25,
  address: {
    city: 'Minsk',
    country: 'Belarus'
  }
};

// Простая деструктуризация
const { name, age } = user;
console.log(name, age); // 'Alice' 25

// Переименование
const { name: userName, age: userAge } = user;
console.log(userName, userAge); // 'Alice' 25

// Значения по умолчанию
const { name, role = 'user' } = user;
console.log(role); // 'user'

// Вложенная деструктуризация
const { address: { city, country } } = user;
console.log(city, country); // 'Minsk' 'Belarus'

// В параметрах функции
const greet = ({ name, age }) => {
  return `Hello, ${name}! You are ${age} years old.`;
};
greet(user); // 'Hello, Alice! You are 25 years old.'
```

### JSON

```javascript
const user = { name: 'Alice', age: 25 };

// Объект → JSON-строка
const jsonString = JSON.stringify(user);
// '{"name":"Alice","age":25}'

// JSON-строка → Объект
const parsed = JSON.parse(jsonString);
// { name: 'Alice', age: 25 }

// Форматированный вывод
const formatted = JSON.stringify(user, null, 2);
/*
{
  "name": "Alice",
  "age": 25
}
*/
```

---

## 8. Set и Map

### Set (Множество)

Set хранит только уникальные значения.

```javascript
// Создание Set
const set = new Set();
const setFromArray = new Set([1, 2, 3, 2, 1]);
console.log(setFromArray); // Set { 1, 2, 3 }

// Методы Set
set.add(1);
set.add(2);
set.add(1); // Игнорируется (уже есть)
set.has(1); // true
set.delete(1);
set.size;   // 1
set.clear();

// Итерация
for (const value of setFromArray) {
  console.log(value);
}

// Преобразование в массив
const arr = [...setFromArray];
const arr2 = Array.from(setFromArray);

// Удаление дубликатов из массива
const unique = [...new Set([1, 2, 2, 3, 3, 3])];
// [1, 2, 3]
```

### Map (Хеш-таблица)

Map хранит пары ключ-значение, где ключом может быть любой тип.

```javascript
// Создание Map
const map = new Map();
const mapFromArray = new Map([
  ['name', 'Alice'],
  ['age', 25]
]);

// Методы Map
map.set('key', 'value');
map.get('key');     // 'value'
map.has('key');     // true
map.delete('key');
map.size;           // 0
map.clear();

// Ключи любого типа (в отличие от объектов)
const objKey = { id: 1 };
map.set(objKey, 'value for object');
map.get(objKey); // 'value for object'

// Итерация
for (const [key, value] of mapFromArray) {
  console.log(key, value);
}

map.keys();    // Iterator по ключам
map.values();  // Iterator по значениям
map.entries(); // Iterator по парам [ключ, значение]
```

### Когда использовать Map vs Object

```javascript
// Используйте Object когда:
// - Ключи — строки или символы
// - Нужен JSON (Object легко сериализуется)
// - Структура данных известна заранее

// Используйте Map когда:
// - Ключи могут быть любого типа
// - Важен порядок добавления
// - Часто добавляются/удаляются элементы
// - Нужно знать размер (.size)
```

---

## 9. Callback-функции и таймеры

### Callback-функции

Callback — это функция, передаваемая как аргумент другой функции.

```javascript
// Простой callback
const processArray = (arr, callback) => {
  const result = [];
  for (const item of arr) {
    result.push(callback(item));
  }
  return result;
};

const numbers = [1, 2, 3, 4, 5];
const doubled = processArray(numbers, n => n * 2);
// [2, 4, 6, 8, 10]

// Callback с несколькими параметрами
const processWithIndex = (arr, callback) => {
  const result = [];
  for (let i = 0; i < arr.length; i++) {
    result.push(callback(arr[i], i, arr));
  }
  return result;
};
```

### setTimeout

Выполняет функцию один раз через указанное время.

```javascript
// Простой setTimeout
setTimeout(() => {
  console.log('Выполнено через 1 секунду');
}, 1000);

// С сохранением ID для отмены
const timeoutId = setTimeout(() => {
  console.log('Это не выполнится');
}, 5000);

clearTimeout(timeoutId); // Отмена таймера

// Передача аргументов
setTimeout((name, greeting) => {
  console.log(`${greeting}, ${name}!`);
}, 1000, 'Alice', 'Hello');
```

### setInterval

Выполняет функцию периодически с указанным интервалом.

```javascript
// Простой setInterval
let count = 0;
const intervalId = setInterval(() => {
  count++;
  console.log(`Итерация ${count}`);
  
  if (count >= 5) {
    clearInterval(intervalId); // Остановка после 5 итераций
  }
}, 1000);

// Часы
const showTime = () => {
  const now = new Date();
  console.log(now.toLocaleTimeString());
};
setInterval(showTime, 1000);
```

### Практический пример: debounce

```javascript
const debounce = (fn, delay) => {
  let timeoutId;
  return (...args) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
};

// Использование: функция вызовется только через 300мс
// после последнего вызова
const search = debounce((query) => {
  console.log(`Поиск: ${query}`);
}, 300);

search('a');
search('ab');
search('abc'); // Только этот вызов выполнится
```

---

## 10. Замыкания

### Что такое замыкание

Замыкание — это функция, которая запоминает своё лексическое окружение даже после того, как внешняя функция завершила выполнение.

```javascript
const createCounter = () => {
  let count = 0; // Эта переменная "замкнута" внутри
  
  return () => {
    count++;
    return count;
  };
};

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3

// Каждый вызов createCounter создаёт новое замыкание
const counter2 = createCounter();
console.log(counter2()); // 1 (независимый счётчик)
```

### Практические примеры замыканий

```javascript
// Фабрика функций умножения
const createMultiplier = (factor) => {
  return (number) => number * factor;
};

const double = createMultiplier(2);
const triple = createMultiplier(3);

double(5);  // 10
triple(5);  // 15

// Приватные данные
const createBankAccount = (initialBalance) => {
  let balance = initialBalance; // Приватная переменная
  
  return {
    deposit: (amount) => {
      balance += amount;
      return balance;
    },
    withdraw: (amount) => {
      if (amount <= balance) {
        balance -= amount;
        return balance;
      }
      return 'Недостаточно средств';
    },
    getBalance: () => balance
  };
};

const account = createBankAccount(1000);
account.deposit(500);   // 1500
account.withdraw(200);  // 1300
account.getBalance();   // 1300
// account.balance — undefined (недоступна напрямую)

// Мемоизация
const memoize = (fn) => {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
};

const expensiveCalculation = memoize((n) => {
  console.log('Вычисление...');
  return n * n;
});

expensiveCalculation(5); // 'Вычисление...' → 25
expensiveCalculation(5); // 25 (из кэша, без вычисления)
```

---

## 11. Async/Await

### Промисы (Promise)

Promise — объект, представляющий результат асинхронной операции.

```javascript
// Создание Promise
const fetchData = () => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      const success = true;
      if (success) {
        resolve({ data: 'Данные получены' });
      } else {
        reject(new Error('Ошибка загрузки'));
      }
    }, 1000);
  });
};

// Использование с .then/.catch
fetchData()
  .then(result => console.log(result))
  .catch(error => console.error(error));
```

### async/await

Синтаксический сахар для работы с промисами.

```javascript
// Асинхронная функция
const getData = async () => {
  try {
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Ошибка:', error);
    throw error;
  }
};

// Вызов асинхронной функции
const main = async () => {
  const data = await getData();
  console.log(data);
};

main();
```

### Параллельное выполнение

```javascript
// Promise.all — ждёт все промисы
const fetchAll = async () => {
  const [users, posts, comments] = await Promise.all([
    fetch('/api/users').then(r => r.json()),
    fetch('/api/posts').then(r => r.json()),
    fetch('/api/comments').then(r => r.json())
  ]);
  return { users, posts, comments };
};

// Promise.race — возвращает первый завершившийся
const fastest = await Promise.race([
  fetch('/api/server1'),
  fetch('/api/server2')
]);

// Promise.allSettled — ждёт все, даже с ошибками
const results = await Promise.allSettled([
  fetch('/api/endpoint1'),
  fetch('/api/endpoint2'),
  Promise.reject('Error')
]);
// [{ status: 'fulfilled', value: ... }, { status: 'rejected', reason: ... }]
```

### Практический пример: загрузка курсов валют

```javascript
const fetchExchangeRates = async (baseCurrency = 'USD') => {
  try {
    const url = `https://api.exchangerate-api.com/v4/latest/${baseCurrency}`;
    const response = await fetch(url);
    
    if (!response.ok) {
      throw new Error(`HTTP error: ${response.status}`);
    }
    
    const data = await response.json();
    return data.rates;
  } catch (error) {
    console.error('Ошибка загрузки курсов:', error);
    return null;
  }
};

const convertCurrency = async (amount, from, to) => {
  const rates = await fetchExchangeRates(from);
  if (!rates || !rates[to]) {
    throw new Error('Неизвестная валюта');
  }
  return amount * rates[to];
};

// Использование
const result = await convertCurrency(100, 'USD', 'EUR');
console.log(`100 USD = ${result.toFixed(2)} EUR`);
```

---

## 12. Обработка ошибок

### try/catch/finally

```javascript
const divideNumbers = (a, b) => {
  try {
    if (b === 0) {
      throw new Error('Деление на ноль');
    }
    return a / b;
  } catch (error) {
    console.error('Ошибка:', error.message);
    return null;
  } finally {
    console.log('Операция завершена');
  }
};

divideNumbers(10, 2);  // 5, 'Операция завершена'
divideNumbers(10, 0);  // 'Ошибка: Деление на ноль', null, 'Операция завершена'
```

### Типы ошибок

```javascript
// Встроенные типы ошибок
throw new Error('Общая ошибка');
throw new TypeError('Неверный тип');
throw new RangeError('Значение вне диапазона');
throw new ReferenceError('Переменная не определена');
throw new SyntaxError('Синтаксическая ошибка');

// Создание собственного типа ошибки
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

const validateUser = (user) => {
  if (!user.name) {
    throw new ValidationError('Имя обязательно', 'name');
  }
  if (!user.email) {
    throw new ValidationError('Email обязателен', 'email');
  }
  return true;
};

try {
  validateUser({ name: 'Alice' });
} catch (error) {
  if (error instanceof ValidationError) {
    console.log(`Ошибка в поле ${error.field}: ${error.message}`);
  } else {
    throw error; // Пробрасываем неизвестные ошибки
  }
}
```

### Обработка ошибок в async/await

```javascript
// С try/catch
const fetchData = async () => {
  try {
    const response = await fetch('/api/data');
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    if (error.name === 'TypeError') {
      console.error('Сетевая ошибка');
    } else {
      console.error('Ошибка:', error.message);
    }
    return null;
  }
};

// Альтернатива: обёртка для обработки ошибок
const safeAsync = async (asyncFn) => {
  try {
    const result = await asyncFn();
    return [result, null];
  } catch (error) {
    return [null, error];
  }
};

// Использование
const [data, error] = await safeAsync(() => fetch('/api/data'));
if (error) {
  console.error(error);
} else {
  console.log(data);
}
```

---

## 13. Модульность

### CommonJS (CJS)

Используется в Node.js по умолчанию.

```javascript
// math.js — экспорт
const add = (a, b) => a + b;
const subtract = (a, b) => a - b;
const multiply = (a, b) => a * b;

module.exports = { add, subtract, multiply };
// или
module.exports.add = add;
exports.subtract = subtract;

// app.js — импорт
const math = require('./math');
console.log(math.add(2, 3)); // 5

// Деструктуризация при импорте
const { add, subtract } = require('./math');
console.log(add(2, 3)); // 5
```

### ES Modules (ESM)

Современный стандарт модулей JavaScript.

```javascript
// math.js — именованный экспорт
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;

// или в конце файла
const multiply = (a, b) => a * b;
const divide = (a, b) => a / b;
export { multiply, divide };

// Экспорт по умолчанию
const Calculator = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b
};
export default Calculator;

// app.js — именованный импорт
import { add, subtract } from './math.js';

// Импорт по умолчанию
import Calculator from './math.js';

// Импорт всего модуля
import * as math from './math.js';
console.log(math.add(2, 3));

// Переименование при импорте
import { add as sum } from './math.js';
```

### Настройка ESM в Node.js

```json
// package.json
{
  "type": "module"
}
```

Или используйте расширение `.mjs` для ES модулей.

---

## 14. Рекурсия

### Основы рекурсии

Рекурсия — это вызов функции самой себя.

```javascript
// Факториал
const factorial = (n) => {
  if (n <= 1) return 1;        // Базовый случай
  return n * factorial(n - 1); // Рекурсивный вызов
};

factorial(5); // 120 (5 * 4 * 3 * 2 * 1)

// Числа Фибоначчи
const fibonacci = (n) => {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
};

fibonacci(10); // 55
```

### Стек вызовов

```javascript
// Каждый рекурсивный вызов добавляется в стек
// При слишком глубокой рекурсии возникает Stack Overflow

// Пример: сумма чисел от 1 до n
const sum = (n) => {
  console.log(`sum(${n})`); // Для отладки
  if (n <= 0) return 0;
  return n + sum(n - 1);
};

sum(5);
// sum(5) → sum(4) → sum(3) → sum(2) → sum(1) → sum(0)
// Стек разворачивается: 0 + 1 + 2 + 3 + 4 + 5 = 15
```

### Хвостовая рекурсия

Оптимизация, когда рекурсивный вызов — последняя операция.

```javascript
// Обычная рекурсия (не хвостовая)
const factorial = (n) => {
  if (n <= 1) return 1;
  return n * factorial(n - 1); // Умножение ПОСЛЕ рекурсии
};

// Хвостовая рекурсия
const factorialTail = (n, acc = 1) => {
  if (n <= 1) return acc;
  return factorialTail(n - 1, n * acc); // Рекурсия — последняя операция
};

factorialTail(5); // 120
```

### Практический пример: обход дерева

```javascript
// Структура дерева
const fileSystem = {
  name: 'root',
  type: 'folder',
  children: [
    {
      name: 'src',
      type: 'folder',
      children: [
        { name: 'index.js', type: 'file' },
        { name: 'utils.js', type: 'file' }
      ]
    },
    { name: 'README.md', type: 'file' }
  ]
};

// Рекурсивный обход
const listFiles = (node, path = '') => {
  const currentPath = path ? `${path}/${node.name}` : node.name;
  
  if (node.type === 'file') {
    console.log(currentPath);
    return;
  }
  
  for (const child of node.children || []) {
    listFiles(child, currentPath);
  }
};

listFiles(fileSystem);
// root/src/index.js
// root/src/utils.js
// root/README.md
```

---

## Дополнительные ресурсы

### Ссылки автора курса

- **GitHub автора**: [github.com/tshemsedinov](https://github.com/tshemsedinov)
- **GitHub курса**: [github.com/HowProgrammingWorks](https://github.com/HowProgrammingWorks)
- **Telegram канал**: [t.me/HowProgrammingWorks](https://t.me/HowProgrammingWorks)
- **Telegram группа**: [t.me/metaedu](https://t.me/metaedu)
- **Оглавление курса**: [github.com/HowProgrammingWorks/Index](https://github.com/HowProgrammingWorks/Index)

### Рекомендации по дальнейшему изучению

1. **Фундаментальный курс** Тимура Шемсединова (более глубокое изучение)
2. **MDN Web Docs** — официальная документация JavaScript
3. **JavaScript.info** — современный учебник JavaScript
4. **TypeScript** — типизированный надмножество JavaScript

---

