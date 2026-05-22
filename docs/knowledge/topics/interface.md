---
id: interface
title: Interface
spec: #55-interface-与-implements
aliases: [interfaces, implements]
---
## Interface

### Interface declarations and implementations
```xray
interface Shape {
    area() -> float
    perimeter() -> float
}

// Interface method return types may be omitted (default ())
interface Greeter {
    greet(name: string)             // same as greet(name: string) -> ()
    log()                           // no parameters, no return value
}

class Circle implements Shape {
    radius: float
    constructor(r: float) { this.radius = r }
    area() -> float { return 3.14 * this.radius * this.radius }
    perimeter() -> float { return 6.28 * this.radius }
}

// Implement multiple interfaces
class Logger implements Shape, Greeter {
    radius: float
    constructor(r: float) { this.radius = r }
    area() -> float { return 3.14 * this.radius * this.radius }
    perimeter() -> float { return 6.28 * this.radius }
    greet(name: string) { print("hello,", name) }
    log() { print("logging") }
}

fn describe(s: Shape) -> string {
    return "area=${s.area()}, perimeter=${s.perimeter()}"
}
```

### Nominal typing
```xray
interface Drawable { draw() -> () }

class Square implements Drawable {        // explicit implements required
    draw() { print("square") }
}

class Wrong {
    draw() { print("wrong") }
}

fn render(d: Drawable) { d.draw() }
render(new Square())     // OK
// render(new Wrong())   // compile error: Wrong is not Drawable
```
