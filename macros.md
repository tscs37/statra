# Statra Macros & Macro Library

A macro is an inline shortcut meant to reduce the amount
of code that needs to be written.

They can behave like functions that are exclusively
pass-by-reference. But keep in mind that the compiler
will just copy-paste them into your code, replacing
the necessary names accordingly.

Any variable name that is *not* published via parameter is hidden from the rest of the code.

Macros are localed inside namespaces, which default to `.` if nothing is specified.

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
    .reduceBalance(owner, -100)
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

## Project Local Macros


Project local macros can define full Statra machines with parameters to replace their public names.

`send.statra-macro`

```
macro reduceMap(startState, endState, triggers, asssertions, map, key, value) {
  transition startState {
    on triggers check assertions {
      map[key] -= value
    } -> endState
  }
}
```

The final project can use this outside of any definition like this:

```
.reduceMap(Standard, BoughtItem, OnBuyItem, HasFunds, Funds, msg.sender, msg.value)
```

Project Macros can a namespace depth of up to 1.

Example:

```
.helper.reduceBuyerBalance(msg.sender, msg.value, item)
```

`helper.statra-macro`

```
namespace "helper"

macro reduceBuyerBalance(sender, value, item) {
  //CODE
}
```
## Global Macros

Global Macros are the most extensive form of Macro. They can include raw EVM code and other language code, which for the Standard library will be Solidity code.

*Note:* Keep in mind that if you want to compile Statra into something other than Solidity, you will need to get or make a port of the Standard library.

Global Macros also support namespaces of any depth.

```
namespace "stdlib.hash"

macro sha3(value) {
  ;sha3($value);
}
```

This macro would be called thusly:

```
transition someState {
  on all {
    bet[msg.sender] = .stdlib.hash.sha3(msg.data)
  } -> someOtherState
}
```

## Inline Solidity

Inline Solidity Code is marked with `;` on both ends of a line.

Multiline Solidity Code can be started with a `;;;` and ended with `;;;`
