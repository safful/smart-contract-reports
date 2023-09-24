## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-33

# [`BaseV2Minter` DAO reward shares are calculated wrong](https://github.com/code-423n4/2023-05-maia-findings/issues/104) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/hermes/minters/BaseV2Minter.sol#L134-L136


# Vulnerability details

## Impact

In `BaseV2Minter` when calculating the DAO shares out of the weekly emissions, the current implementation wrongly also takes into consideration the extra `bHERMES` growth tokens (to the locked) thus is allocated a larger value then intended. This also has an indirect effect of increasing protocol inflation if `HERMES` [needs to be minted in order to reach the required token amount](https://github.com/code-423n4/2023-05-maia/blob/main/src/hermes/minters/BaseV2Minter.sol#L138-L141).

### Issue details

Token DAO shares (`share` variable) is calculated in `BaseV2Minter::updatePeriod` as such:

https://github.com/code-423n4/2023-05-maia/blob/main/src/hermes/minters/BaseV2Minter.sol#L133-L137
```Solidity
    uint256 _growth = calculateGrowth(newWeeklyEmission);
    uint256 _required = _growth + newWeeklyEmission;
    /// @dev share of newWeeklyEmission emissions sent to DAO.
    uint256 share = (_required * daoShare) / base;
    _required += share;
```

We actually do see that the original developer intention (confirmed by sponsor) was that the share value to be calculated relative to `newWeeklyEmission`, not to (`_required = newWeeklyEmission + _growth`)

```Solidity
    /// @dev share of newWeeklyEmission emissions sent to DAO.
```

Also, it is [documented that DAO shares should be calculated as part of weekly emissions](https://v2-docs.maiadao.io/protocols/Hermes/overview/tokenomics/emissions#dao-emissions):

> Up to 30% of weekly emissions can be allocated to the DAO.

## Proof of Concept

DAO shares value is not calculated relative to `newWeeklyEmission`

https://github.com/code-423n4/2023-05-maia/blob/main/src/hermes/minters/BaseV2Minter.sol#L134-L136

## Tools Used

Manual analysis

## Recommended Mitigation Steps

Change the implementation to reflect intention

```diff
diff --git a/src/hermes/minters/BaseV2Minter.sol b/src/hermes/minters/BaseV2Minter.sol
index 7d7f013..217a028 100644
--- a/src/hermes/minters/BaseV2Minter.sol
+++ b/src/hermes/minters/BaseV2Minter.sol
@@ -133,7 +133,7 @@ contract BaseV2Minter is Ownable, IBaseV2Minter {
             uint256 _growth = calculateGrowth(newWeeklyEmission);
             uint256 _required = _growth + newWeeklyEmission;
             /// @dev share of newWeeklyEmission emissions sent to DAO.
-            uint256 share = (_required * daoShare) / base;
+            uint256 share = (newWeeklyEmission * daoShare) / base;
             _required += share;
             uint256 _balanceOf = underlying.balanceOf(address(this));
             if (_balanceOf < _required) {

```


## Assessed type

Other