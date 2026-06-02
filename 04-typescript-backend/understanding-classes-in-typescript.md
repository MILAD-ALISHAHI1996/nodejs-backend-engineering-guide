# TypeScript Classes: A Complete Guide from Beginner to Advanced

## Introduction

If you've ever worked with users, products, invoices, orders, or repositories, you've already worked with concepts that are perfect candidates for classes.

Many developers learn TypeScript classes by memorizing syntax. The problem is that syntax is easy to forget.

The goal of this guide is different.

Instead of memorizing keywords, you'll understand:

* Why classes exist
* What problems they solve
* When to use them
* How they are used in real-world applications

By the end of this article, you'll understand the core concepts behind TypeScript classes and the object-oriented principles used in production applications.

---

# Why Do Classes Exist?

Imagine you're building an e-commerce application.

Without classes, you might create products like this:

```ts
const product1 = {
  name: "Laptop",
  price: 1500,
};

const product2 = {
  name: "Keyboard",
  price: 120,
};

const product3 = {
  name: "Mouse",
  price: 50,
};
```

This works.

But what happens when you have:

* 10 products?
* 100 products?
* 10,000 products?

You need a blueprint.

That's exactly what a class provides.

---

# Class vs Object

Think of a class as an architectural blueprint.

A blueprint is not a house.

It only describes how houses should be built.

```ts
class Product {}
```

This is only a blueprint.

No product exists yet.

To create a real product, we need an object.

```ts
const product = new Product();
```

Now we have an actual instance in memory.

A simple way to remember:

* Class = Blueprint
* Object = Real thing created from the blueprint

---

# What Is an Instance?

When an object is created from a class, it becomes an instance of that class.

```ts
const product = new Product();
```

The variable `product` is:

* An object
* An instance of Product

```ts
product instanceof Product;
```

Returns:

```ts
true
```

---

# Properties

Properties describe the data of an object.

A product might have:

* Name
* Price
* Stock Quantity

```ts
class Product {
  name: string;
  price: number;
  stock: number;
}
```

These values describe the state of a product.

---

# Methods

Methods describe behavior.

A product can:

* Calculate discount
* Update stock
* Check availability

```ts
class Product {
  checkAvailability() {
    return true;
  }
}
```

Properties describe data.

Methods describe actions.

---

# Constructor

A blueprint becomes useful when we can provide initial data.

```ts
class Product {
  name: string;
  price: number;

  constructor(
    name: string,
    price: number
  ) {
    this.name = name;
    this.price = price;
  }
}
```

Creating a product:

```ts
const laptop =
  new Product("Laptop", 1500);
```

The constructor runs automatically when the object is created.

Its main purpose is initialization.

---

# Understanding this

Inside a class:

```ts
this
```

refers to the current object.

```ts
class Product {
  name: string;

  constructor(name: string) {
    this.name = name;
  }
}
```

For:

```ts
const laptop =
  new Product("Laptop");
```

`this` refers to `laptop`.

For:

```ts
const keyboard =
  new Product("Keyboard");
```

`this` refers to `keyboard`.

---

# Access Modifiers

TypeScript allows us to control visibility.

## public

Accessible everywhere.

```ts
public name: string;
```

## private

Accessible only inside the class.

```ts
private passwordHash: string;
```

## protected

Accessible inside the class and its children.

```ts
protected createdAt: Date;
```

These modifiers help enforce boundaries and prevent misuse.

---

# readonly

Some data should never change.

Examples:

* User ID
* Invoice Number
* Product Code

```ts
readonly id: number;
```

Once assigned, it cannot be modified.

---

# Getter and Setter

Sometimes we want a property to look simple while hiding logic behind it.

Example:

```ts
class Product {
  private _price = 0;

  get price() {
    return this._price;
  }

  set price(value: number) {
    if (value < 0) {
      throw new Error(
        "Invalid price"
      );
    }

    this._price = value;
  }
}
```

This allows validation while keeping a clean API.

---

# Static Members

Most properties belong to objects.

Static members belong to the class itself.

```ts
class Product {
  static vatRate = 10;
}
```

Access:

```ts
Product.vatRate;
```

No object is required.

Static members are useful for:

* Configuration values
* Utility methods
* Shared counters

---

# Inheritance

Inheritance allows a class to reuse another class.

```ts
class BaseEntity {
  id = 0;
  createdAt = new Date();
}
```

```ts
class Product
  extends BaseEntity {

  title = "";
}
```

Product automatically inherits everything from BaseEntity.

This reduces duplication and promotes reuse.

---

# The super Keyword

When a child class needs the parent class to initialize itself, we use `super()`.

```ts
class BaseEntity {
  constructor(
    public id: number
  ) {}
}
```

```ts
class Product
  extends BaseEntity {

  constructor(
    id: number,
    public title: string
  ) {
    super(id);
  }
}
```

Think of `super()` as:

> Ask the parent to do its part first.

---

# Abstract Classes

Sometimes a class should only act as a blueprint.

```ts
abstract class PaymentGateway {
  abstract pay(): void;
}
```

Creating an instance directly is not allowed.

```ts
new PaymentGateway();
```

This produces an error.

Only child classes can implement the required behavior.

---

# Final Thoughts

Classes are not just syntax.

They provide:

* Structure
* Reusability
* Encapsulation
* Abstraction
* Extensibility

Mastering these concepts makes it easier to understand:

* Interfaces
* Dependency Injection
* Design Patterns
* Clean Architecture
* NestJS

In the next article, we'll explore Interfaces, Implements, Polymorphism, and the architectural patterns that make TypeScript applications scalable.
