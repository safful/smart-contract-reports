## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-09

# [User can avoid bankrupting by calling PositionManager.moveLiquidity where to index is bankrupted index](https://github.com/code-423n4/2023-05-ajna-findings/issues/179) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L262-L333


# Vulnerability details

## Impact
User can avoid bankrupting by calling PositionManager.moveLiquidity where to index is bankrupted index

## Proof of Concept
Bucket could become insolvent and in that case all LP within the bucket are zeroed out (lenders lose all their LP). Because of that, `PositionManager.reedemPositions` [will not allow to redeem index that is bankrupted](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L372).

When user wants to move his LPs from one bucket to another he can call `PositionManager.moveLiquidity` where he will provide from and to indexes.
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L262-L333
```solidity
    function moveLiquidity(
        MoveLiquidityParams calldata params_
    ) external override mayInteract(params_.pool, params_.tokenId) nonReentrant {
        Position storage fromPosition = positions[params_.tokenId][params_.fromIndex];

        MoveLiquidityLocalVars memory vars;
        vars.depositTime = fromPosition.depositTime;

        // handle the case where owner attempts to move liquidity after they've already done so
        if (vars.depositTime == 0) revert RemovePositionFailed();

        // ensure bucketDeposit accounts for accrued interest
        IPool(params_.pool).updateInterest();

        // retrieve info of bucket from which liquidity is moved  
        (
            vars.bucketLP,
            vars.bucketCollateral,
            vars.bankruptcyTime,
            vars.bucketDeposit,
        ) = IPool(params_.pool).bucketInfo(params_.fromIndex);

        // check that bucket hasn't gone bankrupt since memorialization
        if (vars.depositTime <= vars.bankruptcyTime) revert BucketBankrupt();

        // calculate the max amount of quote tokens that can be moved, given the tracked LP
        vars.maxQuote = _lpToQuoteToken(
            vars.bucketLP,
            vars.bucketCollateral,
            vars.bucketDeposit,
            fromPosition.lps,
            vars.bucketDeposit,
            _priceAt(params_.fromIndex)
        );

        EnumerableSet.UintSet storage positionIndex = positionIndexes[params_.tokenId];

        // remove bucket index from which liquidity is moved from tracked positions
        if (!positionIndex.remove(params_.fromIndex)) revert RemovePositionFailed();

        // update bucket set at which a position has liquidity
        // slither-disable-next-line unused-return
        positionIndex.add(params_.toIndex);

        // move quote tokens in pool
        (
            vars.lpbAmountFrom,
            vars.lpbAmountTo,
        ) = IPool(params_.pool).moveQuoteToken(
            vars.maxQuote,
            params_.fromIndex,
            params_.toIndex,
            params_.expiry
        );

        Position storage toPosition = positions[params_.tokenId][params_.toIndex];

        // update position LP state
        fromPosition.lps -= vars.lpbAmountFrom;
        toPosition.lps   += vars.lpbAmountTo;
        // update position deposit time to the from bucket deposit time
        toPosition.depositTime = vars.depositTime;

        emit MoveLiquidity(
            ownerOf(params_.tokenId),
            params_.tokenId,
            params_.fromIndex,
            params_.toIndex,
            vars.lpbAmountFrom,
            vars.lpbAmountTo
        );
    }
```
As you can see `from` bucket is checked to be not bankrupted before the moving.
And after the move, LPs of `from` and `to` buckets are updated.
Also `depositTime` of `to` bucket is updated to `from.depositTime`.

The problem here is that `to` bucket was never checked to be not bankrupted.
Because of that it's possible that bankrupted `to` bucket now becomes not bankrupted as their depositTime is updated now.

This is how this can be used by attacker.
1.Attacker has lp shares in the bucket, linked to token and this bucket became bankrupt.
2.Then attacker mints small amount of LP in the Pool and then memorizes this index to the token.
3.Attacker calls `moveLiquidity` with `from`: new bucket and `to`: bankrupt bucket.
4.Now attacker can redeem his lp shares from bankrupt bucket as depositedTime is updated now.

As result, attacker was able to steal LPs of another people from `PositionManager` contract.
## Tools Used
VsCode
## Recommended Mitigation Steps
In case if `to` bucket is bankrupt, then clear LP for it before adding moved lp shares.


## Assessed type

Error