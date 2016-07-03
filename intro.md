# Introduction into Statra

Statra is at it's core an incredbily simple language.

If you look at other languages like C++, Java or Haskell, you find a high level of complexity. Not even mentioning the difficulty of writing good code in that language.

Statra is build on DFAs, Deterministic Finite Automata, machines that are extremely simple to design, implement and execute.

Imagine an FDA being like a (very simple) coffee machine. A simple coffee machine has 4 States; `Off`, `Heating`, `Making Coffee` and `Out of Water`.

Pressing the On Button puts the Coffee Machine in the `Heating` State if it has water and in the `Out of Water State`. If it's out of water, adding that will put it into the `Heating` State. Once the water has heated up, it waits for the user to press the Make-Coffee-Button which puts it in the `Making Coffee` State. Once it's finished with that it goes back to `Heating` and gets ready to make more coffee. Pressing the Off-Button always puts it in the `Off` State.

As you can see, Coffee Machines are not that incredibly complex. Smart Contracts can be described similarly. An escrow contract can be divided into several states with transitions between them. Transitions have conditions under which they can happen and there can only be one transition that is valid at all times or no transition at all.

## A Deeper Look into DFA's

FDA's, also known as State Machines, are mostly a mathematical construct used in Computer Science courses.

But they are also used in the real world. Because of their simplicity they can easily be realised as mechanical devices like a Parking Clock or, as used as example, Coffee Machines.

They don't require a lot of memory of resources to run, they trickle through their states until they reach a final end-state and terminate.

In CS Course Books you will find more detailed definitions but a simple and efficient definition shall suffice;

> In theory of computation, a branch of theoretical computer science, a **deterministic finite automaton** (**DFA**)—also known as **deterministic finite accepter** (**DFA**) and **deterministic finite state machine**—is a finite state machine that accepts/rejects finite strings of symbols and only produces a unique computation (or run) of the automaton for each input string.

(Source: https://en.wikipedia.org/wiki/Deterministic_finite_automaton)

If we go off this definition all you need to make a State Machine is the following:

* A finite number of states
* A finite number of transitions
* A finite input

Notice it says *finite* a lot, this just means that at some point that is not particularly defined, there is an end to it and you can easily count it.

So even if you build a state machine that has 10^1000 States or Transitions, it's still fits the definition of a finite state machine.

### Back to Ethereum

On Ethereum we can utilize gas to put a limit on states. Each state consumes code, each Transitions costs computation and Input costs gas, as long as Ethereum has a finite (countable) number of Ether Tokens to pay as gas, we are guaranteed to have a finite state machine.

## Differences to Classical DFA's

Usually a DFA has a start and an end state. Only when it reaches an end state it can terminate, otherwise it is considered active and running.

Although Statra has End and Start States, these are less about Terminating Computation and more about Initializing the Contract on the Blockchain and Removing it from it again (calling the `SUICIDE` opcode).

Not all contracts are meant to terminate, some are intended to run forever, so enforcing an end state is hindering at best.

Furthermore, Transitions not only have Conditions, called Triggers, they also have Checks, called Assertions.

Assertions are run once a Transition was selected and ensure integrity of execution. It is up to the programmer to include enough assertions to do this, but a valid Statra Machine has atleast 1 assertion.

Lastly, Statra Machines have unsafe operations. DFA's are usually of imperative nature but also act functional. They only interact with their own universe, they have little concept of what happens elsewhere.

So for a Statra Machine to send Transactions, it needs to enter an *"unsafe"* area. These unsafe code-lines are mutexed, so they are protected against reentrancy for example. They will also check the state of the machine when entering or leaving.

This does not rule out reentrancy at all, but it forbids a reentrant call to change the state of the machine overall. It can modify the state but when it returns, the state must be the same.

Additionally Statra has so called "ephemeral states". These are states in which a contract is not allowed to terminate, useful especially for ensuring the contract does not go into an invalid state and gets "stuck".

## Into the Code

Let's write a quick example machine to test out Statra and wet our toes.

```
machine CoffeeMachine {
  var {
    int waterTemp
    int waterLevel
  }

  trigger {
    cond isWaterHot waterTemp > 100 // 100°C = Boiling = Perfect
    cond haveWater waterLevel >= 10 // We need 10% to make a cup
    onOff()
    makeCoffee()
  }

  state {
    Off
    Heating
    MakingCoffee
    OutOfWater
  }

  transition {
    Off {
      on onOff() haveWater {} -> Heating

      on onOff() !haveWater {} -> OutOfWater
    }

    OutOfWater {
      on !haveWater {
        invertWarningLight()
      } -> OutOfWater

      on haveWater {} -> Heating

      on onOff() { switchOffWarningLight() } -> Off
    }

    Heating {
      on !isWaterHot {
        switchOnHeater()
      } -> Heating

      on makeCoffee isWaterHot {
        switchOffHeater()
      } -> MakeCoffee

      on onOff() { switchOffHeater() } -> Off
    }

    MakeCoffee {
      on onOff() {} -> Off

      on all {
        switchOnPump()
        .stdlib.time.sleep(4000) // Wait 4000ms, the time it takes us to fill a cup
        switchOffPump()
      } -> Heating
    }
  }
}
```

This code is the coffe machine we talked about in the introduction. And it won't compile.

An essential part of statra was left out here to make introduction a bit easier and it makes function calls, but let's just walk through the code.

### The Machine

In the first line we define the machine and a name for it; `CoffeeMachine`.

In Statra a Machine is a collection of data, assertions, triggers, states and transitions.

At all times, only one State can be active. This state will check each of it's transitions' triggers if they pass. If it finds none, it returns.

If a transition is found, it will execute it's asserts and throw if any fail.

Then the transition is executed and the machine switches into the next state as specified by the transition.

### Variables

```
var {
  int waterTemp
  int waterLevel
}
```

For the sake of simplicity, we refrained from reading out those values and assume they are active sensor readings.

Statra has a minimum number of types; `bool`, `int`, `opint`, `address`, `callable`, `bytes`, `string`.

Except `bytes` and `string` all of these are put into a single int256. The `opint` type will additionally check for overflows and throw if they occur.

>*Note*: The `opint` type should not be necessary if you do proper invariant checking but it exists for edge cases where invariants don't cover all cases or not well enough

`callable` is a special variant of an `address` which is only reachable inside an `unsafe` code.

Variables can be assigned values during initialization, `int waterTemp = 0`, otherwise they are initialized with their null values.

>*Note*: The null-value of `address` and `callable` is both `0x0`, make sure you set these if you need to.

### Triggers

Triggers are the main way to get around in a Statra machine.

They can be functional or conditional. The former works like a functional call to the outside, providing the Statra machine with the necessary frameworks to interop with normal solidity contracts.

A conditional trigger is a simple boolean expression that can evaluate to true or false.

Under the hood, both triggers work the same, functional triggers are just shorthand for checking the transaction data manually and provide compatible ABIs.

