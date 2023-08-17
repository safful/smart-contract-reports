## Tags

- bug
- 2 (Med Risk)
- selected for report

# [Governor can rug pull the escrow](https://github.com/code-423n4/2022-10-thegraph-findings/issues/300) 

# Lines of code

https://github.com/code-423n4/2022-10-thegraph/blob/309a188f7215fa42c745b136357702400f91b4ff/contracts/gateway/BridgeEscrow.sol#L28-L30


# Vulnerability details




## Impact
Governor can rug pull all GRT held by BridgeEscrow, which is a severe undermining of decentralization.

## Proof of Concept
The governor can approve an arbitrary address to spend any amount from BridgeEscrow, so they can steal all escrowed tokens. Even if the governor is well intended, the contract can still be called out which would degrade the reputation of the protocol (e.g. see here: https://twitter.com/RugDocIO/status/1411732108029181960). This is especially an issue as the escrowed tokens are never burnt, so the users would need to trust the governor perpetually (not about stealing their L2 tokens, but about not taking a massive amount of L1 tokens for free for themselves).

This seems an unnecessary power granted to the governor and turns a decentralized bridge into a needless bottleneck of centralization.

## Tools Used
Code inspection

## Recommended Mitigation Steps
Restrict access to `approveAll()` to the ["bridge that manages the GRT funds held by the escrow"](https://github.com/code-423n4/2022-10-thegraph/blob/309a188f7215fa42c745b136357702400f91b4ff/contracts/gateway/BridgeEscrow.sol#L25). Or, similarly to how `finalizeInboundTransfer` in the gateways is restricted to its respective counterpart, only allow spending via other protocol functions.