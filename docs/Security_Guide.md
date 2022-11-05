---
layout: default
title: Security Guide to Proxy Vulns
nav_order: 3
has_children: true
---

If you are unsure which proxy type is in the scope of your audit or security review, see the [proxy identification flow chart](/docs/Proxy-Identification).

# Uninitialized Proxy

Impacts UUPS proxies because...

## Bug Bounties

- Wormhole: https://medium.com/immunefi/wormhole-uninitialized-proxy-bugfix-review-90250c41a43a
- Harvest: https://medium.com/immunefi/harvest-finance-uninitialized-proxies-bug-fix-postmortem-ea5c0f7af96b
- Teller: https://medium.com/immunefi/teller-bug-fix-postmorten-and-bug-bounty-launch-b3f67a65c5ac
- Aave V2: https://blog.trailofbits.com/2020/12/16/breaking-aave-upgradeability/ and https://medium.com/aave/aave-security-newsletter-546bf964689d

## Testing procedure

To test for this vuln, you can approach it from angles XYZ

## CTF Examples

Coming soon...

# Storage Collisions

Impacts X because...

## Hacks

- https://solidity-by-example.org/hacks/delegatecall/
- https://blog.audius.co/article/audius-governance-takeover-post-mortem-7-23-22

## Testing procedure

To test for this vuln, you can approach it from angles XYZ

## CTF Examples

Coming soon...

# Function Collisions/Clashing

- https://github.com/tinchoabbate/function-clashing-poc
- https://medium.com/nomic-foundation-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357
- https://docs.openzeppelin.com/sdk/2.5/pattern#transparent-proxies-and-function-clashes

## Testing procedure

To test for this vuln, you can approach it from angles XYZ...

## CTF Examples

Coming soon...

# CREATE2 contract replacement

Cannot be verified on etherscan?! Need to check on this.

## Testing procedure

To test for this vuln, you can approach it from angles XYZ...

## CTF Examples

Coming soon...

# OZ UUPS vuln

- https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-5vp3-v4hc-gx76
- https://www.iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure

## Testing procedure

To test for this vuln, you can approach it from angles XYZ...

## CTF Examples

Coming soon...