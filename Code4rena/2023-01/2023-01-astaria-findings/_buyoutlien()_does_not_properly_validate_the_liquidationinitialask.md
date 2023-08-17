## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-02

# [_buyoutLien() does not properly validate the liquidationInitialAsk](https://github.com/code-423n4/2023-01-astaria-findings/issues/587) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L148-L163


# Vulnerability details

## Impact
Illegal liquidationInitialAsk, resulting in insufficient bids to cover the debt
## Proof of Concept

_buyoutLien() will validate against liquidationInitialAsk, but incorrectly uses the old stack for validation

```solidity
  function _buyoutLien(
    LienStorage storage s,
    ILienToken.LienActionBuyout calldata params
  ) internal returns (Stack[] memory newStack, Stack memory newLien) {

....

    uint256 potentialDebt = 0;
    for (uint256 i = params.encumber.stack.length; i > 0; ) {
      uint256 j = i - 1;
      // should not be able to purchase lien if any lien in the stack is expired (and will be liquidated)
      if (block.timestamp >= params.encumber.stack[j].point.end) {
        revert InvalidState(InvalidStates.EXPIRED_LIEN);
      }

      potentialDebt += _getOwed(
        params.encumber.stack[j],
        params.encumber.stack[j].point.end
      );

      if (
        potentialDebt >
        params.encumber.stack[j].lien.details.liquidationInitialAsk   //1.****@audit use old stack
      ) {
        revert InvalidState(InvalidStates.INITIAL_ASK_EXCEEDED);
      }

      unchecked {
        --i;
      }
    }  
....
    newStack = _replaceStackAtPositionWithNewLien(
      s,
      params.encumber.stack,
      params.position,
      newLien,
      params.encumber.stack[params.position].point.lienId    //2.****@audit replace newStack
    );    
```

## Tools Used

## Recommended Mitigation Steps

Replace then verify, using the newStack[] for verification