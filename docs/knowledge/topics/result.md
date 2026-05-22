---
id: result
title: Result<T, E>
spec: #82-resultt-e
aliases: [Result, Ok, Err, try!, try?, catch!, error_handling, parse_error, adt]
---
## Result<T, E>

Result<T, E> is the explicit error-return track: an ADT enum with Ok(T) and Err(E).

### ADT shape
`Result<T, E>` is a prelude ADT enum: `Ok(T)` carries success, `Err(E)` carries failure. Prefer an ADT enum for `E` when the failure causes are enumerable, so `match` can check exhaustiveness.

### Payload enum form
```xray
// positional payload
enum Result<T, E> {
    Ok(T),
    Err(E),
}

// named-field payload (recommended for readability)
enum NetEvent {
    Connected,
    Disconnected(reason: string),
    DataReceived(bytes: Bytes),
    Error(code: int, message: string),
}

// state machine
enum ConnState {
    Idle,
    Connecting(retry: int),
    Connected(peer: string, since: int),
    Failed(reason: string),
}

// AST nodes
enum Expr {
    Number(int),
    Binary(op: string, left: Expr, right: Expr),
    Call(name: string, args: Array<Expr>),
}
```

### Common methods
- `isOk()` / `isErr()` inspect the variant
- `ok()` returns `T?`; `err()` returns `E?`
- `unwrap()` extracts `Ok` or throws on `Err` when `E` is an `Exception` subtype
- `unwrapOr(default)` and `unwrapOrElse(handler)` provide fallback values
- `map(fn)` transforms `Ok`; `mapErr(fn)` transforms `Err`; `andThen(fn)` chains Results

### Bridge keywords
- `try! result` early-returns `Err` inside a Result-returning function
- `try! result` throws when crossing from Result into the exception track and `E` is an `Exception` subtype
- `try? result` converts `Err` to `null` and discards the cause
- `catch! expr` catches exceptions and converts them to `Result.Err`
