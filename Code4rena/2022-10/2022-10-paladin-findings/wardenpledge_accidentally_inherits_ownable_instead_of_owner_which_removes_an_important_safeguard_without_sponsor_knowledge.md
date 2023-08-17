## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-05

# [WardenPledge accidentally inherits Ownable instead of Owner which removes an important safeguard without sponsor knowledge](https://github.com/code-423n4/2022-10-paladin-findings/issues/161) 

# Lines of code

https://github.com/code-423n4/2022-10-paladin/blob/d6d0c0e57ad80f15e9691086c9c7270d4ccfe0e6/contracts/WardenPledge.sol#L18


# Vulnerability details

## Impact

Owner may accidentally transfer ownership to inoperable address due to perceived safeguard that doesn't exist

## Proof of Concept

    contract WardenPledge is Ownable, Pausable, ReentrancyGuard {

WardenPledge inherits from Ownable rather than Owner, which is the intended contract. Owner overwrites the critical Ownable#transferOwnership function to make the ownership transfer process a two step process. This adds important safeguards because in the event that the target is unable to accept for any reason (input typo, incompatible multisig/contract, etc.) the ownership transfer process will fail because the pending owner will not be able to accept the transfer. To make matters worse, since it only overwrites the trasnferOwnership function the WardenPledge contract will otherwise function as intended just without this safeguard. It is likely that the owner won't even realize until too late and the safeguard has failed. A perceived safeguard where there isn't one is more damaging than not having any safeguard at all.

## Tools Used

Manual Review

## Recommended Mitigation Steps

    -   contract WardenPledge is Ownable, Pausable, ReentrancyGuard {
    +   contract WardenPledge is Owner, Pausable, ReentrancyGuard {