---
id: testing
title: Testing
spec: #12-测试系统-testing
aliases: [test, assert, assert_eq, unit_test]
---
## Testing

### Basic test
```xray
@test
fn test_addition() {
    assert_eq(1 + 1, 2)
}

@test
fn test_with_assertions() {
    let result = compute()
    assert_eq(result, 42)
    assert(result > 0)
}
```

### Async test
```xray
@test
fn test_async_fetch() {
    let task = go fetch_data("http://...")
    let result = await task
    assert_eq(result.status, 200)
}
```

### Attributes
```xray
@test                                 // mark as a test
fn test_basic() { return }

@test(skip)                           // skip this test
fn test_wip() { return }

@native                               // C implementation
class Array<T> {
    length: int
    push(v: T)
    // no method bodies — provided by src/runtime/object/xarray_methods.c
}

@deprecated("use newAPI() instead")
fn oldAPI() { return }
```

### Run tests
Run with `xray test <file_or_dir>`.
