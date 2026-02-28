# Лекция 1. Events, EventTarget, EventEmitter в JavaScript

## Введение

События (Events) — это один из фундаментальных контрактов асинхронности в JavaScript.
Если колбэки представляют собой одну функцию, которую кто-то вызовет позже, то события
позволяют хранить **коллекцию таких колбэков** и вызывать их при наступлении определённого
именованного момента.

В экосистеме JavaScript существуют две родственные, но различные реализации этой идеи:

- `EventTarget` — родилась в браузере как часть Web API, затем была перенесена в Node.js
  (доступна без флагов начиная с Node.js 19, экспериментально — с версий 16–17)
- `EventEmitter` — нативная реализация Node.js, экспортируется из модуля `node:events`

Обе абстракции совместимы **по концепции**, но **несовместимы по API**.

---

## 1.1 EventTarget — браузерная абстракция

`EventTarget` является частью спецификации Web API. Её ключевая особенность: один
обработчик на одно событие. Если один и тот же обработчик добавлен несколько раз,
в коллекции он окажется только один раз.

```javascript
// Пример 1. Базовое использование EventTarget

// Создаём экземпляр EventTarget
const target = new EventTarget();

// Добавляем обработчик события 'data' через addEventListener
// Второй аргумент — функция-обработчик, принимает объект Event
target.addEventListener('data', (event) => {
  // event.detail содержит переданные данные (для CustomEvent)
  console.log('Получено событие data:', event.detail);
});

// Создаём объект события с помощью CustomEvent
// CustomEvent позволяет передавать произвольные данные через поле detail
const event = new CustomEvent('data', { detail: { value: 42 } });

// Отправляем событие — вызовутся все зарегистрированные обработчики
target.dispatchEvent(event);
// Вывод: Получено событие data: { value: 42 }
```

```javascript
// Пример 2. Дедупликация обработчиков в EventTarget
// EventTarget НЕ добавляет дублирующиеся обработчики

const target = new EventTarget();

function handler(event) {
  console.log('Обработчик вызван, значение:', event.detail.value);
}

// Добавляем один и тот же обработчик три раза
target.addEventListener('click', handler);
target.addEventListener('click', handler);
target.addEventListener('click', handler);

// Несмотря на три вызова addEventListener, обработчик сохранён один раз
// Поэтому событие обработается ОДИН раз
target.dispatchEvent(new CustomEvent('click', { detail: { value: 1 } }));
// Вывод: Обработчик вызван, значение: 1   (только один раз!)
```

```javascript
// Пример 3. Отписка от события через removeEventListener

const target = new EventTarget();

function onData(event) {
  console.log('data:', event.detail);
}

// Подписываемся
target.addEventListener('message', onData);

// Отписываемся — передаём ту же функцию-ссылку
target.removeEventListener('message', onData);

// После удаления обработчика событие никем не будет обработано
target.dispatchEvent(new CustomEvent('message', { detail: 'привет' }));
// Вывод: (пусто — обработчик был удалён)
```

```javascript
// Пример 4. Обработка ошибок в EventTarget
// ВАЖНО: EventTarget НЕ падает при отсутствии обработчика события 'error'
// В отличие от EventEmitter, событие 'error' для EventTarget — обычное именованное событие

const target = new EventTarget();

// Если у нас есть обработчик ошибки — ошибка обрабатывается
target.addEventListener('error', (event) => {
  console.log('Ошибка перехвачена:', event.detail.message);
});

target.dispatchEvent(new CustomEvent('error', { detail: new Error('что-то пошло не так') }));
// Вывод: Ошибка перехвачена: что-то пошло не так

// Теперь уберём обработчик и снова отправим ошибку
target.removeEventListener('error', (event) => {}); // другая ссылка — не удалит!
// EventTarget просто проигнорирует событие без обработчика — процесс НЕ упадёт
target.dispatchEvent(new CustomEvent('error', { detail: new Error('ещё одна ошибка') }));
// Вывод: (пусто — но Node.js не падает!)
```

---

## 1.2 EventEmitter — нативная абстракция Node.js

`EventEmitter` из модуля `node:events` имеет другой интерфейс и другую семантику.
Ключевые отличия от `EventTarget`:

- Позволяет добавлять **несколько экземпляров одного обработчика** (нет дедупликации)
- При отсутствии обработчика события `'error'` **процесс Node.js аварийно завершится**
- Данные передаются напрямую как аргументы, а не через объект `Event`

```javascript
// Пример 5. Базовое использование EventEmitter
import { EventEmitter } from 'node:events';

// Создаём экземпляр EventEmitter
const emitter = new EventEmitter();

// Подписываемся на событие 'data' через метод on
// Первый аргумент — имя события, второй — колбэк
emitter.on('data', (payload) => {
  // Данные приходят напрямую в аргументы колбэка, без обёртки в Event
  console.log('Получены данные:', payload);
});

// Отправляем событие через emit
// Данные передаём как дополнительные аргументы после имени события
emitter.emit('data', { a: 5 });
// Вывод: Получены данные: { a: 5 }
```

```javascript
// Пример 6. Дублирование обработчиков в EventEmitter
// В отличие от EventTarget, EventEmitter ДОПУСКАЕТ дубликаты

import { EventEmitter } from 'node:events';

const emitter = new EventEmitter();

function handler(value) {
  console.log('Обработчик:', value);
}

// Добавляем один и тот же обработчик три раза
emitter.on('name', handler);
emitter.on('name', handler);
emitter.on('name', handler);

// Все три экземпляра будут вызваны
emitter.emit('name', { a: 5 });
// Вывод:
// Обработчик: { a: 5 }
// Обработчик: { a: 5 }
// Обработчик: { a: 5 }

// Удаляем один экземпляр из трёх
emitter.removeListener('name', handler);

emitter.emit('name', { a: 10 });
// Вывод:
// Обработчик: { a: 10 }
// Обработчик: { a: 10 }   (осталось два)
```

```javascript
// Пример 7. Обработка ошибок в EventEmitter
// КРИТИЧНО: если нет обработчика 'error', Node.js аварийно завершает процесс

import { EventEmitter } from 'node:events';

const emitter = new EventEmitter();

// Правильный способ — всегда подписываться на 'error'
emitter.on('error', (err) => {
  console.error('Перехвачена ошибка:', err.message);
});

// Теперь ошибка будет обработана корректно
emitter.emit('error', new Error('что-то пошло не так'));
// Вывод: Перехвачена ошибка: что-то пошло не так

// ОПАСНЫЙ вариант (закомментирован, чтобы не падал процесс):
// const dangerousEmitter = new EventEmitter();
// dangerousEmitter.emit('error', new Error('фатальная ошибка'));
// Результат: процесс Node.js упадёт с необработанным исключением!
```

```javascript
// Пример 8. Метод once — одноразовая подписка
// once() автоматически отписывается после первого вызова

import { EventEmitter } from 'node:events';

const emitter = new EventEmitter();

// Обработчик сработает только один раз, затем будет автоматически удалён
emitter.once('connect', (info) => {
  console.log('Подключение установлено:', info.host);
});

emitter.emit('connect', { host: 'localhost' });
// Вывод: Подключение установлено: localhost

emitter.emit('connect', { host: 'example.com' });
// Вывод: (пусто — обработчик уже был удалён после первого вызова)
```

```javascript
// Пример 9. Функция events.once — ожидание события как Promise
// Позволяет await-ить наступление события

import { EventEmitter, once } from 'node:events';

const emitter = new EventEmitter();

// Асинхронная функция, ожидающая события
async function waitForData() {
  // once() из 'node:events' возвращает Promise
  // Он разрешится когда произойдёт событие 'data'
  const [payload] = await once(emitter, 'data');
  console.log('Дождались события:', payload);
}

// Запускаем ожидание
waitForData();

// Через 100 мс генерируем событие
setTimeout(() => {
  emitter.emit('data', { result: 'готово' });
}, 100);

// Вывод (через ~100 мс): Дождались события: { result: 'готово' }
```

---

## 1.3 Наивная реализация EventEmitter

Понять, как работает EventEmitter изнутри, помогает минималистичная реализация.
Она показывает главную идею: **коллекция массивов функций, индексированных по имени события**.

```javascript
// Пример 10. Наивная реализация EventEmitter с нуля

class SimpleEmitter {
  constructor() {
    // Хранилище обработчиков: { 'eventName': [fn1, fn2, ...] }
    // Можно использовать Map вместо объекта для большей надёжности
    this.events = {};
  }

  // Добавляем обработчик по имени события
  on(name, fn) {
    if (this.events[name]) {
      // Если событие уже зарегистрировано — добавляем ещё один обработчик
      this.events[name].push(fn);
    } else {
      // Создаём новый массив с первым обработчиком
      this.events[name] = [fn];
    }
    return this; // возвращаем this для цепочки вызовов
  }

  // Удаляем один экземпляр обработчика
  removeListener(name, fn) {
    if (!this.events[name]) return this;
    // Находим первый совпадающий обработчик и удаляем его
    const index = this.events[name].indexOf(fn);
    if (index !== -1) {
      this.events[name].splice(index, 1);
    }
    return this;
  }

  // Генерируем событие — вызываем все обработчики
  emit(name, ...args) {
    // В реальной реализации здесь должна быть проверка:
    // если name === 'error' и нет обработчика — бросить ошибку
    if (!this.events[name]) return false;

    // Копируем массив, чтобы безопасно итерироваться
    // даже если обработчики будут добавляться/удаляться во время вызова
    const listeners = [...this.events[name]];

    for (const listener of listeners) {
      // Распаковываем аргументы и вызываем каждый обработчик
      listener(...args);
    }
    return true;
  }
}

// Демонстрация работы
const emitter = new SimpleEmitter();

emitter.on('greet', (name) => console.log(`Привет, ${name}!`));
emitter.on('greet', (name) => console.log(`Hello, ${name}!`));

emitter.emit('greet', 'Мир');
// Вывод:
// Привет, Мир!
// Hello, Мир!
```

---

## 1.4 Сравнительная таблица EventTarget и EventEmitter

| Характеристика | `EventTarget` (Web API) | `EventEmitter` (Node.js) |
|----------------|------------------------|--------------------------|
| Метод подписки | `addEventListener(name, fn)` | `on(name, fn)` |
| Метод отписки | `removeEventListener(name, fn)` | `removeListener(name, fn)` |
| Генерация события | `dispatchEvent(new CustomEvent(...))` | `emit(name, ...args)` |
| Дедупликация | Да (один обработчик на fn-ссылку) | Нет (допускает дубликаты) |
| Ошибка без обработчика `error` | Игнорируется | Процесс аварийно завершается |
| Передача данных | Через `event.detail` | Напрямую как аргументы |
| Доступность в Node.js | С версии 19 (без флагов) | Всегда |
