## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [Stader OPERATOR is single point of failure](https://github.com/code-423n4/2023-06-stader-findings/issues/344) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/PermissionlessNodeRegistry.sol#L183
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/PermissionedNodeRegistry.sol#L254


# Vulnerability details

## Impact
The OPERATOR role holds a lot of power within the system, which can compromise the both the system integrity and it's permission-less nature. 

## Proof of Concept
The OPERATOR key is responsible for confirming marking each validator submitted key as either valid or invalid, without any assurance to validators. 

1. Arbitrary negation of participation makes permissionless pool permissioned.
The documentation states: 

> Any validator in permissionless pool can run a node with 4 ETH + 0.4 ETH worth of SD token.

Which is not strictly true, since any participant in the system must be vetted by the OPERATOR, which can arbitrarily mark as invalid or frontrun key without the need to provide justification or having an appeal system. Alternatively, the OPERATOR can simple ignore the added key and never mark it as `ready to deposit`. 

Therefore, the pool can't be considered permissionless, since participants must rely on the benevolence of the OPERATOR to participate. 

2. Authorization of invalid keys
There is no way for the smart contract system to check or confirm that a given public key is really legit, and could generate income to ETHx holders, so the system relies solely on the OPERATOR to make that distinction, rendering the system vulnerable in case of a comprised wallet.


## Tools Used
Manual Review

## Recommended Mitigation Steps
There is no simple fix for the issue, but at minimum, the protocol shouldn't be advertised as permissioneless.


## Assessed type

Rug-Pull