---
layout: default
title: Proxy basics
nav_order: 2
---

# Proxy Basics

## What is a proxy and why are they useful?

Contract code on a blockchain is immutable, meaning it cannot be altered. This is a very useful property, one of the key benefits of blockchains. But code that cannot be changed also has the downside of being hard or impossible to update. Updates can be good, whether they provide users with new features or fix bugs in the original code. Proxies provide a way for smart contract code to be updated. There are two main approaches for proxies:

1. A lightweight "entry point" contract with a `delegatecall` that uses the logic of another contract. The first contract stores the address of the logic contract in a variable that can be changed, so when a new logic contract is deployed at a new address, the "entry point" can remain unchanged because only a state variable used for the `delegatecall` operation needs changing.
2. A factory contract creates the logic contract with CREATE2. The logic contract contains a `selfdestruct` (normally protected so only an admin or authorized account can call it) to allow the contract to be destroyed. When the contract is destroyed, the factory can use CREATE2 to deploy a new contract with different code at the same address as the old destroyed contract.

## Can smart contracts be upgraded without proxies?

It is possible to design the logic in such a way to allow a protocol to be upgraded without using delegatecall or CREATE2. The alternative approach is called the "data separation pattern" in [this Trail of Bits blog post](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/). The way this pattern works is by using a primary contract that serves as the entry point, but this contract doesn't contain much logic other than calling external contracts. These calls to external contracts only use `call`, not `delegatecall`. The addresses of the external contracts are stored in state variables that have setter functions allowing only the owner to change the value of this state variable to a new contract address. This allows the external contracts to be upgraded and the entry point contract can update the state variables to point to the new versions contract addresses. One example of this "data separation pattern" reviewed by yAcademy previously is [Tribe Turbo](https://github.com/fei-protocol/tribe-turbo/blob/main/src/TurboMaster.sol).