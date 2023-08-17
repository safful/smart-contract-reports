## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-02

# [POSITION LIMIT COULD BE FULLY REDUCED TO ZERO BY CLONES](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/932) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/MintingHub.sol#L126
https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Position.sol#L97-L101


# Vulnerability details

## Impact
A newly opened position could have its limit fully reduced to zero as soon as the cooldown period has elapsed. 

## Proof of Concept
As seen in the function below, a newly opened position with `0` Frankencoin minted could have its `limit` turn `0` if the function parameter, `_minimum`, is inputted with an amount equal to `limit`. In this case, `reduction` is equal to `0`, making `limit - _minimum = 0` while the cloner is assigned `reduction + _minimum = 0 + limit = limit`: 

[Position.sol#L97-L101](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Position.sol#L97-L101)

```
    function reduceLimitForClone(uint256 _minimum) external noChallenge noCooldown alive onlyHub returns (uint256) {
        uint256 reduction = (limit - minted - _minimum)/2; // this will fail with an underflow if minimum is too high
        limit -= reduction + _minimum;
        return reduction + _minimum;
    }
```
With the limit now fully allocated to the cloner, the original position owner is left with zero limit to mint Frankencoin after spending 1000 Frankencoin to open this position. This situation could readily happen especially when it involves popular position contracts.

## Recommended Mitigation Steps
It is recommended position contract charging fees to cloners. Additionally, a reserve limit should be left untouched allocated solely to the original owner to be in line with the context of position opening.