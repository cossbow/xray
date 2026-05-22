---
id: class
title: Class
aliases: [classes, oop, inheritance, extends, override, constructor]
---
## Classes

```xray
class Animal {
    name: string
    constructor(name: string) { this.name = name }
    speak() -> string { return "${this.name} says something" }
}

class Dog extends Animal {
    constructor(name: string) { super(name) }
    override speak() -> string { return "${this.name} says woof" }
}
```

### Features
- `constructor()` — called via `new ClassName()`
- `extends` — single inheritance
- `override` — required keyword when overriding methods
- `static` — class-level methods
- `private` — field visibility
- `this` — current instance reference
- `super` — parent class reference
