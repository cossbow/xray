---
id: class
title: Class
spec: #53-class-声明
aliases: [classes, oop, inheritance, extends, override, constructor]
---
## Class

### Basic class
```xray
class Animal {
    name: string                       // field
    private _age: int = 0              // private field with default value

    constructor(name: string) {
        this.name = name
    }

    speak() -> string {
        return "..."
    }

    static create(name: string) -> Animal {
        return new Animal(name)
    }
}

let a = new Animal("Rex")
print(a.speak())
print(Animal.create("Bob").name)
```

### Inheritance
```xray
class Dog extends Animal {
    constructor(name: string) {
        super(name)                    // **must** be the first statement (derived classes only)
    }

    override speak() -> string {         // override is optional but recommended
        return "woof"
    }
}
```

### Features
- `constructor()` is called through `new ClassName(...)`
- `extends` enables single inheritance
- `override` is optional but recommended when overriding methods
- `static` declares class-level methods or fields
- `private` restricts access to the declaring class
- `this` is the current instance; `super` refers to the parent class
