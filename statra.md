# Statra Design Document

## Goals

The State Transition Language, short Statra aims to be a safe somewhat-functional language for Ethereum contracts, concentrating on States and their Transitions, enforcing assertion inside states and providing a clean and easily understandable syntax.

## Syntax

```
machine StatraToken {
  var {
    int tokenSupply = 0
    int etherSupply = 0
  }

  invar {
    must etherSupply == tokenSupply * tokenPrice
  }

  const int tokenPrice = 10 * finney
  const int tokenDigits = 2

  mapping (address => int) tokenOwners

  invar must sum(tokenOwners.value) == tokenSupply

  assert {
    SupplyCheck {
      etherSupply == tokenSupply * tokenPrice
    }

    EtherCheck {
      etherSupply == this.value
    }

    HasFunds(int fund) {
      tokenOwners[msg.sender] >= fund
    }
  }

  trigger {
    buyTokens()
    sellTokens(int)
  }

  state {
    Token default {
      SupplyCheck
    }

    SendTokens ephemeral {
      SupplyCheck
    }
  }

  transition {
    Token {

      on buyTokens {
        increase(msg.value, tokenPrice)
      } -> SendTokens

      check EtherCheck

      on sellTokens(tokens) check hasFunds(tokens) {
        decrease ( tokens, tokenPrice )
        tokenOwners[msg.sender] -= tokens
        unsafe {
          this.send(tokens)
        }
      } -> Token

    }

    SendTokens{

      check EtherCheck

      on all {
        tokenOwners[msg.sender] = msg.value * tokenPrice
      } -> Token

    }
  }

  macro  {
    increase ( value, price ) {
      tokenSupply += value
      etherSupply += value * price
    }

    decrease ( value, price ) {
      tokenSupply -= value
      etherSupply -= value * price
    }
  }

  view {
    balance () returns (int) {
      return tokenOwner[msg.sender]
    }
  }
}
```

## Under the hood

Statra defines multiple `machines` within a contract. At the end of each Transaction a Statra Contract is in a well-defined and assert-checked state.

To change the state the programmer defines `transition` which can have conditions (`asserts`) and triggers. A trigger is normally a function call, conditions are shared as `asserts`.

Variables and constants have to be initialized and used. Unused Variables or Constants cause compilation error.

A `transition` defines a Begin-State, an End-State, any number of triggers and any number of conditions. Transitions must be unambiguous, meaning a transition cannot happen at the same time as any other; all transitions *from* a state must be mutually exclusive. A Transition is like a function.

A contract runs as long as it's statemachine finds transitions that can be traversed as their conditions are true. Initially triggers are used to find "called functions" in transitions, they are represented in contract code as non-throw asserts.

Asserts are not part of the conditions needed to traverse a transition, they are executed before the transition and can throw if an unsafe state is found.

A `macro` defines inline code, the compiler will simply inject the code by renaming variables. Each macro-injection has it's own base-id for internal variables to avoid conflicts during compilation time.

An `assert` defines conditions that must be true otherwise it throws. Each line is it's own condition unless the line ends with a boolean symbol like `&&`.

Empty Asserts evaluate to `true` and do not throw.

As `condition` defines conditions that can be true or false. A `trigger` is a form of condition.

Inside a Transition normal code similar to Solidity is found with functional differences. Since Statra is meant to be sideeffect-less, the act of sending ether or otherwise manipulating the outside world of the state machine (even other statemachines within the contract) must be wrapped within an `unsafe` statement.

`unsafe` code blocks are run under a mutex to prevent reentrancy problems. Macros cannot contain `unsafe` code and there can be at maximum one `unsafe` code block per transition. The unsafe code must be the last part of a transition and it cannot be the only code of a transition.

`view` are purely functional operations. They cannot modify the state or any variables, it can use temporary scope-local variables.

To work each Statra Contract *must* define the following: at least one `state`, at least one `assert`, at least one `transition`

If the compiler finds even one violation of any of the above rules, compilation fails.

A Statra compiler *cannot* show a warning and compile anyways. **ALL** Warnings are Errors. Warnings *cannot* be disabled. To be Statra-Compliant a compiler **must** check if transitions are consistent.

## Minimal Statra Contract

A Minimal Contract according to the rules looks as follows:

```
machine {
    assert a { }
    state s { }
    transition s->s { on first check a {} }
}
```

This Contract has an unnamed state with an empty assert and a transition to the state itself at any function call.

## Pieces of Statra

A Statra Contract has the following structures

1. `var`
2. `const`
3. `mapping`
4. `invar`
5. `assert`
6. `trigger`
7. `state`
8. `transition`
9. `macro`
10. `view`

Each section can contain 1 definition, which can be grouped together with `{}` to
use multiple definitions at once.

Example:

```
trigger someTrigger()
//or
trigger {
  someTrigger(),
  otherTrigger(),
}
//or
trigger { someTrigger(), otherTrigger() }
```

## var - Few Types, Much Safety, Wow

Variables are the data of a machine.

As they are considered in-universe for the machine, it can interact with them and produce side-effects without every worrying about unsafety (outside of asserts)

A variable has one of a few types; `bool`, `int`, `opint`, `address`, `callable`, `bytes`, `string` and `hash`

### Booleans

Booleans are either 0, thusly false, or true.

### Integers

All Integers are represented as int256 within the machine.

### OPIntegers

Overflow Protected Integers use checks to protect against over- and underflows, throwing if such things occur.

### Addresses

To protect the developer and user, addresses are strictly different types.

A int256 cannot be converted into an address type within the Statra language but this feature may be provided as macro at your own risk.

### Callables

Callables are a subtype of an address. They can be used interchangably inside a statemachine and represent any external contract with an ABI.

There are predefined callables; `this`, which represents the current contract, `tx` which represents the transaction origin, `msg` which is the current tx.

Callables are the only type that has functions in the traditional sense, which is necessary to interface with other contracts.

Callables can *only* be used inside an `unsafe` block.

Example:

```
variable address externalContract = 0xThisIsNotAnAddress

transition s->s {
  on: first {
    a = 1
    b = "test"
    unsafe {
      externalContract.doStuff(a, b)
    }
  }
}
```

### Strings and Bytes

Statra has basic implementation of strings. It can check for string equivalency, concat and substring. It is prefered to handle the actual string implementation off chain and use the `hash` type.

Bytes work similarly to strings and the two types are interchangable. Bytes allow addressing of single characters whereas strings do not, but strings allow for easy equality checks.

Either provides a few upsides and downsides.

### Hash

A hash is a SHA3-256 value. It cannot be directly casted into any other type.

It has a special property in that a equality check will compare against **the hash** of the other side. (Comparing two hashes from external sources is quite silly and comparing two hashes stored internally is quite sillier)

## const - Always the same

Constants are predefined values.

They have the same types as variables but that value is backed into the contract code and cannot be changed retrospectively.

## mapping - General Purpose Arrays

Mappings are access like arrays with an key. This returns the value of the key.

Internally they consist of a hashmap, where the SHA3-256 of the "key" is the *key* to access the value, except addresses which are simply padded with zeroes.

There are three predefined mappings: `msg`, `this` and `tx`.

`msg` represents the current transaction, `tx` the transaction stack, similar to solidity. The compiler provides a translation so that they can be used similarly but all data is accessed by hashing the solidity equivalent with SHA3-256.

`this` is the state of the current contract, ie it's holdings of Ether and Gas or Address.

They are not handled under variables and represent a closed-off type in the compiler, thusly they cannot be represented as constants.

## invar - Safety in Numbers

Invariants are safety measures. The compiler will check statically if the Invariants
are violated. If the compiler can't or notices a violation, a compiler error is
generated.
They are defined with the keyword `invar` followed by any of these:

* `must` - on any boolean expression - Expression must remain true at any time
* `any` - on any boolean expression of a mapping - Atleast one element of the mapping must remain true

## assert - Assertions and Truth

Assertions are special kinds of triggers. They work exactly like conditional triggers but cannot work like functional triggers.

An assert can only be true or false, if it is called and evaluates to false, the contract throws.

A false assertion is viewed as an inconsistent state of the machine and rather than continuing in an unsafe state, the machine terminates and rolls back the transaction via `throw`

## trigger - Getting Triggered

The standard trigger is usually a functional trigger. Sometimes conditional triggers may be used.

A contract will traverse states until no further trigger is found. The compiler will enforce mutual exclusivity of
triggers by checking if two transitions *could* logically be triggred at the same time. This does not mean the contract
could ever enter such state, but this makes checking on compiler-side easier.

A one-way exclusivity is enough; the only condition is that under all circumstances only one transition can ever be triggered, no matter how insane the state is.

As not transitioning out of a state is perfectly legal, the compiler won't enforce to always use a transition.

If more than 1 transition is found, the contract throws.

If exactly 1 transition is found, the contract will move into it and check the asserts. If the assert evaluates to false, the contract throws.

If 0 transitions are found, execution terminates gracefully.

There is no limit on how often a trigger can be checked, so making infinite loops is possible and should be check against. (At best draw your Statra Contract on paper and see if it has any potential loops)

### Predefined Triggers

The trigger `*` and `first` is true when the contract is initialized and only then.

The trigger `@` and `last` is true if the last send/call was successfull.

The trigger `all` is always true. Only the last defined transition of a state is allowed to utilize this trigger and at maximum one can be used per state. It is generally recommended to refrain from using this trigger at all.

## state - And its enemies

States are checkpoints in the Statra language. With asserts they assure the integrity of the data in the machine and then they select a transition.

There are currently six modes of states; nondefault, default, noncritical, critical, normal and ephemeral.

A state is a mixture of atleast three modes.

There can only be one default state, it is the state in which the machine initializes. All other state are non-default.

Within Normal States the machine can terminate, ephemeral states cannot terminate.

Noncritical States allow reentrancy, critical states block reentrancy. A state is by default critical.

Each tag has an opposing tag that means the opposite of the other. See; normal and ephemeral, noncritical and critical.

By default all states are nondefault, critical and normal.

A state may define a few global assertions. These global asserts are executed when the machine is activated and when a state is entered.

Be aware that as a non-functional language, a state that executes a transition **will have side effects and this is intended**. The language is neither stateless nor without side effects, this makes programming a bit easier to think and reason about.

### State Tags

A state may be tagged with any of it's modifiers. If two conflicting modifiers are found the compiler shows an error and terminates.

```
state DefaultState default noncritical {}

state OtherState nondefault critical ephemeral {}
//equivalent to
state OtherState {}
```


## Conditionals

You may have noticed that the example Statra Token Contract does not feature conditionals.

That's because Statra has no concept of loops or ifs or switch-instructions, similar to common State Machines.

Instead, States and Transitions can be used to build these things. This makes it safer to loop or switch-case as each decision is always asserted to be correct.

## View

A view can define scope-local variables via the `variable` syntax.

These are destroyed when the function exits.

View are purely read-only and cannot change the state of the machine.

They can return value and are the only way of returning values from the inside of the machine outside of public variables.
