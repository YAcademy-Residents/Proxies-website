---
layout: default
title: Proxies Deep Dive
nav_order: 3
has_children: true
---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The Proxy

> And Vitalik said, "Let there be Proxies!"

The proxy itself is not inherently upgradeable, but it is the basis for just about all upgradeable proxy patterns.  Calls made to the **proxy** contract are forwarded to the **implementation** contract using `delegatecall`.  The **implementation** contract is also referred to as the **logic** contract.


In some variants, calls to the proxy are only forwarded if the caller matches an "owner" address.

**Implementation address** - Immutable in the proxy contract.

**Upgrade logic** - There is no upgradeability in a pure proxy contract.

**Contract verification** - Works with Etherscan ([example](https://etherscan.io/address/0x09cabec1ead1c0ba254b09efb3ee13841712be14#readProxyContract)) and other block explorers.


### Use cases
* Useful when there is a need to deploy multiple contracts whose code is more or less the same.

### Pros
* Inexpensive deployment.

### Cons
* Adds a single `delegatecall` cost to each call.

### Examples
* [Uniswap V1 AMM pools](https://etherscan.io/address/0x09cabec1ead1c0ba254b09efb3ee13841712be14#code)
* [Synthetix](https://github.com/Synthetixio/synthetix/pull/1191)

### Known vulnerabilities
* [Delegatecall and selfdestruct not allowed in implementation](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct/UUPS_selfdestruct)

### Variations
* [The EIP-1167 standard](https://eips.ethereum.org/EIPS/eip-1167) was created in June '18 with the goal of standardizing a way to clone contract functionality simply, cheaply, and in an immutable way.  This standard contains a minimal bytecode redirect implementation that has been optimized for the proxy contract. This is often used with a [factory pattern](https://github.com/optionality/clone-factory).

### Further reading
* [A Minimal Proxy in the Wild](https://blog.originprotocol.com/a-minimal-proxy-in-the-wild-ae3f7b8da990)
* [OpenZeppelin core Proxy contract](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Proxy)
* [Deep dive into the Minimal Proxy contract](https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/)

---------

## The Initializeable Proxy

> "But what are we supposed to do without a `constructor()`?"

Most modern day proxies are initializeable. One of the main benefits of using a proxy is that you only have to deploy the implementation contract (AKA the logic contract) once, and then you can deploy many proxy contracts that point at it. However, the downside to this is that you cannot use a constructor in the already deployed implementation contract when creating the new proxy.

Instead, an `initialize()` function is used to set initial storage values:

```solidity
    uint8 private _initialized;

    function initializer() external {
        require(msg.sender == owner);
        require(_initialized < 1);
        _initialized = 1;

        // set some state vars
        // do initialization stuff
    }
```

### Use cases
* Most proxies with any kind of storage that needs to be set upon proxy contract deployment.

### Pros
* Allows initial storage to be set at time of new proxy deployment.

### Cons
* Susceptible to attacks related to initialization, especially uninitialized proxies.

### Examples
* This feature is used with most modern proxy types including TPP and UUPS, except for use cases where there is no need to set storage upon proxy deployment.

### Known vulnerabilities
* [Uninitialized proxy](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized/UUPS_Uninitialized)

### Variations
* [Clone factory contract model](https://github.com/optionality/clone-factory) - uses clone initialization in a creation transaction.
* [Clones with Immutable Args](https://github.com/wighawag/clones-with-immutable-args) - enables creating clone contracts with immutable arguments which are stored in the code region of the proxy contract. When called, arguments are appended to the calldata of the `delegatecall`,  Implementation contract function then reads the arguments from calldata.  This pattern can remove the need to use an initializer but the downside is that currently [the contract cannot be verified on Etherscan](https://twitter.com/boredGenius/status/1484713577961250821?s=20&t=5jbuvNruLIJlLRow1nKrMw).


### Further reading
 - [Initializeable - OZ](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable)

---------

## The Upgradeable Proxy

> And the people said, "But we want to upgrade our immutable contracts!"

The Upgradeable Proxy is similar to a [Proxy](#the-proxy), except the implementation contract address is settable and kept in storage in the proxy contract. The proxy contract also contains permissioned upgrade functions. One of the [first upgradeable proxy contracts](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f) was written by [Nick Johnson](https://twitter.com/nicksdjohnson) in 2016.

For security, it is also recommended to use a form of access control to differentiate between the owner/caller and the admin with permission to upgrade the contract.

**Implementation address** - Located in proxy storage.

**Upgrade logic** - Located in the proxy contract.

**Contract verification** - Depending on the exact implementation, it may not work with block explorers like Etherscan.

### Use cases
* A minimalistic upgrade contract. Useful for learning projects.

### Pros
* Reduced deployment costs through use of the [Proxy](#the-proxy).
* Implementation contract is upgradeable.

### Cons
* Prone to storage and function clashing.
* Less secure than modern counterparts.
* Every call incurs cost of `delegatecall` from the [Proxy](#the-proxy).

### Examples
* This basic style is not widely used anymore.

### Known vulnerabilities
* [Delegatecall and selfdestruct not allowed in implementation](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct/UUPS_selfdestruct)
* [Uninitialized proxy](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized/UUPS_Uninitialized)
* [Storage collision](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/storage_collision)
* [Function clashing](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/function_clashing)

### Further reading
* [The First Proxy Contract](https://ethereum-blockchain-developer.com/110-upgrade-smart-contracts/05-proxy-nick-johnson/)
* [Writing Upgradeable Contracts](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)

-------

## EIP-1967 Upgradeable Proxy

> The "solution" to storage collisions

This is similar to the [Upgradeable Proxy](#the-upgradeable-proxy), except that it reduces risk of storage collision by using the [unstructured storage pattern](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#unstructured-storage-proxies). It does **not** store the implementation contract address in slot 0 or any other standard storage slot.

Instead the address is stored in a pre-agreed upon slot. For example [OpenZeppelin contracts](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.3/contracts/proxy/ERC1967/ERC1967Upgrade.sol) use the keccak-256 hash of the string "eip1967.proxy.implementation" **minus one***. Because this slot is widely used, block explorers can identify and handle when proxies are being used.

*The minus provides additional safety because without it, the slot has a known preimage, but after subtracting 1, the preimage is unknown. For a known preimage, the storage slot may be overwritten via a mapping for example, where storage slot for its key is determined using a keccak-256 hash.


[EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) also specifies a slot for admin storage (auth) as well as Beacon Proxies which will be discussed in detail below.

**Implementation address** - Located in a unique storage slot in the proxy contract.

**Upgrade logic** - Varies based on implementation.

**Contract verification** - Yes, most evm block explorers support it.

### Use cases
* When you need more security than the basic [Upgradeable Proxy](#the-upgradeable-proxy).

### Pros
* Reduces risk of storage collisions.
* Block explorer compatibility

### Cons
* Susceptible to function clashing.
* Less secure than modern counterparts.
* Every call incurs cost of `delegatecall` from the [Proxy](#the-proxy).


### Examples
* While the [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) storage slot pattern has been widely adopted in most modern upgradeable proxy types, this bare bones contract is not seen in the wild as much as some of the newer patterns like [TPP](#transparent-proxy-tpp), [UUPS](#universal-upgradeable-proxy-standard-uups), and [Beacon](#beacon-proxy).


### Known vulnerabilities
* [Delegatecall and selfdestruct not allowed in implementation](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct/UUPS_selfdestruct)
* [Uninitialized proxy](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized/UUPS_Uninitialized)
* [Function clashing](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/function_clashing/UUPS_functionClashing)

### Further reading
* [EIP-1967 Standard Proxy Storage Slots](https://ethereum-blockchain-developer.com/110-upgrade-smart-contracts/09-eip-1967/)
* [The Proxy Delegate](https://fravoll.github.io/solidity-patterns/proxy_delegate.html)

-------

## Transparent Proxy (TPP)

> The "solution" to function clashing

This is similar to the [Upgradeable Proxy](#the-upgradeable-proxy) and usually incorporates [EIP-1967](#eip-1967-upgradeable-proxy). But, if the caller is the admin of the proxy, the proxy will not delegate any calls, and if the caller is any other address, the proxy will always delegate the call, even if the func sig matches one of the proxy’s own functions. This is often implemented with a modifier like this one from [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/transparent/TransparentUpgradeableProxy.sol#L45-L51):

```solidity
modifier ifAdmin() {
    if (msg.sender == _getAdmin()) {
        _;
    } else {
        _fallback(); // redirects call to proxy
    }
}
```

and a check in the `fallback()`:
```solidity
require(msg.sender != _getAdmin(), "TransparentUpgradeableProxy: admin cannot fallback to proxy target");
```

**Implementation address** - Located in a unique storage slot in the proxy contract ([EIP-1967](#eip-1967-upgradeable-proxy)).

**Upgrade logic** - Located in the proxy contract with use of a [modifier](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/transparent/TransparentUpgradeableProxy.sol#L45-L51) to re-route non-admin callers.

**Contract verification** - Yes, most evm block explorers support it.

### Use cases
* This pattern is very widely used for its upgradeability and protections against certain function and storage collision vulnerabilities.

### Pros
* Eliminates possibility of function clashing for admins, since they are never redirected to the implementation contract.
* Since the upgrade logic lives on the proxy, if a proxy is left in an uninitialized state or if the implementation contract is selfdestructed, then the implementation can still be set to a new address.
* Reduces risk of storage collisions from use of [EIP-1967](#eip-1967-upgradeable-proxy) storage slots.
* Block explorer compatibility.

### Cons
* Every call not only incurs runtime gas cost of `delegatecall` from the [Proxy](#the-proxy) but also incurs cost of SLOAD for checking whether the caller is admin.
* Because the upgrade logic lives on the proxy, there is more bytecode so the deploy costs are higher.

### Examples
* [dYdX](https://github.com/dydxprotocol/perpetual/blob/99962cc62caed2376596da357a13f5c3d0ea5e59/contracts/protocol/PerpetualProxy.sol)
* [USDC](https://github.com/centrehq/centre-tokens/tree/b42cf04b31639b8b05d53fea9995954d5f3659d9/contracts/upgradeability)
* [Aztec](https://github.com/AztecProtocol/AZTEC/blob/cb78ba3ee32ad82234ac0fbed046333eb7f233cf/packages/protocol/contracts/AccountRegistry/AccountRegistryManager.sol#L62-L66)
* [Hundreds of projects on Github](https://github.com/search?q=adminupgradeabilityproxy&type=Code)

### Known vulnerabilities
* [Delegatecall and selfdestruct not allowed in implementation](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct/UUPS_selfdestruct)
* [Uninitialized proxy](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized/UUPS_Uninitialized)
* [Storage collision](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/storage_collision)

### Further reading
* [The Transparent Proxy Pattern](https://blog.openzeppelin.com/the-transparent-proxy-pattern/)

-------

## Universal Upgradeable Proxy Standard (UUPS)

> What if we move the upgrade logic to the impl contract? 🤔

[EIP-1822](https://eips.ethereum.org/EIPS/eip-1822) describes a standard for an upgradeable proxy pattern where the `upgrade` logic is stored in the implementation contract.  This way, there is no need to check if the caller is admin in the proxy at the proxy level, saving gas.  It also eliminates the possibility of a function on the implementation contract colliding with the upgrade logic in the proxy.

The downside of UUPS is that it is considered riskier than TPP.  If the proxy does not get initialized properly or if the implementation contract were to selfdestruct, then there is no way to save the proxy since the upgrade logic lives on the implementation contract.

The UUPS proxy also contains an additional check when upgrading that ensures the new implementation contract is upgradeable.

This proxy contract usually incorporates [EIP-1967](#eip-1967-upgradeable-proxy).

**Implementation address** - Located in a unique storage slot in the proxy contract ([EIP-1967](#eip-1967-upgradeable-proxy)).

**Upgrade logic** - Located in the implementation contract.

**Contract verification** - Yes, most evm block explorers support it.

### Use cases
* These days this is the most widely used pattern when protocols look to deploy upgradeable contracts.

### Pros
* Eliminates risk of functions on the implementation contract colliding with the proxy contract since the upgrade logic lives on the implementation contract and there is no logic on the proxy besides the `fallback()` which delegatecalls to the impl contract.
* Reduced runtime gas over TPP because the proxy does not need to check if the caller is admin.
* Reduced cost of deploying a new proxy because the proxy only contains no logic besides the `fallback()`.
* Reduces risk of storage collisions from use of [EIP-1967](#eip-1967-upgradeable-proxy) storage slots.
* Block explorer compatibility.

### Cons
* Because the upgrade logic lives on the implementation contract, extra care must be taken to ensure the implementation contract cannot `selfdestruct` or get left in a bad state due to an improper initialization.  If the impl contract gets borked then the proxy cannot be saved.
* Still incurs cost of `delegatecall` from the [Proxy](#the-proxy).

### Examples
* [Superfluid](https://github.com/superfluid-finance/protocol-monorepo)
* [Synthetix](https://github.com/Synthetixio/synthetix-v3)
* [Hundreds of projects on Github](https://github.com/search?q=UUPSUpgradeable&type=code)

### Known vulnerabilities
* [Uninitialized proxy](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized/UUPS_Uninitialized)
* [Function clashing](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/function_clashing)
* [Selfdestruct](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct/UUPS_selfdestruct)


### Further reading
* [EIP-1822](https://eips.ethereum.org/EIPS/eip-1822)
* [Using UUPS Proxy Pattern](https://blog.logrocket.com/using-uups-proxy-pattern-upgrade-smart-contracts/)
* [Perma-brick UUPS proxies with this one trick](https://iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure)

-------

## Beacon Proxy

> Is that a beacon in your proxy or are you just happy to see me? 🤔

Most proxies discussed so far store the implementation contract address in the proxy contract storage. The Beacon pattern, popularized by [Dharma](https://github.com/dharma-eng/dharma-smart-wallet/blob/master/contracts/proxies/smart-wallet/UpgradeBeaconProxyV1.sol) in 2019, stores the address of the implementation contract in a separate "beacon" contract.  The address of the beacon is stored in the proxy contract using [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) storage pattern.

With other types of proxies, when the implementation contract is upgraded, all of the proxies need to be updated.  However, with the Beacon proxy, only the beacon contract itself needs to be updated.

Both the beacon address on the proxy as well as the implementation contract address on the beacon are settable by admin.  This allows for many powerful combinations when dealing with large quantities of proxy contracts that need to be grouped in different ways.


**Implementation address** - Located in a unique storage slot in the beacon contract.  The beacon address lives in a unique storage slot in the proxy contract.

**Upgrade logic** - Upgrade logic typically lives in the beacon contract.

**Contract Verification** - Yes, most evm block explorers support it.

### Use cases
* If you have a need for multiple proxy contracts that can all be upgraded at once by upgrading the beacon.
* Appropriate for situations that involve large amounts of proxy contracts based on multiple implementation contracts.  The beacon proxy pattern enables updating various groups of proxies at the same time.

### Pros
* Easier to upgrade multiple proxy contracts at the same time.

### Cons
* Gas overhead of getting the beacon contract address from storage, calling beacon contract, and then getting the implementation contract address from storage, plus the extra gas required by using a proxy.
* Adds additional complexity.

### Examples
* [USDC](https://polygonscan.com/address/0xb254554636a3ff52e8b2d0f06203921c137e10d5#code)
* [Dharma](https://github.com/dharma-eng/dharma-smart-wallet/blob/master/contracts/proxies/smart-wallet/UpgradeBeaconProxyV1.sol)

### Known vulnerabilities
* [Delegatecall and selfdestruct not allowed in implementation](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct/UUPS_selfdestruct)
* [Uninitialized proxy](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/uninitialized/UUPS_Uninitialized)
* [Function clashing](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/function_clashing)

### Variations
* Immutable Beacon address - To save gas, the beacon address can be made immutable in the proxy contract.  The implementation contract would still be settable by updating it on the beacon.
* [Storageless Upgradeable Beacon Proxy](https://forum.openzeppelin.com/t/a-more-gas-efficient-upgradeable-proxy-by-not-using-storage/4111) - In this pattern, the beacon contract does not store the implementation contract address in storage, but instead stores it in code.  The proxy contract loads it from the beacon directly via `EXTCODECOPY`.

### Further reading
* [How to create a Beacon Proxy](https://medium.com/coinmonks/how-to-create-a-beacon-proxy-3d55335f7353)
* [Dharma](https://github.com/dharma-eng/dharma-smart-wallet/blob/master/contracts/proxies/smart-wallet/UpgradeBeaconProxyV1.sol)

-------

## Diamond Proxy

> "Diamonds are a proxy's best friend?"

[EIP-2535](https://eips.ethereum.org/EIPS/eip-2535) "Diamonds" are modular smart contract systems that can be upgraded/extended after deployment, and have virtually no size limit. From the EIP:

> a diamond is a contract with external functions that are supplied by contracts called facets. Facets are separate, independent contracts that can share internal functions, libraries, and state variables.

The diamond pattern consists of a central Diamond.sol proxy contract. In addition to other storage, this contract contains a registry of functions that can be called on external contracts called facets.

Glossary of Diamond proxy uses a unique vocabulary:

| Diamond term | Definition |
| -------- | -------- |
|Diamond| Proxy|
|Facet| Implementation|
|Cut| Upgrade|
|Loupe| List of delegated functions|
|Finished diamond| Non-upgradeable|
|Single cut diamond| Remove upgradeability functions|


**Contract Verification** - Contracts can be verified on Etherscan with the help of a tool called **Louper** ([example](https://louper.dev/diamond/0x17525e4f4af59fbc29551bc4ece6ab60ed49ce31)).

### Use cases
* A complex system where the highest level of upgradeability and modular interoperability is required.

### Pros
* A stable contract address that provides needed functionality. Emitting events from a single address can simplify event handling.
* Can be used to break up a large contract > 24kb that is over the Spurious Dragon limit.

### Cons
* Additional gas required to access storage when routing functions.
* Increased chance of storage collision due to complexity.
* Complexity may be too much when simple upgradeability is required.

### Examples
* [Simple DeFi](https://www.simpledefi.io/)
* [PartyFinance](https://party.finance/)
* [Complete list of examples](https://github.com/mudgen/awesome-diamonds#projects-using-diamonds).

### Known vulnerabilities
* [Delegatecall and selfdestruct not allowed in implementation](https://github.com/YAcademy-Residents/Solidity-Proxy-Playground/tree/main/src/delegatecall_with_selfdestruct/UUPS_selfdestruct)

### Variations
* [vtable](https://github.com/OpenZeppelin/openzeppelin-labs/tree/master/upgradeability_with_vtable)
* [How to build unlimited size contracts](https://twitter.com/ylv_io/status/1581639395064836102?s=20&t=WoHhqaSl8SlEdLUje5Cziw)

### Further reading
* [Answering some Diamond questions](https://eip2535diamonds.substack.com/p/answering-some-diamond-questions)
* [Dark Forest and the Diamond standard ](https://blog.zkga.me/dark-forest-and-the-diamond-standard)
* [Good idea, bad design. How the Diamond standard falls short](https://blog.trailofbits.com/2020/10/30/good-idea-bad-design-how-the-diamond-standard-falls-short/)
* [Addressing Josselin Feist's Concern's of EIP-2535 Diamonds](https://dev.to/mudgen/addressing-josselin-feist-s-concern-s-of-eip-2535-diamond-standard-me8)

-------

## Metamorphic Contracts

> "create2, use, selfdestruct, rinse, repeat..."

The metamorphic contract is unique from all the other upgradeable patterns in that it does NOT use a proxy. There is no `delegatecall` to an external logic contract.

When it's time for an upgrade, the metamorphic contract uses `selfdestruct` and the new contract is deployed to the same address using `create2`.

This can be achieved by having the initcode retrieve the creation code from the storage of a separate external contract.  In this way, the initcode will always be the same, and therefore `create2` can be used to deploy to the same address.

Metamorphic contracts are not recommended for new contracts because the `selfdestruct` opcode is scheduled to be removed from Ethereum in the near future. See [EIP-4758](https://eips.ethereum.org/EIPS/eip-4758) for details.

**Contract Verification** - Yes, metamorphic contracts can be verified.

### Use cases
* Contracts that contain only logic (similar to Solidity external libraries).
* Contracts with little state that changes infrequently, such as beacons.

### Pros
* Does not require the use of a proxy with `delegatecall`.
* Does not require using an `initialize()` instead of a `constructor()`.

### Cons
* Storage is erased on upgrade because of `selfdestruct`.
* Because `selfdestruct` clears the code at the end of the transaction, an upgrade requires two transactions: one to delete the current contract, and another to create the new one. Any transaction that arrives to our contract in between those two would fail.
* The `selfdestruct` opcode may be removed in the future.

### Examples
* [Example contracts from 0age](https://github.com/0age/metamorphic#metamorphic).
* This is more of an experimental type. Mostly used by MEV searchers (etherscan examples [here](https://etherscan.io/address/0x0000000000007f150bd6f54c40a34d7c3d5e9f56#code) and [here](https://etherscan.io/address/0x000000005736775feb0c8568e7dee77222a26880#code)).

### Known vulnerabilities
* Not vulnerable to the typical upgradeable proxy vulnerabilities since it doesn't use a proxy or an initializer.
* May be vulnerable to attack at time of upgrade.

### Further reading
* [Metamorphosis Smart Contracts using CREATE2](https://ethereum-blockchain-developer.com/110-upgrade-smart-contracts/12-metamorphosis-create2/)
* [The Promise and the Peril of Metamorphic Contracts](https://0age.medium.com/the-promise-and-the-peril-of-metamorphic-contracts-9eb8b8413c5e)
* [a16z Metamorphic Contract Detector Tool](https://a16zcrypto.com/metamorphic-smart-contract-detector-tool/)
