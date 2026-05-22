---
id: testing
title: Testing
aliases: [test, assert, assert_eq, unit_test]
---
## Testing

```xray
@test
fn test_add() {
    assert_eq(1 + 1, 2)
    assert_true(3 > 2)
    assert_false(1 > 2)
    assert_ne("a", "b")
}

@test(skip)
fn test_todo() { return }  // skipped

@test(timeout: 5000)
fn test_slow() { return }  // 5s timeout

// Assert that a function throws
assert_throws(fn() { throw new Exception("error") })
```

Run with: `xray test <file_or_dir>`
