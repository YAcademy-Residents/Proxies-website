---
layout: default
title: Security Guide to Proxy Vulns
nav_order: 4
has_children: true
---

# Security Guide to Proxies
{: .no_toc }

Note: If you are unsure which proxy type is in the scope of your audit or security review, see the [proxy identification guide](/pages/Proxy-Identification).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Uninitialized Proxy Vulnerability

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized)

Why do proxies need an `initialize` function when a contract constructor is called automatically? The reason is explained [here by OpenZeppelin](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat). The code in a contract's constructor is run once at deployment, but there is no way to run constructor code of the implementation contract (AKA logic contract) *in the context* of the proxy contract. Because the implementation contract must store the value of the `_initialized` variable in the proxy contract context, the constructor cannot be used for this purpose, because the implementation contract's constructor code will always run in the context of the implementation contract. This is why there exists an `initialize` function in the implementation contract - because the `initialize` call must happen through the proxy. Because the initialize call must happen as a separate step from the implementation contract deployment, there is a potential race condition that can happen that should also received attention, such as by protecting the `initialize` function with an address control modifier so only a specific `msg.sender` can initialize the function.

A specific variant of the uninitialized UUPS proxy vulnerability is found in [the OpenZeppelin library between version 4.1.0 and 4.3.2](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/security/advisories/GHSA-q4h9-46xg-m3x9). This issue is related to an [edge case of delegatecall and selfdestruct interaction](#delegatecall-with-selfdestruct-vulnerability).

### Testing procedure

To test for this vulnerability, first identify the storage slot of the [initialized state variable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/25aabd286e002a1526c345c8db259d57bdf0ad28/contracts/proxy/utils/Initializable.sol#L62) or a similar variable that the initialization function uses to revert if this is not the first time that the function is called. Try using [these tools](/pages/Proxies-Storage) to find the correct storage slot. If the OpenZeppelin private `_initialized` variable from Initializable.sol is used, a `_initialized` value of zero means the contract has not been initialized while a `_initialized` value of 1 means the contract has been initialized.

A simplistic check that has a low false-positive rate with a high false-negative is to check if the implementation contract has a non-empty constructor. Because any storage values set in the constructor of the implementation contract will not be used when the implementation contract is called through the proxy contract, a constructor can indicate a case of initialization values not getting set as expected.

Slither has a `slither-check-upgradeability` tool that has [several initializer issue detectors](https://github.com/crytic/slither/wiki/Upgradeability-Checks).

### Hacks

- [Parity Wallet](https://www.parity.io/blog/a-postmortem-on-the-parity-multi-sig-library-self-destruct/) and [OpenZeppelin writeup](https://blog.openzeppelin.com/on-the-parity-wallet-multisig-hack-405a8c12e8f7/) (note: the contract was initialized but had no protection against a second call to `initialize()`)

### Bug Bounties

- [Wormhole](https://medium.com/immunefi/wormhole-uninitialized-proxy-bugfix-review-90250c41a43a) ($10 million bounty)
- [Arbitrum Nitro](https://medium.com/@0xriptide/hackers-in-arbitrums-inbox-ca23272641a2) (400 ETH bounty)
- [Harvest](https://medium.com/immunefi/harvest-finance-uninitialized-proxies-bug-fix-postmortem-ea5c0f7af96b) ($200,000 bounty)
- [Teller](https://medium.com/immunefi/teller-bug-fix-postmorten-and-bug-bounty-launch-b3f67a65c5ac) ($50,000 bounty)
- [Aave V2](https://blog.trailofbits.com/2020/12/16/breaking-aave-upgradeability/) and a related [Aave Medium post](https://medium.com/aave/aave-security-newsletter-546bf964689d) ($25,000 bounty)
- [Agave V2 (Aave V2 fork)](https://medium.com/@hacxyk/forked-protocols-are-not-battle-tested-agave-uninitialized-proxy-vulnerability-6b5d587b3a07) ($20,000 bounty)

### CTF Examples

None?

### Further reading

- [OpenZeppelin proxy vulnerability](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5vp3-v4hc-gx76)
- [iosiro blog post about OpenZeppelin vulnerability](https://www.iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure)

---

## Storage Collision Vulnerability

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/storage_collision)

A storage collision happens when the storage slot layout in the implementation contract does not match the storage slot layout in the proxy contract. This causes a problem because the `delegatecall` in the proxy contract means that the implementation contract is using the proxy contract's storage, but the variables in the implementation contract determine where that data is stored. If there is a mismatch between the proxy contract storage slots and the implementation contract storage slots, a storage collision can happen.

Take the Audius hack as an example. The AudiusAdminUpgradeabilityProxy contract storage slots collided with the initialization boolean values that indicated whether the proxy was initialized or not. The links to writeups about the details of the Audius hack are found below.

![AudiusAdminUpgradeabilityProxy storage slots after mitigation](../../assets/images/AudiusAdminUpgradeabilityProxy_before.jpg)

*The proxy contract storage slots visualized [with sol2uml](https://github.com/naddison36/sol2uml)*

![DelegateManager storage slots before mitigation](../../assets/images/DelegateManagerV2_before.jpg)

*The DelegateManager contract storage before mitigation. The storage of the boolean values collide with the proxyAdmin address in the proxy contract.*

![DelegateManager storage slots after mitigation](../../assets/images/DelegateManagerV2_after.jpg)

*The DelegateManager contract storage after mitigation. The storage of the boolean values has been moved to a new storage slot to avoid a storage collision.*

### Testing procedure

There are many approaches to testing for this vulnerability. One way to test for this vulnerability is using the [sol2uml](https://github.com/naddison36/sol2uml) tool. You can visualize the storage slots of the proxy contract and the implementation contract to see if they have any mismatches.

A second approach that is more programmatic is using [slither-read-storage](https://github.com/crytic/slither/blob/master/slither/tools/read_storage/README.md) to collect the storage slots used by the proxy contract and the implementation contract, then comparing them.

A third approach is to find a tool that is designed to compare the storage slots of two contracts. [This tool](https://github.com/ItsNickBarry/hardhat-storage-layout-diff) may work.

Be aware that these approaches would not have caught the vulnerability in the [Furucombo](https://medium.com/furucombo/furucombo-post-mortem-march-2021-ad19afd415e) hack. A solution specific to the Furucombo hack would be to check if the a delegatecall calls another contract with a delegatecall where the contracts used different storage slots to store their implementation contract addresses. One could argue this issue is a subcategory of the uninitialized proxy vulnerability.

OpenZeppelin [previously investigated an automated detection strategy](https://github.com/OpenZeppelin/openzeppelin-sdk/issues/37) for storage upgrades for zos.

Slither has a `slither-check-upgradeability` tool that has [several detectors for storage layout issues](https://github.com/crytic/slither/wiki/Upgradeability-Checks).

There is [a semgrep rule](https://github.com/Decurity/semgrep-smart-contracts/blob/master/solidity/security/proxy-storage-collision.yaml) designed to detect the Audius hack problem pattern, but the semgrep rule does not appear to be a robust method of identifying proxy collisions.

### Hacks

- [Furucombo](https://medium.com/furucombo/furucombo-post-mortem-march-2021-ad19afd415e) (Related writeups [here](https://rekt.news/furucombo-rekt/) and [here](https://github.com/OriginProtocol/security/blob/master/incidents/2021-02-27-Furucombo.md))
- [Audius](https://blog.audius.co/article/audius-governance-takeover-post-mortem-7-23-22) (Related writeup [here](https://rekt.news/audius-rekt/))

### Bug Bounties

None?

### CTF Examples

- [Solidity by Example](https://solidity-by-example.org/hacks/delegatecall/)
- [Ethernaut Level 6 "Delegation"](https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/contracts/levels/Delegation.sol)
- [Ethernaut Level 16 "Preservation"](https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/contracts/levels/Preservation.sol)
- [Ethernaut Level 24 "Puzzle Wallet"](https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/contracts/levels/PuzzleWallet.sol)
- [Underhanded Solidity 2020 entry 4](https://github.com/ethereum/solidity-underhanded-contest/tree/master/2020/submissions_2020/submission4_JaimeIglesias) from Jaime Iglesias

### Further reading

- [OpenZeppelin explanation](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#storage-collisions-between-implementation-versions)
- [semgrep rule to detect a specific case of proxy storage collision](https://github.com/Decurity/semgrep-smart-contracts/blob/master/solidity/security/proxy-storage-collision.yaml)
- [MixBytes storage collision audit finding](https://mixbytes.io/blog/collisions-solidity-storage-layouts)

---

## Function Clashing Vulnerability

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/function_clashing)

Function clashing is a result of compiled smart contracts using a 4 byte identifier (derived from the function name's hash) to identify functions, known as a [function selector](https://docs.soliditylang.org/en/latest/abi-spec.html#function-selector). Functions with different names can contain identical 4 bytes identifiers when the first 32 bits of their hashes are the same. The compiler will detect when the same 4 byte function selector exists twice in a single contract, but it does not prevent the same 4 byte function selector from existing in different contracts of a project.

Function clashing can be found in most but not all proxy types. Specifically UUPS proxies are normally not vulnerable to function clashing because the implementation contract stores all the custom functions.

### Testing procedure

To test for this vuln, you can collect the function selectors of a proxy contract and implementation contract to compare them for any function clashing. One tool for this is solc, where `solc --hashes MyContract.sol` will list all function selectors. Slither has a [Slither's Function ID printer](https://github.com/crytic/slither/wiki/Printer-documentation#function-id) that can do the same thing. Slither also has a `slither-check-upgradeability` tool that can [detect function clashing](https://github.com/crytic/slither/wiki/Upgradeability-Checks#functions-ids-collisions).

### Hacks

None?

### Bug Bounties

None?

### CTF Examples

None?

### Further reading

- [Tincho Function Clashing writeup](https://forum.openzeppelin.com/t/beware-of-the-proxy-learn-how-to-exploit-function-clashing/1070)
- [Nomic Labs blog post](https://medium.com/nomic-foundation-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357)
- [OpenZeppelin Docs explaining function clashing](https://docs.openzeppelin.com/sdk/2.5/pattern#transparent-proxies-and-function-clashes)

---

## Metamorphic Contract Rug Vulnerability

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/metamorphic_rug)

The CREATE2 opcode was introduced in the Constantinople hardfork with [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014). It allows a contract to be deployed at an address that can be calculated in advance, unlike the CREATE opcode. It is possible to deploy a contract, destroy the contract with `selfdestruct`, and then deploy a new contract with different code at the same address as the original contract. If a user is unaware that the code at this address changed since they originally interacted with the contract, they might end up interacting with a malicious contract. The planned removal of the `selfdestruct` opcode with [EIP-4758](https://eips.ethereum.org/EIPS/eip-4758) will remove the ability to create metamorphic contracts in the future.

### Testing procedure

To test for this vulnerability, you can use one of the existing tools mentioned in the "further reading" section, or to manually search for this issue:
1. Find the creation transaction on etherscan (manually or with [the etherscan API](https://docs.etherscan.io/api-endpoints/contracts#get-contract-creator-and-creation-tx-hash))
2. Check if a CREATE2 call was used in the transaction that created this target contract. If the target contract was created with CREATE2, continue testing. If not, the target contract is not at risk of being replaced with new code at the same address.
3. Check if the target contract, created by a CREATE2 call, contains a selfdestruct or a delegatecall. If the delegatecall allows calling another contract's selfdestruct, it is the same result as finding a selfdestruct in the target contract.
4. If CREATE2 was not used to create this contract but a selfdestruct or delegatecall exists, check if the parent of the target was created with a CREATE2 call. Continue checking the ancestry of the contract up the family tree until you reach an EOA address, because a CREATE2 anywhere in the target contract's ancestry can pose a risk.

Even if the target contract you are examining cannot be replace with this vulnerability, it may perform an external call (call, staticcall, or delegatecall) to another contract which is vulnerable to the metamorphic contract rug vulnerability. Consider testing all external addresses that are called by the target contract.

### Hacks

- [Tornado Cash governance hack](https://github.com/pcaversaccio/tornado-cash-exploit)

### Bug Bounties

None?

### CTF Examples

- [Underhanded Solidity 2020 submission #8](https://github.com/ethereum/solidity-underhanded-contest/tree/master/2020/submissions_2020/submission8_RichardMoore) from Richard Moore

### Further reading

- [Rajeev forum post about CREATE2 security implications](https://ethereum-magicians.org/t/potential-security-implications-of-create2-eip-1014/2614)
- [CertiK metamorphic contract detector tool](https://medium.com/certik/introducing-certiks-create2-audit-tool-2c75f0b53f54)
- [a16z metamorphic contract detector tool](https://a16zcrypto.com/metamorphic-smart-contract-detector-tool/)
- [PoC metamorphic contract detector tool](https://gist.github.com/engn33r/ec2d8f176bff962064afdadedb2d6faf)

---

## Delegatecall with Selfdestruct Vulnerability

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct)

There are unexpected edge cases when `delegatecall` and `selfdestruct` are used together. Specifically, if contract A has a `delegatecall` to contract B, and the function in contract B contains `selfdestruct`, it is contract A that will be destroyed.

### Testing procedure

This vulnerability is easy to identify. First, if a contract has a `delegatecall` that delegates to a user-provided address (such as a function argument in an external function), this is a substantial security risk overall

If a contract has a `delegatecall` to a hardcoded target contract, check if the target contract contains a `selfdestruct`. If the target contract does not contain `selfdestruct` but contains a `delegatecall`, then check the contract that is delegated to for a `selfdestruct` (and continue the process if another `delegatecall` is found). If there is a `selfdestruct` in the target contract, the original contract that contains the `delegatecall` could be destroyed. If the master contract used for EIP-1167 cloning is selfdestructed, all clones created from this contract [will stop working](https://github.com/optionality/clone-factory#warnings).

### Hacks

- [Parity Wallet](https://www.parity.io/blog/a-postmortem-on-the-parity-multi-sig-library-self-destruct/) and [OpenZeppelin writeup](https://blog.openzeppelin.com/on-the-parity-wallet-multisig-hack-405a8c12e8f7/) (note: the contract was initialized but had no protection against a second call to `initialize()`)

### Bug Bounties

None?

### CTF Examples

- [Paradigm CTF 2021 "Vault"](https://github.com/paradigmxyz/paradigm-ctf-2021/tree/master/vault)
- [Ethernaut Level 25 "Motorbike"](https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/contracts/levels/Motorbike.sol)

### Further reading

- [OpenZeppelin proxy vulnerability](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5vp3-v4hc-gx76)
- [iosiro blog post about OpenZeppelin vulnerability](https://www.iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure)

---

## Delegatecall to Arbitrary Address

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground)

A `delegatecall` passes the execution from the proxy contract to another contract, but the state variables and context (msg.sender, msg.value) from the proxy contract are used. If the implementation contract that `delegatecall` passes execution to can be an arbitrary contract, substantial problems emerge. For one, a denial-of-service is possible by combining `delegatecall` with `selfdestruct` (see [the relevant section](#delegatecall-with-selfdestruct-vulnerability)). Another risk is that if users have used `approve` or set an allowance to trust the proxy contract containing the `delegatecall` to an arbitrary address, the arbitrary `delegatecall` target can be used to steal user funds. The address that a contract transfer execution to with `delegatecall` must be a trusted contract and must not be open-ended to allow a user to provide the address to delegate to.

### Testing procedure

For automated testing, the ["Controlled Delegatecall" slither detector](https://github.com/crytic/slither/wiki/Detector-Documentation#controlled-delegatecall) can detect this issue. For manual testing, examine the address used for any delegatecall operation. If this value can be set by an untrusted user input at any point, there is a risk of code execution being passed to an arbitrary address.

### Hacks

None?

### Bug Bounties

- [dYdX deposit proxy post-mortem](https://dydx.exchange/blog/deposit-proxy-post-mortem) ($500,000 bounty)
- [Astaria beacon proxy thread](https://twitter.com/apoorvlathey/status/1671308180545241088)

### CTF Examples

None?

### Further reading

- [SWC-112](https://swcregistry.io/docs/SWC-112)

---

## Delegatecall external contract missing existence check

When `delegatecall` is used, there is no automated check for whether the external contract exists. If the external contract called does not exist, the return value will be `true`. This is documented in a [warning note in the solidity documentation](https://docs.soliditylang.org/en/latest/control-structures.html#error-handling-assert-require-revert-and-exceptions) with the following:

> The low-level functions `call`, `delegatecall` and `staticcall` return `true` as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed.

### Testing procedure

The first step is to identify the external contract address that the call is using. If it is possible for there to be no contract at this address, and there is no check to verify that the contract exists before the `delegatecall`, then `delegatecall` may return true unexpectedly.

### Hacks

None?

### Bug Bounties

- [ZeppelinOS bug](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/)

### CTF Examples

None?

### Further reading

- [Manticore detection of nonexistent contract](https://github.com/trailofbits/manticore/pull/1119)
