## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-05

# [Owner of Denied Position is not able to withdraw collateral until expiry.](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/874) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Position.sol#L112
https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Position.sol#L263
https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Position.sol#L373-L376


# Vulnerability details

Denying a position puts it into perma cooldown state ie. cooldown ends at expiry. It’s impossible to withdraw collateral in the cooldown state. 

## Impact
Locks owner funds until expiry, expiry time is not capped and can be expected to be long. There is no benefit to the owner to set it shorter and be forced to repay the position at an inconvenient time. Hence a high risk exists to lock the collateral semi permanently

## Proof of Concept
Consider owner trying to call `withdrawCollateral` on a denied position
```
    function deny(address[] calldata helpers, string calldata message) public {
        if (block.timestamp >= start) revert TooLate();
        IReserve(zchf.reserve()).checkQualified(msg.sender, helpers);
        cooldown = expiration; // since expiration is immutable, we put it under cooldown until the end
        emit PositionDenied(msg.sender, message);
    }

    function withdrawCollateral(address target, uint256 amount) public onlyOwner noChallenge noCooldown {
        uint256 balance = internalWithdrawCollateral(target, amount);
        checkCollateral(balance, price);
    }

    modifier noCooldown() {
        if (block.timestamp <= cooldown) revert Hot();
        _;
    }


```
1. Successful call all to `deny` will set `cooldown = expiry`
2. Subsequent call to `withdrawCollateral` will be reverted by `noCooldown` modifier

## Tools Used
VS code, Pen and Paper

## Recommended Mitigation Steps
Return the collateral to the owner at the end of `deny`
```
    function deny(address[] calldata helpers, string calldata message) public {
        if (block.timestamp >= start) revert TooLate();
        IReserve(zchf.reserve()).checkQualified(msg.sender, helpers);
        cooldown = expiration; // since expiration is immutable, we put it under cooldown until the end
        internalWithdrawCollateral(owner, IERC20(collateral).balanceOf(address(this)));
        emit PositionDenied(msg.sender, message);
    }
```