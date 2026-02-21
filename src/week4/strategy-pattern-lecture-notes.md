# Паттерн «Стратегия» (Strategy Pattern) — GoF для JavaScript и TypeScript

## Обзор

Паттерн **Стратегия** (Strategy) — это поведенческий паттерн проектирования, который позволяет выбирать один из алгоритмов или реализаций во время выполнения программы (runtime). В JavaScript и TypeScript этот паттерн реализуется более элегантно и гибко, чем в традиционных объектно-ориентированных языках, благодаря использованию коллекций и функций первого класса.

---

## Основные концепции

### Что позволяет делать паттерн Стратегия?

Паттерн Стратегия позволяет:

- **Выбирать алгоритмы** — различные способы решения одной задачи
- **Выбирать реализации** — разные классы, фабрики или функции
- **Переключаться между поведениями** в runtime
- **Создавать семейства алгоритмов** с одинаковым интерфейсом

### Типичные применения

**Примеры использования:**

- Алгоритмы шифрования (AES, RSA, DES)
- Транспортные протоколы (HTTP, WebSocket, TCP)
- Способы сериализации/десериализации (JSON, XML, MessagePack)
- Рендеринг в разных форматах (HTML, Markdown, Console, SVG, PNG)
- Стратегии сортировки и фильтрации данных

---

## Реализация в JavaScript vs другие языки

### Особенности JavaScript-подхода

В отличие от Java, C# и других классических ОО-языков, в JavaScript **не нужно создавать большие семейства классов**. Вместо этого можно использовать:

- **Коллекции функций** (объекты с методами)
- **Обычные объекты** с ключами и значениями
- **Map/Set** для хранения стратегий
- **Выбор по ключу** из коллекции в runtime

**Преимущества:**

✅ Замена наследования композицией
✅ Замена классов функциями
✅ Простота выбора стратегии по ключу
✅ Динамическое переключение поведения

**Минусы:**

⚠️ Код может стать немного сложнее (но не так сложно, как в Java/C#)
⚠️ Нужно следить за единообразием интерфейсов

---

## Пример 1: Классическая реализация через наследование

### Структура

```javascript
// Абстрактный базовый класс
class Renderer {
  render(data) {
    throw new Error('Abstract method must be implemented');
  }
}

// Конкретные реализации
class ConsoleRenderer extends Renderer {
  render(data) {
    console.table(data);
  }
}

class WebRenderer extends Renderer {
  render(data) {
    return `<table>${this.generateHTML(data)}</table>`;
  }

  generateHTML(data) {
    // Генерация HTML из данных
    return data.map(item =>
      `<tr>${Object.values(item).map(val => `<td>${val}</td>`).join('')}</tr>`
    ).join('');
  }
}

class MarkdownRenderer extends Renderer {
  render(data) {
    const headers = Object.keys(data[0]).join(' | ');
    const separator = Object.keys(data[0]).map(() => '---').join(' | ');
    const rows = data.map(item =>
      Object.values(item).join(' | ')
    ).join('\n');

    return `${headers}\n${separator}\n${rows}`;
  }
}
```

### Контекст для использования стратегий

```javascript
// Класс-контекст, который использует стратегию
class RenderContext {
  constructor(renderer) {
    this.renderer = renderer;
  }

  process(data) {
    return this.renderer.render(data);
  }
}
```

### Использование

```javascript
const data = [
  { name: 'Alice', age: 30, city: 'Moscow' },
  { name: 'Bob', age: 25, city: 'London' },
  { name: 'Charlie', age: 35, city: 'New York' }
];

// Создаем контексты с разными стратегиями рендеринга
const consoleContext = new RenderContext(new ConsoleRenderer());
const webContext = new RenderContext(new WebRenderer());
const markdownContext = new RenderContext(new MarkdownRenderer());

// Используем одинаковый интерфейс
console.log(consoleContext.process(data));
console.log(webContext.process(data));
console.log(markdownContext.process(data));
```

**Ключевая идея:** Все контексты имеют одинаковый интерфейс (`process`), но внутри используются разные стратегии рендеринга. Мы не знаем деталей реализации — просто вызываем `process()`.

---

## Пример 2: Реализация через коллекцию функций (JavaScript-way)

### Преимущество подхода

Вместо создания множества классов можно использовать **простой объект с функциями**. Это более идиоматично для JavaScript.

```javascript
// Коллекция стратегий в виде объекта с функциями
const renderers = {
  // Абстрактная стратегия по умолчанию
  abstract: (data) => {
    return 'Renderer not implemented for this format';
  },

  // Консольный рендерер
  console: (data) => {
    console.table(data);
  },

  // Web-рендерер (HTML)
  web: (data) => {
    const headers = Object.keys(data[0])
      .map(key => `<th>${key}</th>`)
      .join('');
    const rows = data.map(item =>
      `<tr>${Object.values(item).map(val => `<td>${val}</td>`).join('')}</tr>`
    ).join('');

    return `<table><thead><tr>${headers}</tr></thead><tbody>${rows}</tbody></table>`;
  },

  // Markdown-рендерер
  markdown: (data) => {
    const headers = Object.keys(data[0]).join(' | ');
    const separator = Object.keys(data[0]).map(() => '---').join(' | ');
    const rows = data.map(item =>
      Object.values(item).join(' | ')
    ).join('\n');

    return `${headers}\n${separator}\n${rows}`;
  }
};
```

### Контекст с выбором стратегии

```javascript
// Фабрика контекста
const createContext = (rendererName) => {
  // Выбираем рендерер по имени или берем абстрактный
  const renderer = renderers[rendererName] || renderers.abstract;

  // Возвращаем функцию с единообразным интерфейсом
  return (data) => renderer(data);
};
```

### Использование

```javascript
const data = [
  { name: 'Alice', age: 30, city: 'Moscow' },
  { name: 'Bob', age: 25, city: 'London' }
];

// Создаем контексты для разных форматов
const pngContext = createContext('png');      // Несуществующий формат
const consoleContext = createContext('console');
const webContext = createContext('web');
const markdownContext = createContext('markdown');

// Используем одинаковым образом
console.log(pngContext(data));        // "Renderer not implemented..."
consoleContext(data);                  // Выводит таблицу в консоль
console.log(webContext(data));         // Возвращает HTML
console.log(markdownContext(data));    // Возвращает Markdown
```

**Преимущества этого подхода:**

✅ Нет необходимости создавать классы
✅ Легко добавлять новые стратегии
✅ Простой выбор по ключу
✅ Компактный и читаемый код

---

## Пример 3: Стратегия сортировки массивов

### Проблема

Стандартный метод `Array.prototype.sort()` требует передачи функции-компаратора. Но можно создать **декларативную стратегию сортировки**, которая будет более выразительной.

### Реализация декларативной стратегии

```javascript
// Создаем стратегию сортировки из декларативного описания
const createSortStrategy = (options) => {
  // Получаем поле для сортировки
  const field = Object.keys(options)[0];
  const direction = options[field]; // 'asc' или 'desc'

  // Определяем порядок сортировки
  const order = direction === 'asc' ? 1 : -1;

  // Возвращаем функцию-компаратор
  return (a, b) => {
    const valueA = a[field];
    const valueB = b[field];

    if (valueA < valueB) return -1 * order;
    if (valueA > valueB) return 1 * order;
    return 0;
  };
};
```

### Использование стратегии сортировки

```javascript
const scientists = [
  { name: 'Einstein', born: 1879 },
  { name: 'Newton', born: 1643 },
  { name: 'Tesla', born: 1856 },
  { name: 'Curie', born: 1867 },
  { name: 'Darwin', born: 1809 }
];

// Создаем стратегию: сортировка по году рождения (по убыванию)
const sortByBornDesc = createSortStrategy({ born: 'desc' });

// Применяем стратегию
const sorted = scientists.sort(sortByBornDesc);

console.log(sorted);
// Результат: Einstein (1879), Tesla (1856), Curie (1867), Darwin (1809), Newton (1643)

// Сортировка по имени (по возрастанию)
const sortByNameAsc = createSortStrategy({ name: 'asc' });
const sortedByName = scientists.sort(sortByNameAsc);

console.log(sortedByName);
// Результат: Curie, Darwin, Einstein, Newton, Tesla
```

### Детали реализации

```javascript
// Полная реализация с поддержкой разных типов
const createSortStrategy = (options) => {
  const field = Object.keys(options)[0];
  const direction = options[field];
  const order = direction === 'asc' ? 1 : -1;

  return (a, b) => {
    const valueA = a[field];
    const valueB = b[field];

    // Сравнение с учетом типов
    if (typeof valueA === 'string' && typeof valueB === 'string') {
      return valueA.localeCompare(valueB) * order;
    }

    if (valueA < valueB) return -1 * order;
    if (valueA > valueB) return 1 * order;
    return 0;
  };
};
```

**Ключевое преимущество:** Вместо написания функции-компаратора вручную каждый раз, мы **декларативно описываем** желаемую сортировку.

---

## Пример 4: Стратегия фильтрации

### Простая фильтрация

```javascript
const data = [
  { name: 'Alice', age: 30, active: true },
  { name: 'Bob', age: 25, active: false },
  { name: 'Charlie', age: 35, active: true }
];

// Обычная функция-стратегия
const activeUsersFilter = (item) => item.active;
const adults = data.filter(activeUsersFilter);
```

### Декларативная стратегия фильтрации

```javascript
// Создаем стратегию фильтрации из описания
const createFilterStrategy = (criteria) => {
  return (item) => {
    // Проверяем все критерии
    return Object.entries(criteria).every(([field, value]) => {
      if (typeof value === 'function') {
        return value(item[field]);
      }
      return item[field] === value;
    });
  };
};

// Использование
const filter1 = createFilterStrategy({ active: true, age: (age) => age > 25 });
const filtered = data.filter(filter1);

console.log(filtered);
// Результат: [{ name: 'Alice', age: 30, active: true },
//             { name: 'Charlie', age: 35, active: true }]
```

**Паттерн в действии:** Мы превращаем **декларативное описание** критериев в **функциональную стратегию** фильтрации.

---

## Метапрограммирование и Стратегия

### Концепция

Создание стратегий из декларативных описаний — это элемент **метапрограммирования**. Мы пишем код, который генерирует другой код (функции) на основе конфигурации.

```javascript
// Декларативное описание → Функция-стратегия
const config = { born: 'desc' };
const strategy = createSortStrategy(config);

// Теперь strategy — это функция, которую можно использовать
array.sort(strategy);
```

**Преимущества подхода:**

- Переиспользование логики создания стратегий
- Уменьшение дублирования кода
- Более выразительный и понятный код
- Легкость тестирования

---

## Продвинутые паттерны: Коллекции в коллекциях

### Многоуровневые стратегии

Можно создавать **вложенные коллекции стратегий** для более сложных сценариев.

```javascript
// Стратегии работы с разными источниками данных
const dataSourceStrategies = {
  database: {
    mysql: class MySQLCursor { /* ... */ },
    postgres: class PostgresCursor { /* ... */ },
    mongodb: class MongoCursor { /* ... */ }
  },

  file: {
    csv: class CSVReader { /* ... */ },
    json: class JSONReader { /* ... */ },
    xml: class XMLReader { /* ... */ }
  },

  cloud: {
    s3: class S3Stream { /* ... */ },
    azure: class AzureStream { /* ... */ },
    gcp: class GCPStream { /* ... */ }
  }
};

// Выбор стратегии
const getStrategy = (source, type) => {
  return dataSourceStrategies[source]?.[type];
};

// Использование
const MySQLClass = getStrategy('database', 'mysql');
const cursor = new MySQLClass(config);
```

### Абстрактная фабрика через коллекции

```javascript
// Стратегии рендеринга графических примитивов
const renderingStrategies = {
  svg: {
    Circle: class SVGCircle { /* ... */ },
    Rectangle: class SVGRectangle { /* ... */ },
    Line: class SVGLine { /* ... */ }
  },

  canvas: {
    Circle: class CanvasCircle { /* ... */ },
    Rectangle: class CanvasRectangle { /* ... */ },
    Line: class CanvasLine { /* ... */ }
  },

  webgl: {
    Circle: class WebGLCircle { /* ... */ },
    Rectangle: class WebGLRectangle { /* ... */ },
    Line: class WebGLLine { /* ... */ }
  }
};

// Создание фабрики для конкретной стратегии
const createShapeFactory = (renderingType) => {
  const strategies = renderingStrategies[renderingType];

  return {
    createCircle: (x, y, radius) => new strategies.Circle(x, y, radius),
    createRectangle: (x, y, w, h) => new strategies.Rectangle(x, y, w, h),
    createLine: (x1, y1, x2, y2) => new strategies.Line(x1, y1, x2, y2)
  };
};

// Использование
const svgFactory = createShapeFactory('svg');
const circle = svgFactory.createCircle(100, 100, 50);

const canvasFactory = createShapeFactory('canvas');
const rect = canvasFactory.createRectangle(0, 0, 200, 100);
```

**Это паттерн Абстрактная фабрика**, реализованный через **вложенные коллекции стратегий**!

---

## Практические советы

### Когда использовать Strategy

✅ **Используйте, когда:**

- Нужно выбирать между несколькими алгоритмами в runtime
- Есть семейство классов, отличающихся только поведением
- Требуется избежать множественных условных операторов
- Нужна гибкость в добавлении новых стратегий

❌ **Не используйте, когда:**

- Существует только одна реализация
- Различия между алгоритмами минимальны
- Клиенту нужно знать детали реализации

### Рекомендации для JavaScript/TypeScript

1. **Предпочитайте функции классам** для простых стратегий
2. **Используйте коллекции** (объекты, Map) для хранения стратегий
3. **Создавайте фабрики** для генерации стратегий из конфигурации
4. **Обеспечивайте единообразие** интерфейсов всех стратегий
5. **Добавляйте стратегию по умолчанию** для обработки неизвестных случаев

### TypeScript: Типизация стратегий

```typescript
// Определяем интерфейс стратегии
interface RenderStrategy<T, R> {
  (data: T): R;
}

// Типизированная коллекция стратегий
const renderers: Record<string, RenderStrategy<any[], string>> = {
  console: (data) => {
    console.table(data);
    return '';
  },

  web: (data) => {
    return `<table>${/* ... */}</table>`;
  },

  markdown: (data) => {
    return `| ${/* ... */} |`;
  }
};

// Типизированная фабрика контекста
function createContext<T, R>(
  name: string
): RenderStrategy<T, R> {
  return renderers[name] || renderers.abstract;
}
```

---

## Резюме

### Ключевые выводы

1. **Паттерн Стратегия** позволяет выбирать алгоритмы и реализации в runtime
2. В **JavaScript** можно использовать коллекции вместо иерархий классов
3. **Функции первого класса** делают реализацию очень гибкой
4. **Декларативные стратегии** упрощают создание и использование
5. **Вложенные коллекции** позволяют реализовать сложные паттерны

### Примеры из реальной практики

- **Методы массивов:** `sort()`, `filter()`, `map()` — все принимают стратегии
- **Middleware в Express.js** — стратегии обработки запросов
- **Serializers в API** — JSON, XML, Protobuf
- **Validation strategies** — разные правила валидации
- **Authentication strategies** в Passport.js

### Связь с другими паттернами

- **Стратегия + Фабрика** = выбор стратегии через фабричный метод
- **Стратегия + Декоратор** = добавление поведения к стратегиям
- **Стратегия + Абстрактная фабрика** = коллекции в коллекциях
- **Стратегия + Состояние** = похожи, но State меняет поведение объекта

---

## Дополнительные материалы

### Примеры кода

В лекции были продемонстрированы следующие примеры:

1. **Классическая реализация** — через наследование классов
2. **Прототипное наследование** — устаревший подход (для справки)
3. **Коллекция функций** — современный JavaScript-подход
4. **Декларативная сортировка** — метапрограммирование
5. **Декларативная фильтрация** — расширение идеи

### Рекомендуемая практика

- Изучить примеры из репозитория
- Реализовать собственные стратегии для реальных задач
- Попробовать создать вложенные коллекции стратегий
- Изучить использование Strategy в популярных библиотеках

### Вопросы для самопроверки

1. Чем отличается реализация Strategy в JavaScript от Java/C#?
2. Когда следует использовать коллекции вместо классов?
3. Что такое декларативная стратегия?
4. Как связаны Strategy и Абстрактная фабрика?
5. Какие встроенные методы JavaScript используют паттерн Strategy?

---

**Конец конспекта лекции**

*Паттерн Стратегия — один из самых полезных и часто используемых паттернов в JavaScript. Освоив его, вы сможете писать более гибкий, расширяемый и поддерживаемый код.*
