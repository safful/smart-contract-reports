## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [Wrong design/implementation of freeTrial allows attacker to steal funds from the protocol](https://github.com/code-423n4/2021-11-unlock-findings/issues/188) 

# Handle

WatchPug


# Vulnerability details

The current design/implementation of freeTrial allows users to get full refund before the freeTrial ends. Plus, a user can transfer partial of thier time to another user using `shareKey`.

This makes it possible for the attacker to steal from the protocol by transferring freeTrial time from multiple addresses to one address and adding up to `expirationDuration` and call refund to steal from the protocol.

### PoC

Given:

- `keyPrice` is 1 ETH;
- `expirationDuration` is 360 days;
- `freeTrialLength` is 31 days.

The attacker can create two wallet addresses: Alice and Bob.

1. Alice calls `purchase()`, transfer 30 days via `shareKey()` to Bob, then calls `cancelAndRefund()` to get full refund; Repeat 12 times;
2. Bob calls `cancelAndRefund()` and get 1 ETH.

### Recommendation

Consider disabling `cancelAndRefund()` for users who transferred time to another user.

