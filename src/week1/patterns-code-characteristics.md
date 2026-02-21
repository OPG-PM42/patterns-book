# Паттерны: характеристики кода и стратегии оптимизации

## Обзор лекции

**Тема:** JavaScript, TypeScript — SOLID/SRP, SoC
**Цель:** Научиться писать код, который одновременно:
- Соответствует принципам SOLID и GRASP
- Оптимизирован для виртуальной машины V8
- Читабельный, надёжный и поддерживаемый

---

## 1. Введение: Зачем нужен обзор техник перед паттернами

### Основная идея

Прежде чем изучать паттерны проектирования, необходимо понять:
- Какие возможности JavaScript и TypeScript мы будем использовать
- Как разные парадигмы (процедурная, ООП, функциональная) применяются на практике
- Как совместить принципы качественного кода с оптимизацией для V8

### Подход к балансу характеристик кода

Мы стремимся сбалансировать:
- **Читаемость** - код легко понять
- **Надёжность** - меньше ошибок
- **Стабильность** - предсказуемое поведение
- **Модифицируемость** - легко вносить изменения
- **Тестируемость** - простота покрытия тестами
- **Производительность** - эффективное выполнение

---

## 2. Анализ плохого кода: 18 категорий проблем

### Исходная задача

**Задача:** Обработать сырой датасет товаров с неконсистентными данными и извлечь цены.

**Проблемы входных данных:**
- Объекты с полями в разном порядке
- Массивы вместо объектов
- Смешанные типы данных (string и number)
- Отсутствующие или переименованные поля (`price` vs `cost`)

---

### 2.1. Проблемы оптимизации V8

#### Проблема 1: Деоптимизация виртуальной машины

**Что:** Полиморфный код постоянно меняет форму объектов и скрытые классы.

**Почему плохо:**
- V8 оптимизирует код на основе предположений о структуре объектов
- Изменение формы объектов заставляет V8 отказываться от оптимизаций
- Производительность падает

**Ключевые концепции:**
- **Hidden Classes (скрытые классы)** - внутреннее представление структуры объектов в V8
- **Мегаморфный код** - код, работающий с более чем 4 формами объектов

#### Проблема 13: Изменение формы объектов

**Антипаттерны:**
```javascript
// Плохо: delete меняет форму объекта
const obj = { name: 'Item', price: 100 };
delete obj.price; // Форма изменилась!

// Плохо: добавление полей в разном порядке
const obj1 = { name: 'A', price: 10 };
const obj2 = { price: 20, name: 'B' }; // Другая форма!
```

**Решение:**
```javascript
// Хорошо: всегда одна и та же форма
const obj1 = { name: 'A', price: 10 };
const obj2 = { name: 'B', price: 20 };
```

#### Проблема 14: Дырявые массивы

**Что:** Массивы с пропусками (holes) создаются при удалении элементов.

```javascript
// Плохо
const arr = [1, 2, 3];
delete arr[1]; // [1, empty, 3] - дырявый массив
```

---

### 2.2. Проблемы читаемости и сложности

#### Проблема 2: Избыток if-ов (6 штук)

**Почему плохо:**
- Высокая когнитивная нагрузка
- Сложность тестирования (много ветвлений)
- Трудно отследить логику

**Решение:** Использовать меньше условий через декомпозицию и изоляцию проверок.

#### Проблема 3: Разнесённая проверка на массив

**Проблема:**
```javascript
// Плохо: проверка в одном месте
if (!Array.isArray(data)) {
  // обработка объекта
}
// ... много кода ...
// обработка массива в другом месте
```

**Почему плохо:** Глаза должны прыгать по коду, нарушается последовательность чтения.

#### Проблема 8: Дупликация кода

**Примеры дупликации:**
- Повторяющиеся проверки (`options.converter`)
- Похожие преобразования типов в разных местах

---

### 2.3. Проблемы производительности

#### Проблема 4: Неэффективная деструктуризация массива

```javascript
// Плохо: создаёт итератор
const [, value] = data; // Пропускаем первый, берём второй

// Хорошо: прямой доступ
const value = data[1];
```

**Почему плохо:** Деструктуризация массива создаёт итератор, что медленнее прямого доступа.

#### Проблема 12: Array как tuple

**Проблема:**
```javascript
// Плохо в JavaScript
const item = ['Book', 100]; // [name, price]
const [name, price] = item;
```

**Почему плохо:**
- Создание итератора при деструктуризации
- Непонятно, что в каком индексе
- В JavaScript объект предпочтительнее

**Решение:**
```javascript
// Хорошо
const item = { name: 'Book', price: 100 };
const { name, price } = item;
```

#### Проблема 14: Closures, модифицирующие внешний scope

```javascript
// Плохо
const out = [];
data.forEach(item => {
  out.push(process(item)); // Модификация внешней переменной
});

// Хорошо
const out = data.map(item => process(item));

// Ещё лучше (для V8)
const out = [];
for (let i = 0; i < data.length; i++) {
  out.push(process(data[i]));
}
```

---

### 2.4. Проблемы типизации

#### Проблема 5: Избыток Union типов

**Проблема:**
```typescript
// Плохо: сложный Union
type Item = { name: string; price?: number } | { name?: string; price: number };
function getPrice(data: Item | ItemData): number | undefined | NaN {
  // ...
}
```

**Цель:** Минимизировать Union типы через нормализацию данных.

#### Проблема 6: Выходной тип `number | undefined | NaN`

**Проблема:** NaN в JavaScript ведёт себя странно (`NaN !== NaN`).

**Решение:** Возвращать `number | undefined`, заменяя NaN на undefined.

```typescript
// Плохо
return parseInt(value); // Может вернуть NaN

// Хорошо
const parsed = parseInt(value);
return isNaN(parsed) ? undefined : parsed;
```

---

### 2.5. Проблемы архитектуры

#### Проблема 9: Слабые контракты (Weak Contracts)

**Суть:** Сырые данные проникают глубоко в бизнес-логику.

**Решение:** Изолировать сложность входных данных:
1. На входе нормализовать данные
2. В бизнес-логике работать только с консистентными данными

#### Проблема 10: Mixed Responsibility (Смешение ответственностей)

**Функция делает слишком много:**
- Конвертирует сырые данные
- Определяет типы
- Нормализует данные
- Преобразует массивы в объекты
- Извлекает конкретное поле (price)

**Принцип:** Single Responsibility Principle (SRP) - одна функция = одна ответственность.

#### Проблема 11: Inconsistent Return

**Проблема:** Функция возвращает разные типы в разных ветках.

```javascript
// Плохо: ESLint ругается
function getPrice(data) {
  if (!data) return undefined;
  if (typeof data === 'string') return NaN;
  return data.price;
}
```

---

### 2.6. Стилистические проблемы

#### Проблема 15: Неявное преобразование типов

**Антипаттерны:**
```javascript
// Приведение к строке
const str1 = '' + number;
const str2 = `${number}`;

// Приведение к числу
const num1 = +string;
const num2 = string * 1;
const num3 = string - 0;
const num4 = string / 1;
```

**Почему плохо:** Повышает когнитивную нагрузку, неочевидно.

**Решение:**
```javascript
// Явное преобразование
const str = String(number);
const num = Number(string);
const num2 = parseInt(string, 10);
const num3 = parseFloat(string);
```

#### Проблема 16: Сравнение с автоприведением типов

**Правило:** Всегда использовать строгое сравнение.

```javascript
// Плохо
if (value == null) { }
if (a != b) { }

// Хорошо
if (value === null) { }
if (a !== b) { }
```

#### Проблема 17: Последовательное присвоение

```javascript
// Плохо
let a = b = c = 0;

// Хорошо
let a = 0;
let b = 0;
let c = 0;
```

#### Проблема 18: Использование var

**Правило:** Использовать `const` по умолчанию, `let` только при необходимости.

```javascript
// Плохо
var x = 10;

// Хорошо
const x = 10;

// Если нужна мутация
let counter = 0;
counter++; // Но не менять тип!
```

**Важно:** Если переменная объявлена через `let`:
- Не менять её тип (`number` → `string`)
- Стараться не смешивать `float` и `integer`

---

## 3. Рефакторинг №1: Процедурный стиль с разделением ответственности

### Основная идея

Разделить функцию на две:
- `normalizeItem()` - нормализует данные
- `getPrice()` - извлекает цену из нормализованных данных

### Ключевые улучшения

#### 1. Разделение ответственностей (SRP)

```javascript
// Плохо: одна функция делает всё
function getPrice(data, options) {
  // Проверки типов
  // Нормализация
  // Извлечение price
  // Конвертация типов
}

// Хорошо: две функции с чёткими ответственностями
function normalizeItem(data, options = {}) {
  // Только нормализация данных
  const isTuple = Array.isArray(data);
  const obj = isTuple ? { name: data[0], price: data[1] } : data;
  const { price, cost } = obj;
  const value = price ?? cost;

  const { converter } = options;
  const normalizedValue =
    typeof value === 'string' && converter
      ? converter(value)
      : parseFloat(value);

  return { name: obj.name, price: normalizedValue };
}

function getPrice(item) {
  // Только извлечение price
  return item.price;
}
```

#### 2. Использование const вместо let

```javascript
// Плохо
let price;
if (typeof value === 'string') {
  price = parseFloat(value);
} else {
  price = value;
}

// Хорошо
const price = typeof value === 'string' ? parseFloat(value) : value;
```

#### 3. Уменьшение if-ов

**Было:** 6 if-ов
**Стало:** 3 тернарных оператора

Тернарные операторы проще читать, когда они однострочные.

#### 4. Линейный код

Код стал последовательным, без прыжков взгляда:
```javascript
const isTuple = Array.isArray(data);
const obj = isTuple ? { name: data[0], price: data[1] } : data;
const { price, cost } = obj;
const value = price ?? cost;
// ... и так далее
```

#### 5. Улучшение типизации

**Было:**
```typescript
type Item = { name?: string; price?: number } | ItemData;
```

**Стало:**
```typescript
type RawItemTuple = [string, number | string];
type RawItemObject = { name: string; price?: number; cost?: number };
type RawItem = RawItemTuple | RawItemObject;

type NormalizedItem = { name: string; price: number };

function normalizeItem(data: RawItem, options?: Options): NormalizedItem {
  // ...
}
```

**Преимущества:**
- Чёткое разделение raw и normalized данных
- Меньше Union типов во внутренней логике
- Consistent return type

### Результаты оптимизации

**Характеристики кода:**
- ✅ Легче тестировать
- ✅ Легче читать
- ✅ Нет дупликации
- ✅ Меньше if-ов (3 тернарных вместо 6 if-ов)
- ✅ Изоляция проверок от бизнес-логики
- ✅ Применён SRP (Single Responsibility Principle)
- ✅ Применён SoC (Separation of Concerns)
- ✅ Consistent return
- ✅ Готов к inlining (V8 может встроить функции)

---

## 4. Рефакторинг №2: Именованные параметры

### Техника именованных параметров

```javascript
// Было: позиционные параметры
function normalizeItem(data, options) {
  // ...
}
normalizeItem(item, { converter: parseFloat });

// Стало: именованные параметры
function normalizeItem({ data, options = {} }) {
  // ...
}
normalizeItem({ data: item, options: { converter: parseFloat } });
```

### Когда использовать

**Плюсы именованных параметров:**
- Читаемость - видно, что передаётся
- Гибкость - можно опускать/переставлять аргументы
- Самодокументирование кода

**Минусы:**
- Создаёт дополнительные обёртки (деструктуризация)
- Немного медленнее для V8

### Рекомендация

- **Высокоуровневый код:** использовать именованные параметры (приоритет читаемости)
- **Низкоуровневый код:** использовать позиционные параметры (приоритет производительности)

---

## 5. Рефакторинг №3: Классы и статические фабрики

### Зачем использовать классы

- Некоторые паттерны естественнее выражаются через ООП
- Инкапсуляция данных
- Использование `instanceof` для проверки типов
- Мультипарадигменный подход

### Паттерн: Статические фабрики

**Структура класса:**

```javascript
class Item {
  constructor(name, price, options = {}) {
    const { converter } = options;
    this.name = name;
    this.price =
      typeof price === 'string' && converter
        ? converter(price)
        : parseFloat(price);
  }

  // Фабрика для объектов
  static fromStruct({ name, price, cost, ...options }) {
    return new Item(name, price ?? cost, options);
  }

  // Фабрика для массивов (tuple)
  static fromTuple([name, price], options) {
    return new Item(name, price, options);
  }

  // Общая фабрика (Strategy выбора)
  static fromData(data, options = {}) {
    const factory = Array.isArray(data) ? Item.fromTuple : Item.fromStruct;
    return factory(data, options);
  }
}
```

### Использование

```javascript
// Использование
const items = rawData.map(data => Item.fromData(data, { converter: parseFloat }));
const prices = items.map(item => item.price);
```

### Преимущества подхода

1. **Декомпозиция:** Каждая фабрика имеет одну ответственность
   - `fromStruct` - разобрать объект
   - `fromTuple` - разобрать массив
   - `fromData` - выбрать нужную фабрику

2. **Strategy pattern:** `fromData` выбирает стратегию на основе типа данных

3. **Приватность:** Можно сделать поля приватными с геттерами

```javascript
class Item {
  #price; // Приватное поле

  constructor(name, price) {
    this.name = name;
    this.#price = price;
  }

  get price() {
    return this.#price;
  }
}
```

### Подход к Strategy

```javascript
// Выбор фабрики из переменной
const factory = Array.isArray(data) ? Item.fromTuple : Item.fromStruct;
return factory(data, options);
```

Это начало паттерна Strategy - выбор алгоритма из коллекции на основе ключа/условия.

---

## 6. Рефакторинг №4: Функциональный подход с замыканиями

### Идея

Вместо хранения данных в полях объекта использовать замыкания (closures).

### Реализация

```javascript
// Фабрика объектов с замыканиями
function item(name, price, options = {}) {
  const { converter } = options;
  const normalizedPrice =
    typeof price === 'string' && converter
      ? converter(price)
      : parseFloat(price);

  // Данные в замыкании
  const finalName = name;
  const finalPrice = normalizedPrice;

  // Возвращаем объект с методами доступа
  return {
    getName: () => finalName,
    getPrice: () => finalPrice
  };
}

// Фабрики остаются похожими
function fromStruct({ name, price, cost, ...options }) {
  return item(name, price ?? cost, options);
}

function fromTuple([name, price], options) {
  return item(name, price, options);
}

function fromData(data, options = {}) {
  const factory = Array.isArray(data) ? fromTuple : fromStruct;
  return factory(data, options);
}
```

### Использование

```javascript
const items = rawData.map(data => fromData(data));
const prices = items.map(item => item.getPrice());
```

### Преимущества

1. **Иммутабельность:** Данные нельзя модифицировать извне
2. **Инкапсуляция:** Данные скрыты в замыкании
3. **Защита:** Нет прямого доступа к переменным

### Результат

```javascript
console.log(item);
// { getName: [Function], getPrice: [Function] }
// Сами name и price не видны
```

---

## 7. Рефакторинг №5: Процедурный код (финальная версия)

### Философия процедурного стиля

**Почему процедурный?**
- Баланс между ФП и императивным стилем
- Использование чистых функций
- Не модифицируем state наружу, делаем return
- Допускаются переменные (в отличие от чистого ФП)

**Отличие от функционального:**
- В ФП не должно быть переменных - только expressions
- В ФП функции очень маленькие (по 1 строке)
- В процедурном допустимы if, for, переменные - более читаемо

### Принципы

1. **Чистые функции** - не модифицируют внешний state
2. **Const по умолчанию** - переменные только когда нужно
3. **Простые циклы** - `for` вместо рекурсии
4. **Структуры данных** - возвращаем объекты, а не храним в замыканиях

### Реализация

```javascript
// Всегда возвращаем объект одинаковой формы
function normalizeItem(data, options = {}) {
  const isTuple = Array.isArray(data);
  const obj = isTuple ? { name: data[0], price: data[1] } : data;
  const { name, price, cost } = obj;
  const value = price ?? cost;

  const { converter } = options;
  const normalizedPrice =
    typeof value === 'string' && converter
      ? converter(value)
      : parseFloat(value);

  // Всегда одинаковая форма: { name: string, price: number }
  return { name, price: normalizedPrice };
}

function getPrice(item) {
  return item.price;
}
```

### Результат

```javascript
const normalized = rawData.map(data => normalizeItem(data));
const prices = normalized.map(getPrice);

console.table(normalized);
// Все объекты имеют одинаковую форму { name, price }
```

### Итоговые характеристики

✅ **Читаемость:** последовательный код
✅ **Производительность:** одна форма объектов → V8 оптимизирует
✅ **Тестируемость:** чистые функции легко тестировать
✅ **Модифицируемость:** понятная структура
✅ **Надёжность:** consistent types

---

## 8. Ключевые принципы и выводы

### Принцип Single Responsibility (SRP)

**Определение:** Каждая функция/класс должны иметь одну ответственность.

**Пример из лекции:**
- `normalizeItem` - только нормализация
- `getPrice` - только извлечение цены
- `fromStruct` - только разбор объекта
- `fromTuple` - только разбор массива

### Принцип Separation of Concerns (SoC)

**Определение:** Разделение задач - каждый узел системы выполняет одну задачу.

**Применение:**
- Изолировать обработку сырых данных от бизнес-логики
- Разделить raw и normalized данные
- Логика проверок не проникает в бизнес-логику

### Weak Contracts vs Strong Contracts

**Weak Contracts (плохо):**
```javascript
// Бизнес-логика работает с сырыми данными
function businessLogic(rawData) {
  if (Array.isArray(rawData)) { /* ... */ }
  if (typeof rawData.price === 'string') { /* ... */ }
  // Проверки размазаны по всему коду
}
```

**Strong Contracts (хорошо):**
```javascript
// Нормализация на входе
const normalized = rawData.map(normalizeItem);

// Бизнес-логика работает с чистыми данными
function businessLogic(normalizedData) {
  // Никаких проверок - данные уже консистентны
  return normalizedData.map(item => item.price * 1.2);
}
```

### Оптимизация для V8

**Правила:**
1. **Одна форма объекта** - всегда { name, price } в одном порядке
2. **Не менять форму** - не использовать delete, не добавлять поля
3. **Не смешивать типы** - в переменной всегда один тип
4. **Избегать дырявых массивов** - не delete из массивов
5. **Const > let > var** - предсказуемость для оптимизатора
6. **Меньше 4 форм** - иначе мегаморфный код

### Баланс характеристик

Код должен быть:
- ✅ Читаемый
- ✅ Надёжный
- ✅ Стабильный
- ✅ Понятный
- ✅ Модифицируемый
- ✅ Тестируемый
- ✅ Достаточно оптимизированный

**Не надо:** Жертвовать всем ради максимальной производительности или максимальной красоты.

---

## 9. Практические рекомендации

### Что использовать

#### ✅ Использовать всегда

- `const` по умолчанию
- Строгое сравнение (`===`, `!==`)
- Явное преобразование типов (`Number()`, `String()`)
- Чистые функции
- Consistent object shapes
- Разделение raw и normalized данных

#### ⚠️ Использовать осторожно

- `let` - только если действительно нужна мутация
- Тернарные операторы - только однострочные
- Именованные параметры - в высокоуровневом коде
- Классы - когда подходят паттерны ООП

#### ❌ Избегать

- `var`
- `==` и `!=`
- `delete` на объектах
- Неявное приведение типов (`+str`, `str * 1`)
- Последовательное присвоение (`a = b = c`)
- Модификация аргументов функции
- Closures, модифицирующие внешний scope
- Деструктуризация массивов как tuples
- Массивы с элементами разных типов
- Изменение типа переменной

### Когда использовать разные стили

| Стиль | Когда использовать |
|-------|-------------------|
| **Процедурный** | Основной стиль для прикладного кода. Баланс читаемости и производительности. |
| **ООП (классы)** | Паттерны, требующие инкапсуляции и полиморфизма. Статические фабрики. |
| **Функциональный** | Когда нужна иммутабельность и защита данных. Композиция функций. |

### Subset JavaScript

**Философия:** Использовать подмножество возможностей JavaScript, а не всю мощь языка.

**Почему:**
- Повышает предсказуемость
- Упрощает чтение кода
- Помогает V8 оптимизировать
- Снижает количество ошибок

---

## 10. Типичные ошибки и как их избежать

### Ошибка 1: Переусложнение

```javascript
// Плохо: использование всех возможностей JS
const price = options?.converter?.(value) ?? +value || parseFloat(value);

// Хорошо: простой и понятный код
const price = options.converter
  ? options.converter(value)
  : parseFloat(value);
```

### Ошибка 2: Игнорирование формы объектов

```javascript
// Плохо: разные формы
const items = [
  { name: 'A', price: 10 },
  { price: 20, name: 'B' }, // Другой порядок полей!
  { name: 'C', cost: 30 }   // Другое поле!
];

// Хорошо: всегда одна форма
const normalizeItem = (raw) => ({
  name: raw.name,
  price: raw.price ?? raw.cost
});
const items = rawData.map(normalizeItem);
```

### Ошибка 3: Смешение ответственностей

```javascript
// Плохо: функция делает всё
function processAndSave(rawData) {
  // Валидация
  // Нормализация
  // Бизнес-логика
  // Сохранение в БД
}

// Хорошо: разделение
function validate(data) { /* ... */ }
function normalize(data) { /* ... */ }
function process(data) { /* ... */ }
function save(data) { /* ... */ }

// Композиция
const result = save(process(normalize(validate(rawData))));
```

### Ошибка 4: Проверки в бизнес-логике

```javascript
// Плохо
function calculateDiscount(item) {
  if (!item.price) return 0;
  if (typeof item.price === 'string') {
    item.price = parseFloat(item.price);
  }
  return item.price * 0.1;
}

// Хорошо: нормализация на входе
function calculateDiscount(item) {
  // item.price гарантированно number
  return item.price * 0.1;
}
```

---

## 11. Связь с паттернами проектирования

Эта лекция закладывает основу для изучения паттернов:

### Strategy (начало в Рефакторинге №3)

```javascript
// Выбор стратегии на основе данных
const factory = Array.isArray(data) ? fromTuple : fromStruct;
return factory(data);
```

### Factory Method

```javascript
// Статические фабрики - разновидность Factory
static fromStruct(data) { /* ... */ }
static fromTuple(data) { /* ... */ }
static fromData(data) { /* ... */ }
```

### SOLID принципы

- **S** - Single Responsibility (разделение normalizeItem и getPrice)
- **O** - Open/Closed (фабрики можно расширять)
- **L** - Liskov Substitution (все фабрики возвращают совместимые объекты)
- **I** - Interface Segregation (минимальные интерфейсы)
- **D** - Dependency Inversion (options injection)

---

## 12. Резюме

### Главные выводы

1. **Баланс важнее крайностей** - нужен компромисс между производительностью и читаемостью

2. **Разделяй и властвуй** - декомпозиция улучшает все характеристики кода

3. **Изолируй сложность** - проверки и нормализация на входе, чистая логика внутри

4. **Консистентность данных** - одна форма объектов, predictable types

5. **Subset > Full power** - использовать простое подмножество JS, а не все возможности

### Практический чеклист

При написании кода проверяйте:

- [ ] Используется `const` везде, где возможно
- [ ] Строгое сравнение (`===`, `!==`)
- [ ] Одна ответственность на функцию
- [ ] Нет дупликации кода
- [ ] Сырые данные нормализованы на входе
- [ ] Объекты имеют одинаковую форму
- [ ] Нет изменения формы объектов
- [ ] Явное преобразование типов
- [ ] Чистые функции (не модифицируют внешний state)
- [ ] Consistent return types

### Дальнейшее изучение

**Для углубления:**
- Оптимизации V8 (скрытые классы, inline caching)
- SOLID и GRASP принципы
- Паттерны проектирования на JavaScript/TypeScript
- Функциональное программирование
- TypeScript advanced types

---

## Ссылки и материалы

**Из лекции упоминались:**
- Блог Вячеслава Егорова (разработчик V8) - про оптимизации виртуальной машины
- Статьи про скрытые классы и inline caching
- Дополнительные видео и стримы по паттернам

**Код из лекции:**
- Исходники всех примеров рефакторинга
- Задачи для самостоятельной работы
