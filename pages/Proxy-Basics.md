---
layout: default
title: Proxy Basics
nav_order: 2
---

# Proxy Basics

**Warning:** this research assumes you are familiar with how `delegatecall` works and how the logic of the callee contract applies the caller's context. If you are unfamiliar with how `delegatecall` works, consult in-depth explanations such as [this one](https://medium.com/coinmonks/delegatecall-calling-another-contract-function-in-solidity-b579f804178c), [this one](https://blog.openzeppelin.com/ethereum-in-depth-part-1-968981e6f833/), or [this one](https://docs.soliditylang.org/en/latest/introduction-to-smart-contracts.html#delegatecall-and-libraries).

## Why do I need a proxy and how do I use it?

By design, contract code on a blockchain is **immutable**. Though a key feature, it leads to difficulty when considering upgradeability. Newer entrants may wonder why "upgrading" on a blockchain is necessary. Inevitabilities requiring code changes still remain, including: bug fixes, patches, optimizations, feature releases, etc.

The **Proxy** or **Proxy Delegate** pattern allows for this upgradeability. Put simply, there are two main approaches for proxies:

1. A lightweight, entry-point caller contract `A` to use the logic of a callee contract `B` through `delegatecall`:
- `A` stores the address of `B` in a state variable that can be changed. 
- If an upgrade is desired, a new callee contract `C` is deployed at a different address.
- `A` updates the state variable to point to the new callee contract, `C`.

2. A factory contract `A` that creates a logic contract `B` with opcode `CREATE2`:
- `B` contains a `selfdestruct` to allow the contract to be destroyed. 
  - *Note: This is normally protected so only an admin or authorized account can call it.*
- If an upgrade is desired, `B` is destroyed, and the factory can use `CREATE2` to deploy a new contract at the same address as `B`.

Additional reading for the proxy pattern can be found [here](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies), and [here](https://fravoll.github.io/solidity-patterns/proxy_delegate.html).

## Can smart contracts be upgraded without proxies?

Yes, contract logic can be designed in such a way to allow a protocol to be upgraded without using `delegatecall` or `CREATE2`. An alternative approach is called the "data separation" pattern in [this Trail of Bits blog post](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/) and another is [contract migration](https://blog.trailofbits.com/2018/10/29/how-contract-migration-works/). 

### Data Separation Pattern

This pattern works by using a primary contract `A` to serve as the entry-point, and doesn't contain complex logic besides calling external contracts. Calls to external contracts **only use** `call`, **not** `delegatecall`. 

Addresses of the external contracts are stored in state variables that have restrictive setter functions, allowing only the owner to ever mutate their state. This allows `A` to update the state variables to point to new contract addresses whenever an upgrade is desired.

One example of this "data separation pattern" reviewed by yAcademy previously is [Tribe Turbo](https://github.com/fei-protocol/tribe-turbo/blob/main/src/TurboMaster.sol).
