# 🏗 Understanding Classes in TypeScript

> Learn why classes exist, when to use them, and how they help build maintainable backend applications.

---

## 🎯 The Problem

Imagine you're building an inventory management system.

Each product needs:

* Name
* Price
* Quantity

A beginner might write:

```ts
const product1 = {
  name: "Laptop",
  price: 1500,
  quantity: 10,
};

const product2 = {
  name: "Phone",
  price: 800,
  quantity: 20,
};

const product3 = {
  name: "Monitor",
  price: 400,
  quantity: 15,
};
```

Everything looks fine.

Until your application starts growing.

Now you need:

* Validation
* Business Logic
* Reusable Functions
* Consistent Structure

Things quickly become messy.

---

## ❌ The Wrong Approach

As the project grows, every object starts duplicating logic.

```ts
const product1 = {
  name: "Laptop",
  price: 1500,
  quantity: 10,
  getTotalValue() {
    return this.price * this.quantity;
  },
};

const product2 = {
  name: "Phone",
  price: 800,
  quantity: 20,
  getTotalValue() {
    return this.price * this.quantity;
  },
};
```

Problems:

* Repeated code
* Hard to maintain
* Easy to introduce bugs
* No centralized structure

---

## 💡 The Solution: Classes

A class acts as a blueprint.

Instead of defining the same structure repeatedly, we define it once.

```ts
class Product {}
```

Then create as many objects as we want.

---

## 🛠 Creating Your First Class

```ts
class Product {
  name: string;
  price: number;
  quantity: number;
}
```

At this point we've only defined the structure.

We still need a way to assign values.

---

## 🏗 Constructors

A constructor runs automatically when a new object is created.

```ts
class Product {
  name: string;
  price: number;
  quantity: number;

  constructor(
    name: string,
    price: number,
    quantity: number
  ) {
    this.name = name;
    this.price = price;
    this.quantity = quantity;
  }
}
```

Creating objects:

```ts
const laptop = new Product(
  "Laptop",
  1500,
  10
);

const phone = new Product(
  "Phone",
  800,
  20
);
```

---

## 🤔 What Does `this` Mean?

Many beginners struggle with `this`.

Inside a class:

```ts
this.name
```

means:

> The `name` property belonging to the current object.

Example:

```ts
const laptop = new Product(
  "Laptop",
  1500,
  10
);
```

For this object:

```ts
this.name
```

becomes:

```ts
laptop.name
```

---

## ⚡ Constructor Shortcut

TypeScript provides a shorter syntax.

Instead of:

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

You can write:

```ts
class Product {
  constructor(
    public name: string,
    public price: number
  ) {}
}
```

Both produce the same result.

---

## 🛠 Adding Methods

Classes can contain behavior.

```ts
class Product {
  constructor(
    public name: string,
    public price: number,
    public quantity: number
  ) {}

  getTotalValue() {
    return this.price * this.quantity;
  }
}
```

Usage:

```ts
const laptop = new Product(
  "Laptop",
  1500,
  10
);

console.log(
  laptop.getTotalValue()
);
```

Output:

```txt
15000
```

---

## 🔒 Access Modifiers

TypeScript allows controlling access to properties.

### Public

Accessible everywhere.

```ts
public name: string;
```

### Private

Accessible only inside the class.

```ts
private password: string;
```

### Protected

Accessible inside the class and subclasses.

```ts
protected salary: number;
```

Example:

```ts
class User {
  private password: string;

  constructor(password: string) {
    this.password = password;
  }
}
```

This prevents direct access:

```ts
const user = new User("123");

console.log(user.password);
```

Result:

```txt
Error
```

---

## 🧬 Inheritance

Classes can inherit from other classes.

```ts
class Animal {
  move() {
    console.log("Moving...");
  }
}
```

```ts
class Dog extends Animal {
  bark() {
    console.log("Woof!");
  }
}
```

Usage:

```ts
const dog = new Dog();

dog.move();
dog.bark();
```

Output:

```txt
Moving...
Woof!
```

---

## 🏢 Real Backend Example

Suppose we're building an invoice system.

```ts
class InvoiceItem {
  constructor(
    public productName: string,
    public quantity: number,
    public unitPrice: number
  ) {}

  calculateTotal() {
    return this.quantity * this.unitPrice;
  }
}
```

Usage:

```ts
const item = new InvoiceItem(
  "Steel Beam",
  10,
  500
);

console.log(
  item.calculateTotal()
);
```

Output:

```txt
5000
```

This is much closer to how classes are used in real backend projects.

---

## 📦 Where Are Classes Used in Backend Development?

Classes are commonly used for:

```txt
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

Examples:

* Controllers
* Services
* Repositories
* Entities
* DTOs
* Use Cases

Frameworks like NestJS heavily rely on classes.

---

## ⚠ Common Mistakes

### Forgetting `new`

Wrong:

```ts
const product = Product(
  "Laptop",
  1500
);
```

Correct:

```ts
const product = new Product(
  "Laptop",
  1500
);
```

---

### Forgetting `this`

Wrong:

```ts
name = name;
```

Correct:

```ts
this.name = name;
```

---

### Making Everything Public

Bad:

```ts
public password: string;
```

Better:

```ts
private password: string;
```

---

## 🎤 Interview Questions

### What is a class?

A blueprint used to create objects.

### What is the purpose of a constructor?

To initialize object data during creation.

### Difference between class and interface?

A class provides implementation.
An interface only defines structure.

### What is inheritance?

The ability for one class to reuse another class's behavior.

### What are access modifiers?

Keywords that control property visibility.

---

## 📌 Key Takeaways

* Classes combine data and behavior.
* Constructors initialize objects.
* Methods define functionality.
* Access modifiers improve encapsulation.
* Inheritance enables code reuse.
* Classes are heavily used in NestJS and backend architecture.

---

## 🚀 What's Next?

After understanding classes, the next recommended topic is:

**Interfaces in TypeScript**

Because classes define implementation, while interfaces define contracts.
