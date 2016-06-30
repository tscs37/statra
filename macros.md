# Statra Macros & Macro Library

A macro is an inline shortcut meant to reduce the amount
of code that needs to be written.

They can behave like functions that are exclusively
pass-by-reference. But keep in mind that the compiler
will just copy-paste them into your code, replacing
the necessary names accordingly.

## In-Contract Macros

In-contract macros behave like traditional functions that have
no return type and are pass-by-reference.

Example:

Given these two definitions

```
macro reduceBalance(key, value) {
  balances[key] -= value
}
```

```
transition s->s {
  on: first {
    reduceBalance(owner, -100)
  }
}
```

the compiler will create this source code:

```
transition s->s {
  on: first {
    balances[owner] -= -100
  }
}
```
