---
layout: default
title: Proxies Storage
nav_order: 1
parent: Full List of Proxy Types
---

### Storage Slot Analysis for Verified Contracts
- [slither-read-storage](https://github.com/crytic/slither/blob/master/slither/tools/read_storage/README.md): A slither tool to retrieve storage slots for verified contracts
- [sol2uml](https://github.com/naddison36/sol2uml): A visualizer of storage slot usage for a verified contract
- [Slither unitialized proxy detector](https://github.com/crytic/slither/wiki/Detector-Documentation#race-conditions-at-contracts-initialization): Only can detect race condition vulnerability because slither's static analyzer is designed for contract code that is not deployed on chain yet

### Storage Slot Analysis for Unverified Contracts
- [Contract Library](https://library.dedaub.com/): When viewing a contract, can navigate to the "Read/Write" tab and choose "Storage dump" to see all values stored in all storage slots. Doesn't require a verified contract.
- [smart-contract-storage-viewer](https://github.com/tintinweb/smart-contract-storage-viewer): Useful tool from tintinweb to view what a contract has stored in storage slots
- [Etherscan](https://etherscan.io/): The "Switch to Opcodes View" button next to the contract creation code and review all the SLOAD operations to identify storage slots storing values