---
layout: default
title: Security Guide to Proxy Vulns
nav_order: 3
has_children: true
---

If you are unsure which proxy type is in the scope of your audit or security review, see the [proxy identification flow chart](/docs/Proxy-Identification).

# Uninitialized Proxy

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized)

Why do proxies need an `initialize` function when a contract constructor is called automatically? The reason is explained [here by OpenZeppelin](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat). The code in a contract's constructor is run once at deployment, but there is no way to run constructor code of the implementation contract (AKA logic contract) *in the context* of the proxy contract. Because the implementation contract must store the value of the `_initialized` variable in the proxy contract context, the constructor cannot be used for this purpose, because the implementation contract's constructor code will always run in the context of the implementation contract. This is why there exists an `initialize` function in the implementation contract - because the `initialize` call must happen through the proxy.

A specific variant of the uninitialized UUPS proxy vulnerability is found in [the OpenZeppelin library between version 4.1.0 and 4.3.2](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/security/advisories/GHSA-q4h9-46xg-m3x9). This issue is related to an [edge case of delegatecall and selfdestruct interaction](/docs/Delegatecall-with-Selfdestruct).

## Bug Bounties

- Wormhole: https://medium.com/immunefi/wormhole-uninitialized-proxy-bugfix-review-90250c41a43a
- Arbitrum Nitro: https://medium.com/@0xriptide/hackers-in-arbitrums-inbox-ca23272641a2
- Harvest: https://medium.com/immunefi/harvest-finance-uninitialized-proxies-bug-fix-postmortem-ea5c0f7af96b
- Teller: https://medium.com/immunefi/teller-bug-fix-postmorten-and-bug-bounty-launch-b3f67a65c5ac
- Aave V2: https://blog.trailofbits.com/2020/12/16/breaking-aave-upgradeability/ and https://medium.com/aave/aave-security-newsletter-546bf964689d
- Agave V2 (Aave V2 fork): https://medium.com/@hacxyk/forked-protocols-are-not-battle-tested-agave-uninitialized-proxy-vulnerability-6b5d587b3a07

## Testing procedure

To test for this vulnerability, first identify the storage slot of the [initialized state variable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/25aabd286e002a1526c345c8db259d57bdf0ad28/contracts/proxy/utils/Initializable.sol#L62) or a similar variable that the initialization function uses to revert if this is not the first time that the function is called. Try using [these tools](/docs/Proxies-Storage.md) to find the correct storage slot. If the OpenZeppelin private `_initialized` variable from Initializable.sol is used, a `_initialized` value of zero means the contract has not been initialized while a `_initialized` value of 1 means the contract has been initialized.

A simplistic check that has a low false-positive rate with a high false-negative is to check if the implementation contract has a non-empty constructor. Because any storage values set in the constructor of the implementation contract will not be used when the implementation contract is called through the proxy contract, a constructor can indicate a case of initialization values not getting set as expected.

## CTF Examples

Coming soon...

### Further reading

- [OpenZeppelin proxy vulnerability](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5vp3-v4hc-gx76)
- [iosiro blog post about OpenZeppelin vulnerability](https://www.iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure)

# Storage Collisions

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized)

A storage collision happens when the storage slot layout in the implementation contract does not match the storage slot layout in the proxy contract. This causes a problem because the `delegatecall` in the proxy contract means that the implementation contract is using the proxy contract's storage, but the variables in the implementation contract determine where that data is stored. If there is a mismatch between the proxy contract storage slots and the implementation contract storage slots, a storage collision can happen.

Take the Audius hack as an example. TODO: Add screenshots of storage slot layout that caused the issue. Also note the address(es) of the Audius contracts for others to investigate too.

## Hacks
- [Furucombo](https://medium.com/furucombo/furucombo-post-mortem-march-2021-ad19afd415e) (Related writeups [here](https://rekt.news/furucombo-rekt/) and [here](https://github.com/OriginProtocol/security/blob/master/incidents/2021-02-27-Furucombo.md))
- [Audius](https://blog.audius.co/article/audius-governance-takeover-post-mortem-7-23-22) (Related writeup [here](https://rekt.news/audius-rekt/))

## Testing procedure

There are many approaches to testing for this vulnerability. One way to test for this vulnerability is using the [sol2uml](https://github.com/naddison36/sol2uml) tool. You can visualize the storage slots of the proxy contract and the implementation contract to see if they have any mismatches.

A second approach that is more programmatic can rely on [slither-read-storage](https://github.com/crytic/slither/blob/master/slither/tools/read_storage/README.md) to compare the proxy contract and implementation contract storage slots in a more automated fashion.

Neither of these approaches would have caught the vulnerability in the [Furucombo](https://medium.com/furucombo/furucombo-post-mortem-march-2021-ad19afd415e) hack, though a solution specific to the Furucombo hack would be to check if the proxy contract uses the standard storage slot of 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc (this value is [from EIP-1967](https://eips.ethereum.org/EIPS/eip-1967#abstract)) to store the implementation contract's address.

## CTF Examples

- [Solidity by Example](https://solidity-by-example.org/hacks/delegatecall/)
- [Underhanded Solidity 2020 entry 4](https://github.com/ethereum/solidity-underhanded-contest/tree/master/2020/submissions_2020/submission4_JaimeIglesias) from Jaime Iglesias

### Further reading

- [OpenZeppelin explanation](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#storage-collisions-between-implementation-versions)
- [semgrep rule to detect a specific case of proxy storage collision](https://github.com/Decurity/semgrep-smart-contracts/blob/master/solidity/proxy-storage-collision.yaml)

# Function Collisions/Clashing

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized)

Function collisions can be found in most but not all proxy types. Specifically UUPS proxies are normally not vulnerable to function collisions because the implementation contract stores all the custom functions.

## Testing procedure

To test for this vuln, you can use [Slither's Function ID printer](https://github.com/crytic/slither/wiki/Printer-documentation#function-id) to compare function signatures of the proxy contract and the implementation contract.

TODO: Approach if the source code is not available?

## CTF Examples

Coming soon...

### Further reading

- https://github.com/tinchoabbate/function-clashing-poc
- https://medium.com/nomic-foundation-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357
- https://docs.openzeppelin.com/sdk/2.5/pattern#transparent-proxies-and-function-clashes

# CREATE2 contract replacement

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized)

TODO: can contract vulnerable to this vector be verified on etherscan?! Need to check on this.

## Testing procedure

To test for this vuln:
1. Find the creation transaction on eitherscan (manually or with [the etherscan API](https://docs.etherscan.io/api-endpoints/contracts#get-contract-creator-and-creation-tx-hash))
2. Check if a CREATE2 call was used in the transaction that create this contract. If the contract WAS created with CREATE2, continue testing. If not, the contract is not at risk of being replaced at the same address.
3. Check if the target contract, created by a CREATE2 call, contains a selfdestruct or a delegatecall. If the delegatecall allows calling another contract's selfdestruct, it is the same result as finding a selfdestruct in the target contract.
4. If CREATE2 was not used to create this contract but a selfdestruct or delegatecall exists, check if the parent of the target was created with a CREATE2 call. Continue checking the ancestry of the contract up the family tree, because a CREATE2 anywhere in the target contract's ancestry can pose a risk.

## CTF Examples

Coming soon...

### Further reading

- [Underhanded Solidity submission #8](https://github.com/ethereum/solidity-underhanded-contest/tree/master/2020/submissions_2020/submission8_RichardMoore) from Richard Moore

# delegatecall with selfdestruct

[Playground Link](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct)

There are unexpected edge cases when `delegatecall` and `selfdestruct` are used together. Specifically, if contract A has a `delegatecall` to contract B, and the function in contract B contains `selfdestruct`, it is contract A that will be destroyed.

## Testing procedure

This vulnerability is easy to identify. If a contract has a `delegatecall` to a hardcoded target contract, check if the target contract contains a `selfdestruct` (or contains a `delegatecall`, in which case you should repeat this step on the target contract). If there is a `selfdestruct` in the target contract, the 

## CTF Examples



### Further reading

- [OpenZeppelin proxy vulnerability](https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5vp3-v4hc-gx76)
- [iosiro blog post about OpenZeppelin vulnerability](https://www.iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure)
