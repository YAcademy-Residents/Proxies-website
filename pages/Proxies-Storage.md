---
layout: default
title: Proxies Storage
nav_order: 1
parent: Proxies Deep Dive
---

# Proxies Storage

Note: If you are new to understanding how smart contract storage slots work, review an entry level guide first, like the [solidity docs](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html#layout-of-state-variables-in-storage).

Storing state variables is important for any smart contract, but especially so for proxies. Checking the values stored at specific storage slots in a contract is a useful way to check if the proxy is working as expected. Once the storage slot for a variable is known, a tool like [seth](https://github.com/dapphub/dapptools/tree/master/src/seth#seth-storage), [cast](https://book.getfoundry.sh/reference/cast/cast-storage), or your favorite web3 JS/Python library ([ethersJS](https://docs.ethers.io/v5/api/providers/provider/#Provider-getStorageAt), etc.) can pull the data at that location directly from the blockchain. These tools can help identify the storage slot of the variables found in the deployed contract.

## Storage Slot Analysis for Verified Contracts
- [slither-read-storage](https://github.com/crytic/slither/blob/master/slither/tools/read_storage/README.md): A slither tool to retrieve storage slots for verified contracts. A [blog post](https://blog.trailofbits.com/2022/07/28/shedding-smart-contract-storage-with-slither/) was published with more examples.
- [sol2uml](https://github.com/naddison36/sol2uml): A visualizer of storage slot usage for a verified contract
- [forge inspect](https://book.getfoundry.sh/reference/forge/forge-inspect?highlight=layout): Lists the storage slots for variables in a contract. If the code is verified on-chain, it can be downloaded to your local system using [ethereum-sources-downloader](https://www.npmjs.com/package/ethereum-sources-downloader).
- [hardhat-storage-layout](https://github.com/aurora-is-near/hardhat-storage-layout): A hardhat plugin to view storage slots of variables.
- [solc \-\-storage-layout](https://docs.soliditylang.org/en/latest/using-the-compiler.html#input-description): Similar to foundry, can output the storage slot and offset for each state variable.
- [MetaDock Chrome Extension](https://chrome.google.com/webstore/detail/metadock/fkhgpeojcbhimodmppkbbliepkpcgcoo) enhances etherscan and other blockchain scanners to show the values of private variables that normally cannot be accessed on etherscan.

## Storage Slot Analysis for Unverified Contracts (and Verified Contracts)
- [Contract Library](https://library.dedaub.com/): When viewing a contract, can navigate to the "Read/Write" tab and choose "Storage dump" to see all values stored in all storage slots. Doesn't require a verified contract. The decompiler is also very helpful.
- [smart-contract-storage-viewer](https://github.com/tintinweb/smart-contract-storage-viewer): Useful tool from tintinweb to view what a contract has stored in specific storage slots. This tool is only useful if you already know the storage slot value locations to view or want to view a range of many sequential storage slot values.
- [Etherscan](https://etherscan.io/): The "Switch to Opcodes View" button next to the contract creation code and review all the SLOAD operations to identify storage slots that hold (or held) values.