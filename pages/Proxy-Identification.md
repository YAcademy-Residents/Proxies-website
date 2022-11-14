---
layout: default
title: Proxy Identification Guide
nav_order: 1
parent: Security Guide to Proxy Vulns
---

# Proxy Identification Guide

## Basic delegatecall Proxy Identifiers

- Does not need to follow EIP-1967. The proxy may instead use a regular storage variable to store the external contract address used with `delegatecall`.
- May use [EIP-1167](https://eips.ethereum.org/EIPS/eip-1167) minimal proxy bytecode. The bytecode for the Uniswap V1 proxy is `0x3660006000376110006000366000732157a7894439191e520825fe9399ab8655e0f7085af41558576110006000f3`. For example, see contract [0xa2881a90bf33f03e7a3f803765cd2ed5c8928dfb](https://etherscan.io/address/0xa2881a90bf33f03e7a3f803765cd2ed5c8928dfb#code).

## Transparent Proxy (TPP) Identifiers

- The proxy contract fallback function will act differently if the admin (or similar privileged role) calls the contract, so look for an if statement in the proxy contract that checks if msg.sender is the admin. One example of a modifier to do this check is [in the OpenZeppelin library](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/transparent/TransparentUpgradeableProxy.sol#L45-L51).
- The proxy contract often has an upgrade function and a function to change the address of the proxy admin. These functions should be protected with an access control modifier.

## UUPS Proxy Identifiers

- Normally imports a contract with the letters "UUPS" [from OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/utils/UUPSUpgradeable.sol) or a similar library.
- Contains an initialize function
- May include the number 1822 in a comment of the contract or an imported contract (from [EIP-1822](https://eips.ethereum.org/EIPS/eip-1822), the UUPS EIP).

## Metamorphic Contract Identifiers

- Contains `selfdestruct` (or `delegatecall` to call `selfdestruct` in another contract) to allow the existing code to be removed and replaced by another contract that will be deployed at the same address.
- Often does not have a `delegatecall`
- Contract was created with CREATE2 from another contract, not an EOA. You can find the creation transaction on etherscan (manually in the "Contract Creator" information field of etherscan or automatically with [the etherscan API](https://docs.etherscan.io/api-endpoints/contracts#get-contract-creator-and-creation-tx-hash)).

## Diamond Proxy Identifiers

- Most likely has contract names that include the words "diamond", "facet", "loupe".
- Per the [EIP-2535](https://eips.ethereum.org/EIPS/eip-2535) spec, the IDiamondLoupe interface must be implemented with these four functions: `facets()`, `facetFunctionSelectors(address _facet)`, `facetAddresses()`, `facetAddress(bytes4 _functionSelector)`
- The function which contains `delegatecall` will allow the user to specify an argument to identify the facet that should be called by delegatecall. This argument specifying the facet may not necessarily be a function argument but could, for example, be `msg.data`.
