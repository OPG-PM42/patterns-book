# Builder Pattern - Lecture Notes

## Overview

The **Builder Pattern** is a creational design pattern from the Gang of Four (GoF) that separates the construction of complex objects from their representation. It allows you to construct complex objects step by step, providing flexibility in creating different representations of an object using the same construction process.

### Main Purpose

The Builder pattern addresses the problem of complex object instantiation by:
- Separating object construction logic from the domain model/class itself
- Moving instantiation logic to a dedicated builder class
- Providing a clean interface for object initialization and modification
- Allowing different configurations and variations of object creation

---

## Key Concepts

### The Problem

When a constructor becomes too complex and we want to separate the instantiation logic from the domain model or class itself, we can extract this responsibility into a separate class that specializes in creating instances.

**Key insight**: The Builder is a class dedicated to instantiating another class (or multiple classes).

### Core Components

The Builder pattern typically involves three main components:

1. **Product** - The complex object being built
2. **Builder** - The class responsible for constructing the product
3. **Director** (optional) - Controls the sequence of building steps

**Important**: The Director is not always required - it's an optional component that manages the order of construction steps.

---

## Advantages of the Builder Pattern

### 1. Separation of Responsibilities

The pattern achieves clear separation of concerns by isolating the construction logic from the business logic of the object being created.

### 2. Flexible Initialization

Through different options and methods, you can create various initialization variants. The initialization process doesn't have to be linear - it can support multiple alternative configurations.

### 3. Complexity Hiding

If complexity can be hidden and isolated behind the Builder pattern, this is actually beneficial:
- The complex code can be properly tested in isolation
- The main codebase remains clean and simple
- Code size increases, but complexity is encapsulated

---

## Disadvantages of the Builder Pattern

### 1. Increased Complexity

Without the Builder pattern there's already complexity, but adding the pattern introduces additional structural complexity.

### 2. Code Size

Additional classes need to be created (Director, Builder), which means more code to maintain.

**However**: If complexity is properly hidden and isolated, and the code is well-tested, the increased code size becomes less of a concern. The benefits of encapsulation often outweigh the cost.

---

## Classic Implementation Example

### Product Class

The product is the object we want to build. In this example, it has three fields:

```javascript
class Product {
  constructor() {
    this.field1 = null;
    this.field2 = null;
    this.field3 = null;
  }
}
```

### Abstract Builder

The abstract builder defines the interface for all concrete builders. In classical implementation, it includes:
- A constructor that throws an exception (cannot be instantiated directly)
- Step methods for building (step1, step2, step3)
- CreateInstance method for creating the product instance
- GetInstance method for retrieving the finished product

```javascript
class AbstractBuilder {
  constructor() {
    // Cannot be called directly - throws exception
    throw new Error('Abstract class cannot be instantiated');
  }

  step1() {
    throw new Error('Method must be implemented');
  }

  step2() {
    throw new Error('Method must be implemented');
  }

  step3() {
    throw new Error('Method must be implemented');
  }

  createInstance() {
    throw new Error('Method must be implemented');
  }

  getInstance() {
    throw new Error('Method must be implemented');
  }
}
```

### Concrete Builder

The concrete builder inherits from the abstract builder and implements all methods:

```javascript
class ConcreteBuilder extends AbstractBuilder {
  constructor() {
    super(); // This won't throw because we're in subclass
    this.instance = null;
  }

  createInstance() {
    // Creates a new product instance
    this.instance = new Product();
    return this;
  }

  step1() {
    // Initialize field1
    this.instance.field1 = 'value1';
    return this;
  }

  step2() {
    // Initialize field2
    this.instance.field2 = 'value2';
    // Could also:
    // - Establish connections
    // - Set up timeouts
    // - Attach callbacks
    // - Any complex initialization
    return this;
  }

  step3() {
    // Initialize field3
    this.instance.field3 = 'value3';
    return this;
  }

  getInstance() {
    // Returns the finished product
    const result = this.instance;
    this.instance = null; // Reset for next build
    return result;
  }
}
```

### Director

The Director manages the sequence of building steps:

```javascript
class Director {
  constructor(builder) {
    this.builder = builder;
  }

  create() {
    // Orchestrates the building process
    this.builder.createInstance();
    this.builder.step1();
    this.builder.step2();
    this.builder.step3();
    return this.builder.getInstance();
  }
}
```

### Usage - Classic Pattern

```javascript
// Create a concrete builder
const builder = new ConcreteBuilder();

// Pass it to the director
const director = new Director(builder);

// Director executes all steps and returns the product
const product = director.create();

// All complexity is hidden during construction
```

**Note**: While this follows the classical pattern from books (adapted from Java-style examples), JavaScript offers more flexible alternatives due to first-class functions.

---

## Modern JavaScript Approach

### Alternative Without Director

In JavaScript, with first-class functions, we can simplify the pattern. Steps can be separate functions or procedures that are called sequentially, then combined in a factory function.

```javascript
// Simple factory approach
function createProduct(config) {
  const product = new Product();

  // Apply configuration steps
  if (config.field1) product.field1 = config.field1;
  if (config.field2) product.field2 = config.field2;
  if (config.field3) product.field3 = config.field3;

  return product;
}
```

This is often simpler and more idiomatic in JavaScript than the full classical pattern.

---

## Practical Example: Query Builder

A real-world example that demonstrates the Builder pattern is a SQL Query Builder. This example shows how to construct complex SQL queries step by step.

### QueryBuilder Class

```javascript
class QueryBuilder {
  constructor(table) {
    // Initialize options object with table and empty clauses
    this.options = {
      table: table,
      fields: '*',
      where: '',
      order: '',
      limit: ''
    };
  }

  // Method to set WHERE clause
  where(conditions) {
    this.options.where = conditions;
    return this; // Return this for method chaining
  }

  // Method to set ORDER BY clause
  order(field) {
    this.options.order = field;
    return this; // Return this for method chaining
  }

  // Method to set LIMIT clause
  limit(count) {
    this.options.limit = count;
    return this; // Return this for method chaining
  }

  // Build and return the final SQL query
  then(resolve, reject) {
    // Extract all accumulated options
    const { table, fields, where, order, limit } = this.options;

    // Build query parts
    let query = `SELECT ${fields} FROM ${table}`;

    if (where) {
      query += ` WHERE ${where}`;
    }

    if (order) {
      query += ` ORDER BY ${order}`;
    }

    if (limit) {
      query += ` LIMIT ${limit}`;
    }

    // Return the complete query
    // In real implementation, this would execute the query
    resolve(query);
  }
}
```

### Key Features

**Method Chaining**: Each method returns `this`, enabling fluent interface:

```javascript
builder.where('country = "UA"').order('population').limit(10);
```

**Thenable Contract**: The class implements a `then` method, making it compatible with async/await and Promise chains. This means you can use `await` before the constructor:

```javascript
const result = await new QueryBuilder('cities')
  .where('country = "UA"')
  .order('population')
  .limit(10);
```

### Usage Example - Method Chaining

```javascript
// Using method chaining syntax
const query = await new QueryBuilder('cities')
  .where('country = "UA" AND type = 1')
  .order('population')
  .limit(10);

// Query built:
// "SELECT * FROM cities WHERE country = "UA" AND type = 1 ORDER BY population LIMIT 10"
```

**Advantages of chaining**:
- Clean, readable syntax
- Each step is explicit
- Flexible - can add or remove steps easily

---

## Adding a Director to Query Builder

While method chaining is convenient, sometimes we want to pass all options at once (e.g., when receiving JSON from frontend or reading from a file).

### Director for Batch Configuration

```javascript
class QueryDirector {
  constructor(builder, options) {
    this.builder = builder;
    this.options = options;
  }

  build() {
    // Apply all options at once
    const { where, order, limit } = this.options;

    if (where) this.builder.where(where);
    if (order) this.builder.order(order);
    if (limit) this.builder.limit(limit);

    return this.builder;
  }
}
```

### Usage with Director

```javascript
// Options object (could come from JSON, file, frontend, etc.)
const options = {
  where: 'country = "UA" AND type = 1',
  order: 'population',
  limit: 10
};

// Create builder
const builder = new QueryBuilder('cities');

// Use director to apply all options
const director = new QueryDirector(builder, options);
const query = await director.build();
```

**Benefits**:
- Can work with configuration objects
- Useful when options come from external sources (JSON, API, files)
- Combines well with the chaining approach

---

## Method Chaining vs Object Configuration

### Method Chaining Approach

```javascript
const result = await new QueryBuilder('cities')
  .where('country = "UA"')
  .order('population')
  .limit(10);
```

**Pros**:
- Very readable and explicit
- Fluent, natural syntax
- Easy to see the construction steps

**Cons**:
- Not convenient when you already have an options object
- Requires writing each method call separately

### Object Configuration Approach

```javascript
const options = {
  table: 'cities',
  where: 'country = "UA"',
  order: 'population',
  limit: 10
};

// Direct initialization or via Director
const result = await buildQuery(options);
```

**Pros**:
- Options can come from external sources (JSON, files, frontend)
- Easy to store and transmit configurations
- More compact for complex configurations

**Cons**:
- Less explicit about the building process
- Harder to see the construction steps

### Best Practice: Support Both

A well-designed builder should support both approaches:

```javascript
// Chaining when building step by step
const query1 = await new QueryBuilder('cities')
  .where('country = "UA"')
  .limit(10);

// Object when configuration is already available
const query2 = await new QueryBuilder('cities', options);
```

---

## TypeScript Example: Classic Builder Pattern

Here's a comprehensive TypeScript implementation showing the full pattern:

```typescript
// Product being built
class Product1 {
  public parts: string[] = [];

  public listParts(): void {
    console.log(`Product parts: ${this.parts.join(', ')}\n`);
  }
}

// Builder interface
interface Builder {
  producePartA(): void;
  producePartB(): void;
  producePartC(): void;
}

// Concrete builder implementation
class ConcreteBuilder1 implements Builder {
  private product: Product1;

  constructor() {
    this.reset();
  }

  public reset(): void {
    this.product = new Product1();
  }

  public producePartA(): void {
    this.product.parts.push('PartA1');
  }

  public producePartB(): void {
    this.product.parts.push('PartB1');
  }

  public producePartC(): void {
    this.product.parts.push('PartC1');
  }

  // Return the product and reset for next build
  public getProduct(): Product1 {
    const result = this.product;
    this.reset();
    return result;
  }
}

// Director class
class Director {
  private builder: Builder;

  public setBuilder(builder: Builder): void {
    this.builder = builder;
  }

  // Build minimal product
  public buildMinimalViableProduct(): void {
    this.builder.producePartA();
  }

  // Build full-featured product
  public buildFullFeaturedProduct(): void {
    this.builder.producePartA();
    this.builder.producePartB();
    this.builder.producePartC();
  }
}
```

### Usage Examples

```typescript
// Setup
const director = new Director();
const builder = new ConcreteBuilder1();
director.setBuilder(builder);

// Build minimal product
console.log('Standard basic product:');
director.buildMinimalViableProduct();
builder.getProduct().listParts();
// Output: Product parts: PartA1

// Build full product
console.log('Standard full featured product:');
director.buildFullFeaturedProduct();
builder.getProduct().listParts();
// Output: Product parts: PartA1, PartB1, PartC1

// Custom product (without director)
console.log('Custom product:');
builder.producePartA();
builder.producePartC();
builder.getProduct().listParts();
// Output: Product parts: PartA1, PartC1
```

---

## Real-World Use Cases

### 1. HTTP Request Builders

```javascript
const request = await new RequestBuilder('https://api.example.com')
  .method('POST')
  .header('Content-Type', 'application/json')
  .body({ user: 'John' })
  .timeout(5000);
```

### 2. Document/Report Builders

```javascript
const report = new ReportBuilder()
  .title('Monthly Sales Report')
  .section('Summary', summaryData)
  .section('Details', detailsData)
  .footer('Generated: ' + new Date())
  .build();
```

### 3. Configuration Builders

```javascript
const config = new ConfigBuilder()
  .database('localhost', 5432)
  .logging('debug')
  .timeout(30000)
  .retries(3)
  .build();
```

### 4. UI Component Builders

```javascript
const dialog = new DialogBuilder()
  .title('Confirm Action')
  .message('Are you sure?')
  .button('OK', handleOk)
  .button('Cancel', handleCancel)
  .show();
```

---

## Key Takeaways

1. **Purpose**: The Builder pattern separates complex object construction from its representation

2. **When to Use**:
   - When constructors become too complex
   - When you need multiple ways to create an object
   - When object creation requires many steps
   - When you want to hide construction complexity

3. **Components**:
   - **Product**: The object being built
   - **Builder**: Constructs the product step by step
   - **Director** (optional): Controls the building sequence

4. **Benefits**:
   - Encapsulates construction logic
   - Supports different representations
   - Provides control over construction process
   - Enables method chaining for fluent interfaces

5. **JavaScript Considerations**:
   - First-class functions allow simpler alternatives
   - Method chaining (`return this`) creates fluent APIs
   - Thenable contract enables async/await compatibility
   - Can support both chaining and object configuration

6. **Trade-offs**:
   - Increases code size with additional classes
   - May add complexity to simple cases
   - Worth it when construction logic is genuinely complex

---

## Common Patterns with Builder

### Fluent Interface (Method Chaining)

Return `this` from each builder method to enable chaining:

```javascript
class FluentBuilder {
  method1() {
    // ... do work
    return this; // Enable chaining
  }

  method2() {
    // ... do work
    return this; // Enable chaining
  }
}

// Usage
const result = new FluentBuilder()
  .method1()
  .method2();
```

### Thenable Pattern

Implement `then()` to make builder work with promises:

```javascript
class ThenableBuilder {
  then(resolve, reject) {
    // Build and return final result
    const result = this.build();
    resolve(result);
  }
}

// Usage with await
const result = await new ThenableBuilder().option1().option2();
```

### Reset Pattern

Reset builder state after getting the product for reuse:

```javascript
class ReusableBuilder {
  getProduct() {
    const result = this.product;
    this.reset(); // Clear state for next build
    return result;
  }

  reset() {
    this.product = new Product();
  }
}
```

---

## Comparison with Other Patterns

### Builder vs Factory Method

- **Factory Method**: Creates objects in one step
- **Builder**: Constructs objects step by step with configuration

### Builder vs Abstract Factory

- **Abstract Factory**: Creates families of related objects
- **Builder**: Constructs a single complex object with many options

### Builder vs Prototype

- **Prototype**: Clones existing objects
- **Builder**: Constructs new objects from scratch with specific configuration

---

## Summary

The Builder pattern is essential when dealing with complex object construction. While the classical implementation involves abstract builders and directors, modern JavaScript allows for more flexible and simpler approaches using:

- Method chaining for fluent APIs
- First-class functions for step composition
- Thenable contract for async compatibility
- Hybrid approaches supporting both chaining and object configuration

The key is to balance between following the pattern formally and adapting it to JavaScript's strengths, always keeping the code maintainable, testable, and easy to use.

**Remember**: Use the Builder pattern when construction logic is genuinely complex. For simple cases, a regular constructor or factory function is often sufficient and more appropriate.
