---
layout: default
title: History of Callcode and Delegatecall
nav_order: 3
parent: Proxies Deep Dive
---

# History of Callcode and Delegatecall

## Callcode opcode

If you read the [solidity docs about `selfdestruct`](https://docs.soliditylang.org/en/latest/introduction-to-smart-contracts.html#deactivate-and-self-destruct), you will find a note saying that `delegatecall` and `callcode` can provide a way to `selfdestruct` a contract that does not contain the `selfdestruct` opcode. What is `callcode` and how similar is it to `delegatecall`? `callcode` is the same as `delegatecall` but it does not propagate `msg.sender` or `msg.value`. `callcode` was deprecated [in Solidity 0.5.0](https://docs.soliditylang.org/en/latest/050-breaking-changes.html#functions). [EIP-2488](https://eips.ethereum.org/EIPS/eip-2488) proposed to fully deprecate `callcode` but the EIP did not get far. Even though you will still see the callcode opcode (0xF2) on [evm.codes](https://www.evm.codes/), if you write a contract with the `callcode` opcode and a solidity version of 0.5.X or newer, you will get an error preventing the contract from compiling with the message `TypeError: "callcode" has been deprecated in favour of "delegatecall".` Similar cases of deprecated opcodes are found in this table for [SWC-111](https://swcregistry.io/docs/SWC-111).

## Delegatecall opcode

[EIP-7](https://eips.ethereum.org/EIPS/eip-7) introduced the `delegatecall` opcode as a bug fix to `callcode`, which was introduced in the [Homestead hard fork in 2016](https://ethereum.org/en/history/#homestead). Because proxy development only started gathering momentum in 2017, `callcode` was never used for proxies, which is why `delegatecall` is the opcode that we all associate with proxies today.

## Comparison between Callcode and Delegatecall

Below is a diagram comparing proxies built with `callcode` and `delegatecall` (*Note: it is not recommended to use `callcode` in proxies*).

![Comparison with Proxy](../../assets/images/Comparison_Callcode_Delegatecall.png)

Both `callcode` and `delegatecall` have the same behavior on storage. That is, both of them can execute the implementation's code and perform operations with proxy's storage. The difference between them is in `msg.value` and `msg.sender`. In `callcode`, `msg.value` can be customized to hold a new value in the implementation contract and `msg.sender` is changed to Proxy's address. In `delegatecall`, both `msg.value` and `msg.sender` remain the same in the proxy and implementation contracts.
