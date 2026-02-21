# Структура приложений Node.js: модули, пакеты и зависимости

## Обзор

Эта лекция охватывает фундаментальные концепции модульной системы Node.js, включая CommonJS и ECMAScript модули, механизмы загрузки и кэширования модулей, работу с пакетами и создание собственной системы модульности. Мы рассмотрим внутреннее устройство модульной системы Node.js и научимся создавать изолированные окружения выполнения (sandboxes).

---

## Основы модульной системы CommonJS

### Структура типичного модуля CommonJS

В Node.js каждый файл является отдельным модулем. Модули используют специальные идентификаторы для экспорта и импорта функциональности.

#### Пример базового модуля (file1-export.js)

```javascript
// Определяем функции и переменные
const myFunction = () => {
  console.log('Функция из модуля');
};

const MyClass = class {
  constructor(name) {
    this.name = name;
  }
};

const collection = new Map();

// Экспортируем через module.exports
module.exports = {
  myFunction,
  MyClass,
  collection
};
```

**Важно**: Идентификатор `module` не объявлен в коде явно - он внедряется загрузчиком Node.js через механизм `vm.createScript`.

### Встроенные идентификаторы модулей

Каждый CommonJS модуль имеет доступ к следующим встроенным идентификаторам:

- `module` - объект текущего модуля
- `exports` - ссылка на `module.exports`
- `require` - функция для загрузки модулей
- `__filename` - абсолютный путь к текущему файлу
- `__dirname` - абсолютный путь к директории файла

Все эти идентификаторы внедряются извне загрузчиком Node.js.

---

## Загрузка модулей через require

### Загрузка встроенных модулей Node.js

#### Традиционный способ

```javascript
const fs = require('fs');
const events = require('events');
const timers = require('timers');
```

#### Современный способ (с Node.js 16+)

```javascript
// Использование префикса node: для явного указания встроенного модуля
const fs = require('node:fs');
const events = require('node:events');
const timers = require('node:timers/promises'); // подмодуль с промисами
```

**Преимущества префикса `node:`:**
- Защита от конфликтов с npm-пакетами с такими же именами
- Явное указание, что загружается встроенный модуль Node.js
- Рекомендуется использовать всегда для встроенных модулей

### Загрузка npm-пакетов

```javascript
// Загрузка пакета из node_modules
const WebSocket = require('ws');
```

### Загрузка локальных модулей

```javascript
// Относительный путь
const myModule = require('./file1-export');

// Абсолютный путь
const myModule = require('/absolute/path/to/module');
```

---

## Функция require и её свойства

### require.resolve()

Преобразует относительный путь к модулю в абсолютный без загрузки модуля.

```javascript
const modulePath = require.resolve('./file1-export');
console.log(modulePath); // Выведет абсолютный путь к файлу
```

### require.cache

Объект-коллекция всех загруженных и закэшированных модулей.

```javascript
// Просмотр информации о кэшированном модуле
const modulePath = require.resolve('./file1-export');
const cachedModule = require.cache[modulePath];

console.log(cachedModule);
// {
//   id: '/абсолютный/путь/к/модулю',
//   path: '/абсолютный/путь/к/директории',
//   exports: { /* экспортированное содержимое */ },
//   filename: '/абсолютный/путь/к/модулю',
//   loaded: true,
//   children: [ /* зависимости этого модуля */ ],
//   paths: [ /* пути поиска модулей */ ]
// }
```

---

## Кэширование модулей и паттерн Singleton

### Принцип работы кэша

**Ключевая особенность**: Когда модуль загружается через `require`, он загружается только один раз и затем кэшируется. Все последующие вызовы `require` возвращают один и тот же объект (ссылку).

#### Демонстрация синглтона

**Модуль file1-export.js:**
```javascript
const collection = new Map();

module.exports = {
  collection
};
```

**Файл 5-singleton.js:**
```javascript
const module1 = require('./file1-export');

// Модифицируем коллекцию
module1.collection.set('key', 'value');

console.log(module1.collection.get('key')); // 'value'
```

**Файл 6-global.js:**
```javascript
// Сначала загружаем файл, который модифицирует модуль
require('./5-singleton');

// Теперь загружаем тот же модуль
const module2 = require('./file1-export');

console.log(module2.collection.get('key')); // 'value' - изменения сохранились!
```

**Вывод**: `module1.collection` и `module2.collection` указывают на один и тот же объект Map в памяти.

### Опасности мутабельности

Из-за кэширования возможны следующие проблемы:

1. **Непредсказуемые изменения состояния**
   - Один модуль может изменить экспортированный объект
   - Эти изменения будут видны во всех модулях, которые его импортируют

2. **Сложность отладки**
   - Трудно отследить, кто и когда изменил состояние
   - Особенно проблематично с транзитивными зависимостями

#### Пример: Monkey Patching встроенных модулей

```javascript
const fs = require('node:fs');

// Сохраняем оригинальную функцию
const originalReadFile = fs.readFile;

// Подменяем функцию
fs.readFile = function(path, options, callback) {
  console.log('Перехвачено чтение файла:', path);

  // Вызываем оригинальную функцию
  return originalReadFile.call(this, path, options, (...args) => {
    console.log('Файл прочитан, размер буфера:', args[1]?.length);
    callback(...args);
  });
};

// Теперь в любом месте приложения, где используется fs.readFile,
// будет выполняться наша обёртка!
```

**Опасность**: Любая зависимость может модифицировать встроенные модули Node.js или глобальные прототипы, что приведёт к:
- Непредсказуемому поведению
- Сложности в аудите безопасности
- Проблемам с отладкой

### Очистка кэша

Можно вручную удалить модуль из кэша для повторной загрузки:

```javascript
const module1 = require('./file1-export');

// Удаляем из кэша
const modulePath = require.resolve('./file1-export');
delete require.cache[modulePath];

// Загружаем снова - будет создан новый объект
const module2 = require('./file1-export');

console.log(module1 === module2); // false - разные объекты!
```

**Ограничение**: Это не защищает полностью от модификаций, так как:
- Другие зависимости могли загрузить модуль до или после очистки
- Невозможно контролировать момент загрузки во всех зависимостях

---

## Алгоритм поиска модулей

### Последовательность поиска для require()

Когда вызывается `require('module-name')` (без префикса `./` или `/`), Node.js ищет модуль в следующих директориях:

```javascript
// Пример массива paths из кэшированного модуля
[
  '/path/to/current/directory/node_modules',
  '/path/to/parent/directory/node_modules',
  '/path/to/grandparent/directory/node_modules',
  // ... до корня файловой системы
  '/node_modules'
]
```

**Процесс поиска:**
1. Текущая директория → `./node_modules`
2. Родительская директория → `../node_modules`
3. Директория выше → `../../node_modules`
4. Продолжается до корня файловой системы

### Порядок разрешения расширений файлов

Если расширение не указано, Node.js пробует в следующем порядке:

1. `.js` - JavaScript-файл (CommonJS)
2. `.json` - JSON-файл
3. `.node` - нативное расширение C++
4. `.mjs` - ECMAScript-модуль (если используется ECMAScript modules)

**Рекомендация**: Всегда указывайте расширение явно для ускорения загрузки и избежания неоднозначности.

---

## ECMAScript модули (ESM)

### Отличия от CommonJS

**Синтаксис экспорта:**

```javascript
// CommonJS
module.exports = {
  myFunction,
  MyClass
};

// ECMAScript модули
export {
  myFunction,
  MyClass
};
```

**Синтаксис импорта:**

```javascript
// CommonJS
const { myFunction, MyClass } = require('./module');

// ECMAScript модули
import { myFunction, MyClass } from './module.mjs';
```

### Расширения файлов

- `.mjs` - ECMAScript модуль
- `.cjs` - CommonJS модуль (явное указание)
- `.js` - зависит от настройки `type` в `package.json`

### Пример ECMAScript модуля (file1-export.mjs)

```javascript
// Экспорт именованных идентификаторов
export const myFunction = () => {
  console.log('ES модуль функция');
};

export class MyClass {
  constructor(name) {
    this.name = name;
  }
}

export const collection = new Map();
```

### Импорт ECMAScript модулей

```javascript
// Импорт именованных экспортов
import { myFunction, MyClass, collection } from './file1-export.mjs';

// Импорт встроенных модулей
import * as events from 'node:events';
```

### Совместимость: импорт CommonJS из ESM

ECMAScript модули могут импортировать CommonJS модули:

```javascript
// Импорт CommonJS модуля из ESM
import fs from 'node:fs'; // default import для CommonJS
import { readFile } from 'node:fs'; // именованный импорт

// Импорт локального CommonJS модуля
import myModule from './commonjs-module.js';
```

**Важно**: Результат импорта идентичен, независимо от того, откуда импортируется (CommonJS или ESM).

---

## Динамический импорт

### Синтаксис и использование

Динамический импорт возвращает Promise и может использоваться как в CommonJS, так и в ESM.

```javascript
// Динамический импорт возвращает Promise
const eventsPromise = import('node:events');

eventsPromise.then((events) => {
  console.log(events); // модуль events
});

// С async/await
const events = await import('node:events');
console.log(events);
```

### Top-level await

**В ECMAScript модулях (.mjs):**
```javascript
// Можно использовать await в корне файла
const events = await import('node:events');
console.log(events.EventEmitter);
```

**В CommonJS модулях (.js с "use strict"):**
```javascript
'use strict';

// await нельзя использовать в корне
// await import('node:events'); // Ошибка!

// Но можно внутри async функции
(async () => {
  const events = await import('node:events');
  console.log(events);
})();
```

---

## Получение require в ECMAScript модулях

ECMAScript модули не имеют встроенного `require`. Для его использования нужно создать вручную:

```javascript
import { createRequire } from 'node:module';
import { fileURLToPath } from 'node:url';

// Получаем путь к текущему файлу через import.meta.url
const __filename = fileURLToPath(import.meta.url);

// Создаём require для текущего модуля
const require = createRequire(import.meta.url);

// Теперь можно использовать require
const fs = require('node:fs');
const ws = require('ws'); // npm пакет
```

### import.meta

Объект `import.meta` содержит метаданные о текущем модуле:

```javascript
console.log(import.meta);
// {
//   url: 'file:///absolute/path/to/module.mjs'
// }
```

**Применение:**
- Получение пути к текущему файлу
- Создание `require` в ESM
- Загрузка ресурсов относительно модуля

---

## Создание собственной системы модульности

### Наивная реализация CommonJS

Рассмотрим, как создать простую систему модульности, похожую на CommonJS:

```javascript
const fs = require('node:fs').promises;
const vm = require('node:vm');

async function load(filename, sandbox = {}) {
  // 1. Читаем содержимое файла
  const source = await fs.readFile(filename, 'utf8');

  // 2. Оборачиваем исходник в функцию
  const wrapper =
    `(require, module, __filename, __dirname) => {\n` +
    source +
    `\n}`;

  // 3. Создаём скрипт
  const script = new vm.Script(wrapper, { filename });

  // 4. Создаём изолированный контекст (sandbox)
  const context = vm.createContext(
    Object.freeze({ ...sandbox })
  );

  // 5. Выполняем скрипт в контексте
  const wrappedFunction = script.runInContext(context, {
    timeout: 5000,
    displayErrors: false
  });

  // 6. Создаём объект module для экспорта
  const module = { exports: {} };

  // 7. Создаём псевдо-require (заглушку)
  const require = (name) => {
    console.log('Вызван require для:', name);
    // Здесь может быть рекурсивная загрузка
  };

  // 8. Вызываем обёрнутую функцию с нужными параметрами
  wrappedFunction(
    require,
    module,
    filename,
    path.dirname(filename)
  );

  // 9. Возвращаем экспортированное содержимое
  return module.exports;
}

// Использование
const myModule = await load('./file1-export.js', {
  // Можем подменить глобальные идентификаторы
  Map: class PseudoMap extends Map {
    constructor() {
      super();
      console.log('Создан PseudoMap!');
    }
  }
});
```

**Как это работает:**

1. **Чтение файла** - получаем исходный код модуля
2. **Обёртка** - добавляем параметры `require`, `module`, `__filename`, `__dirname`
3. **Компиляция** - создаём VM-скрипт из обёрнутого кода
4. **Изоляция** - создаём отдельный контекст выполнения (sandbox)
5. **Выполнение** - запускаем скрипт с внедрёнными зависимостями
6. **Экспорт** - возвращаем `module.exports`

### Реальная реализация в Node.js

Это упрощённая версия. В реальном загрузчике Node.js (lib/internal/modules/cjs/loader.js):

```javascript
// Из исходников Node.js
const wrapper = [
  '(function (exports, require, module, __filename, __dirname) { ',
  '\n});'
];

function wrap(script) {
  return wrapper[0] + script + wrapper[1];
}
```

**Отличия от нашей реализации:**
- Использует обычную функцию вместо стрелочной
- Более сложная обработка ошибок
- Интеграция с кэшем модулей
- Поддержка циклических зависимостей

---

## Альтернативная система модульности

Создадим систему модульности, где модуль - это просто объект с методами:

### Формат модуля (example.mm)

```javascript
({
  // Обычные методы
  method1() {
    console.log('Method 1');
  },

  // Асинхронные методы
  async method2() {
    return 'Result';
  },

  // Свойства
  constant: 42,
  collection: new Map()
})
```

### Загрузчик для альтернативной системы

```javascript
const fs = require('node:fs').promises;
const vm = require('node:vm');

async function loadModule(filename, sandbox = {}) {
  // 1. Читаем исходник
  const source = await fs.readFile(filename, 'utf8');

  // 2. Добавляем перевод строки (для отладки)
  const wrappedSource = '\n' + source + '\n';

  // 3. Создаём скрипт
  const script = new vm.Script(wrappedSource, { filename });

  // 4. Создаём контекст
  const context = vm.createContext({ ...sandbox });

  // 5. Выполняем и сразу возвращаем результат
  return script.runInContext(context, {
    timeout: 5000
  });
}

// Использование
const myModule = await loadModule('./example.mm', {
  console // передаём console в sandbox
});

console.log(myModule.constant); // 42
myModule.method1(); // 'Method 1'
```

**Преимущества:**
- Проще, чем CommonJS (нет обёртки в функцию)
- Модуль сразу возвращает объект
- Меньше замыканий
- Можно писать код до объявления объекта

**Недостатки:**
- Нет стандартизации
- Меньше экосистемной поддержки

---

## Работа с пакетами (packages)

### Структура пакета

Пакет - это директория с файлом `package.json`, который описывает пакет.

#### Базовый package.json

```json
{
  "name": "my-package",
  "version": "1.0.0",
  "author": "Author Name",
  "main": "main.js"
}
```

**Ключевые поля:**
- `name` - имя пакета
- `version` - версия (semver)
- `main` - точка входа (по умолчанию `index.js`)
- `author` - автор пакета

### Различные способы загрузки пакетов

#### 1. Загрузка корня пакета

```javascript
// Все эквивалентны - загрузят main.js из package.json
require('./package1');
require('./package1/');
require('./package1/.');
```

#### 2. Загрузка конкретного файла

```javascript
// Явно указываем файл - package.json игнорируется
require('./package1/main');
require('./package1/main.js');
```

#### 3. Загрузка из node_modules

```javascript
// Ищет в node_modules по алгоритму поиска
const pkg = require('package-name');
```

#### 4. Загрузка подмодулей

```javascript
// Загрузит utils.js из пакета
require('./package1/utils');
require('./package1/utils.js');
```

**Порядок проверки расширений:**
1. Проверяет директорию `utils/` с `package.json`
2. Проверяет файл `utils.json`
3. Проверяет файл `utils.js`
4. Проверяет файл `utils.cjs`

**Рекомендация:** Всегда указывайте `.js` расширение для файлов, чтобы избежать лишних проверок.

### Загрузка JSON файлов

CommonJS `require` умеет загружать JSON:

```javascript
// Загрузка package.json как объект
const packageInfo = require('./package1/package.json');

console.log(packageInfo.name); // 'my-package'
console.log(packageInfo.version); // '1.0.0'
```

---

## ECMAScript пакеты

### Указание типа модульной системы

В `package.json` можно указать `type`:

```json
{
  "name": "esm-package",
  "version": "1.0.0",
  "type": "module",
  "main": "main.js"
}
```

**Эффект:**
- Файлы `.js` теперь трактуются как ECMAScript модули
- Для CommonJS нужно использовать расширение `.cjs`
- Можно использовать `import`/`export` в `.js` файлах

### Пример main.js с type: "module"

```javascript
// main.js воспринимается как ESM благодаря "type": "module"
export const myFunction = () => {
  console.log('ESM функция');
};

export class MyClass {
  constructor() {
    console.log('ESM класс');
  }
}
```

---

## Экспорт нескольких точек входа

### Поле exports в package.json

Современный способ определения точек входа:

```json
{
  "name": "multi-entry-package",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": "./main.mjs",
    "./utils": "./utils.mjs"
  }
}
```

**Использование:**

```javascript
// Загружает main.mjs
import pkg from 'multi-entry-package';

// Загружает utils.mjs
import utils from 'multi-entry-package/utils';
```

### Conditional Exports (условный экспорт)

Можно определять разные точки входа для разных систем модулей:

```json
{
  "name": "universal-package",
  "version": "1.0.0",
  "exports": {
    ".": {
      "import": "./main.mjs",
      "require": "./main.cjs"
    }
  }
}
```

**Поведение:**

```javascript
// Из ECMAScript модуля - загрузит main.mjs
import pkg from 'universal-package';

// Из CommonJS модуля - загрузит main.cjs
const pkg = require('universal-package');
```

**Применение:**
- Создание универсальных библиотек
- Оптимизация для разных окружений
- Поддержка legacy кода

---

## Кэширование и изоляция модулей

### Общий кэш для require и import?

Проверим, используют ли `require` и динамический `import` один кэш:

```javascript
import { createRequire } from 'node:module';

// Создаём require в ESM
const require = createRequire(import.meta.url);

// Загружаем модуль через import
const m1 = await import('node:module');

// Загружаем через динамический import снова
const m2 = await import('node:module');

// Загружаем через require
const m3 = require('node:module');

// Загружаем через require снова
const m4 = require('node:module');

// Проверяем идентичность
console.log(m1 === m2); // true - динамический import кэширует
console.log(m3 === m4); // true - require тоже кэширует
console.log(m1 === m3); // false (!) - разные кэши!
```

**Вывод:**
- `import` и `require` используют **разные кэши**
- Каждая система модулей имеет свой cache
- Это может привести к загрузке одного модуля дважды

### Conditional exports и кэш

При использовании conditional exports:

```javascript
// package.json:
// "exports": {
//   ".": {
//     "import": "./main.mjs",
//     "require": "./main.cjs"
//   }
// }

const m1 = require('package'); // загружает main.cjs
const m2 = await import('package'); // загружает main.mjs

console.log(m1 === m2); // false - это разные файлы!
```

**Важно:** Если пакет имеет разные точки входа для `import` и `require`, они загружаются как отдельные модули и имеют отдельное состояние.

---

## Песочница (Sandbox) и изоляция кода

### Создание изолированного контекста

Используя `vm.createContext`, можно создать изолированное окружение:

```javascript
const vm = require('node:vm');

// Создаём sandbox с ограниченным API
const sandbox = {
  console: {
    log: (...args) => {
      console.log('[SANDBOX]', ...args);
    }
  },
  // Подменяем встроенные классы
  Map: class RestrictedMap extends Map {
    set(key, value) {
      console.log('Map.set вызван:', key, value);
      return super.set(key, value);
    }
  }
};

// Создаём контекст
const context = vm.createContext(sandbox);

// Код, который выполнится в песочнице
const code = `
  const map = new Map();
  map.set('key', 'value');
  console.log(map.get('key'));
`;

// Выполняем в изолированном контексте
vm.runInContext(code, context, {
  timeout: 5000,
  displayErrors: false
});
```

**Применение:**
- Выполнение ненадёжного кода
- Изоляция плагинов
- Ограничение доступа к API
- Тестирование с mock-объектами

### Ограничения песочниц

**Важно:** Песочницы VM в Node.js не являются полноценной защитой:
- Возможен выход через прототипы
- Доступ к нативным функциям
- Не защищает от DoS (расход CPU/памяти)

Для настоящей изоляции используйте:
- Worker Threads
- Child Processes
- Контейнеры (Docker)
- Виртуальные машины

---

## Защита от модификаций и monkey patching

### Проблема

Любая зависимость может модифицировать:
- Встроенные модули Node.js
- Глобальные прототипы (`Array.prototype`, `Object.prototype`)
- Глобальные объекты

### Решения

#### 1. Использование Object.freeze

```javascript
// Заморозка экспортируемых объектов
const API = Object.freeze({
  method1() { },
  method2() { }
});

module.exports = API;
```

#### 2. Использование песочниц

```javascript
// Запуск зависимостей в изолированном контексте
const vm = require('node:vm');

const sandbox = vm.createContext({
  // Контролируемое окружение
});

// Загрузка модуля в песочнице
```

#### 3. Аудит зависимостей

- Регулярная проверка зависимостей
- Использование `npm audit`
- Минимизация зависимостей
- Code review важных зависимостей

#### 4. Immutable exports

```javascript
// Экспортируем только функции (не объекты)
module.exports = {
  createInstance() {
    return { /* новый объект каждый раз */ };
  }
};
```

---

## Лучшие практики модульной системы

### 1. Явное указание расширений и префиксов

```javascript
// Хорошо
const fs = require('node:fs');
const myModule = require('./module.js');

// Плохо
const fs = require('fs'); // может конфликтовать с npm пакетом
const myModule = require('./module'); // лишние проверки расширений
```

### 2. Immutable exports

```javascript
// Хорошо - неизменяемый API
module.exports = Object.freeze({
  method1: () => {},
  method2: () => {}
});

// Плохо - мутабельный объект
module.exports = {
  data: new Map() // может быть изменён извне
};
```

### 3. Избегайте глобального состояния

```javascript
// Плохо - глобальное состояние в модуле
let counter = 0;

module.exports = {
  increment() { counter++; },
  getCount() { return counter; }
};

// Хорошо - фабрика экземпляров
module.exports = {
  createCounter() {
    let counter = 0;
    return {
      increment() { counter++; },
      getCount() { return counter; }
    };
  }
};
```

### 4. Документирование контрактов модулей

```javascript
/**
 * Модуль для работы с пользователями
 * @module users
 */

/**
 * Создаёт нового пользователя
 * @param {string} name - Имя пользователя
 * @param {string} email - Email пользователя
 * @returns {Object} Объект пользователя
 */
function createUser(name, email) {
  return { name, email };
}

module.exports = { createUser };
```

### 5. Используйте ESM для новых проектов

```javascript
// Современный подход - ECMAScript модули
// package.json: "type": "module"

// Явный импорт
import { readFile } from 'node:fs/promises';
import { createServer } from 'node:http';

// Асинхронная загрузка
const config = await import('./config.js');
```

---

## Структура приложения и слои

### Применение модульной системы для чистой архитектуры

Модульная система позволяет организовать код по принципам:

1. **Разделение ответственности**
   - Бизнес-логика в отдельных модулях
   - Инфраструктурный код изолирован

2. **Dependency Injection через sandbox**
   - Внедрение зависимостей через параметры модуля
   - Изоляция слоёв приложения

3. **Минимизация coupling**
   - Модули знают минимум друг о друге
   - Зависимости через интерфейсы

**Пример структуры:**

```
app/
  ├── domain/          # Бизнес-логика (чистые функции)
  │   ├── user.js
  │   └── order.js
  ├── services/        # Сервисы приложения
  │   ├── userService.js
  │   └── orderService.js
  ├── infrastructure/  # Детали реализации
  │   ├── database.js
  │   ├── http.js
  │   └── logger.js
  └── main.js         # Точка входа, композиция
```

---

## Итоги и ключевые моменты

### Основные концепции

1. **Системы модульности в Node.js**
   - CommonJS - традиционная система (`require`/`module.exports`)
   - ECMAScript Modules - современный стандарт (`import`/`export`)
   - Обе системы могут сосуществовать в одном проекте

2. **Механизм загрузки**
   - Модули оборачиваются в функции с внедрёнными зависимостями
   - Используется `vm.Script` для выполнения в изолированном контексте
   - Алгоритм поиска модулей идёт от текущей директории вверх

3. **Кэширование**
   - `require` кэширует модули (паттерн Singleton)
   - `import` имеет отдельный кэш
   - Очистка кэша возможна, но не решает все проблемы

4. **Пакеты**
   - Описываются через `package.json`
   - Поддерживают множественные точки входа (`exports`)
   - Могут иметь условный экспорт для разных систем модулей

5. **Безопасность**
   - Monkey patching - серьёзная угроза
   - Песочницы обеспечивают базовую изоляцию
   - Требуется аудит зависимостей

### Рекомендации

1. Используйте префикс `node:` для встроенных модулей
2. Указывайте расширения файлов явно
3. Делайте экспорты неизменяемыми (immutable)
4. Избегайте глобального состояния в модулях
5. Документируйте контракты модулей
6. Применяйте ESM для новых проектов
7. Используйте песочницы для ненадёжного кода

### Дальнейшее изучение

- Документация Node.js по модулям: https://nodejs.org/api/modules.html
- ECMAScript Modules: https://nodejs.org/api/esm.html
- VM API: https://nodejs.org/api/vm.html
- Package.json exports: https://nodejs.org/api/packages.html#exports

---

## Практические задания

1. **Создайте свою систему модульности**
   - Реализуйте функцию `load()` с поддержкой кэша
   - Добавьте рекурсивную загрузку зависимостей
   - Реализуйте песочницу с ограничениями

2. **Исследуйте кэш модулей**
   - Загрузите модуль и изучите структуру `require.cache`
   - Попробуйте очистить кэш и перезагрузить модуль
   - Проверьте, как кэшируются циклические зависимости

3. **Создайте универсальный пакет**
   - Напишите пакет с поддержкой CommonJS и ESM
   - Используйте conditional exports
   - Добавьте множественные точки входа

4. **Защита от модификаций**
   - Создайте модуль с immutable API
   - Реализуйте песочницу с ограниченными возможностями
   - Напишите тесты, проверяющие невозможность модификации

