---
layout: default
title: Proxy basics
nav_order: 2
---

# Proxy Basics

**Warning:** this in-depth proxy research assumes you are familiar with how `delegatecall` works and how the context of the calling contract is used with the logic of the called contract. If you are unfamiliar with how `delegatecall` works, consult in-depth explanations such as [this one](https://medium.com/coinmonks/delegatecall-calling-another-contract-function-in-solidity-b579f804178c), [this one](https://blog.openzeppelin.com/ethereum-in-depth-part-1-968981e6f833/), or [this one](https://docs.soliditylang.org/en/latest/introduction-to-smart-contracts.html#delegatecall-and-libraries).

## What is a proxy and why are they useful?

Contract code on a blockchain is immutable, meaning it cannot be altered. This is a very useful property, one of the key benefits of blockchains. But code that cannot be changed also has the downside of being hard or impossible to update. Updates can be good, whether they provide users with new features or fix bugs in the original code. Proxies provide a way for smart contract code to be updated. There are two main approaches for proxies:

1. A lightweight "entry point" contract with a `delegatecall` that uses the logic of another contract. The first contract stores the address of the logic contract in a variable that can be changed, so when a new logic contract is deployed at a new address, the "entry point" can remain unchanged because only a state variable used for the `delegatecall` operation needs changing.
2. A factory contract creates the logic contract with CREATE2. The logic contract contains a `selfdestruct` (normally protected so only an admin or authorized account can call it) to allow the contract to be destroyed. When the contract is destroyed, the factory can use CREATE2 to deploy a new contract with different code at the same address as the old destroyed contract.

## Can smart contracts be upgraded without proxies?

Yes, contract logic can be designed in such a way to allow a protocol to be upgraded without using delegatecall or CREATE2. One alternative approach is called the "data separation" pattern in [this Trail of Bits blog post](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/) and another is [contract migration](https://blog.trailofbits.com/2018/10/29/how-contract-migration-works/). The way the "data separation" pattern works is by using a primary contract that serves as the entry point, but this contract doesn't contain complex logic besides calling external contracts. These calls to external contracts only use `call`, not `delegatecall`. The addresses of the external contracts are stored in state variables that have setter functions allowing only the owner to change the value of this state variable to a new contract address. This allows the external contracts to be upgraded and the entry point contract can update the state variables to point to the new versions contract addresses. One example of this "data separation pattern" reviewed by yAcademy previously is [Tribe Turbo](https://github.com/fei-protocol/tribe-turbo/blob/main/src/TurboMaster.sol).