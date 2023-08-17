## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-04

# [Approved operators of Position token can't call Trading.initiateCloseOrder](https://github.com/code-423n4/2022-12-tigris-findings/issues/124) 

# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L235
https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L847-L849


# Vulnerability details

## Impact
Approved operators of owner of Position token can't call several function in Trading.

## Proof of Concept
Functions that accept Position token in Trading are checking that the caller is owner of token using _checkOwner function.
https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L847-L849
```soldiity
    function _checkOwner(uint _id, address _trader) internal view {
        if (position.ownerOf(_id) != _trader) revert("2"); //NotPositionOwner   
    }
```
As you can see this function doesn't allow to approved operators of token's owner to pass the check. As result functions are not possible to call for them on behalf of owner.
For example [here](https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L235) there is a check that doesn't allow to call initiateCloseOrder function.
## Tools Used
VsCode
## Recommended Mitigation Steps
Allow operators of token's owner to call functions on behalf of owner.