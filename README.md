# State Transition Language

[![Gitter](https://badges.gitter.im/eyecikjou567/statra.svg)](https://gitter.im/eyecikjou567/statra?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

## Goals

* Provide a clean and compact language to write smart contracts
* Security by design, Simplicity by design
* Easily understandable
* Compiles into readable Solidity Code
* Provide Macros as a Standard Library
* Representation as a Deterministic Finite Automaton

## Why?

* Imperative Programming is "unsafe" in the smart contract space
* Functional Programming is "hard"
* State Machines are sufficient in representing common problems in smart contracts
* Formal Validation is easier
* Securing State Machines against problems like Reentrancy is easy

## Status

At the moment, Statra is still an Work-In-Progress Language Specification.

Once the Specification is fleshed out, work on the compiler can begin. As a goal,
the compiler should produce human-readable solidity code, optionally EVM code.

After finishing the compiler, focus goes to creating a standard macro Library
and IDE support/integration.

## Links

[Language Specification](/statra.md)

Compiler Specification (TBD)

Macro Standard Library (TBD)

IDE Integration (TBD)
