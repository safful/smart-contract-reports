## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-17

# [Address.isContract() is not a reliable way of checking if the input is an EOA](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/189) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L435


# Vulnerability details

## Impact
The underlying assumption of `eoaRepresentative` being an EOA can be untrue. This can cause many unintended effects as the contract comments strongly suggests that this must be an EOA account.

## Proof of Concept
When BLS public key is registered in `registerBLSPublicKeys()`, it has the check of 
> `require(!Address.isContract(_eoaRepresentative), "Only EOA representative permitted")`

However, this check can be passed even though input is a smart contract if
1. Function is called in the constructor. `Address.isContract()` checks for the code length, but during construction code length is 0.
2. Smart contract that has not been deployed yet can be used. The CREATE2 opcode can be used to deterministically calculate the address of a smart contract before it is created. This means that the user can bypass this check by calling this function before deploying the contract.

## Tools Used
Manual Review

## Recommended Mitigation Steps
It is generally not recommended to enforce an address to be only EOA and AFAIK, this is impossible to enforce due to the aforementioned cases. I recommend the protocol team to take a closer look at this and build the protocol with the assumption that `_eoaRepresentative == EOA`.