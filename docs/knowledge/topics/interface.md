---
id: interface
title: Interface
aliases: [interfaces, implements]
---
## Interfaces

```xray
interface Shape {
    area() -> float
    describe() -> string
}

class Circle implements Shape {
    radius: float
    constructor(r: float) { this.radius = r }
    area() -> float { return 3.14159 * this.radius * this.radius }
    describe() -> string { return "Circle(r=${this.radius})" }
}
```
