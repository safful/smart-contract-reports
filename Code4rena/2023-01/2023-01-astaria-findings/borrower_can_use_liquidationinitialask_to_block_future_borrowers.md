## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- H-12

# [Borrower can use liquidationInitialAsk to block future borrowers](https://github.com/code-423n4/2023-01-astaria-findings/issues/289) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L471-L489
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L153-L174


# Vulnerability details

## Impact
When a new lien is taken (or bought out), one of the validations is to ensure that the `potentialDebt` of each borrower on the stack is less than or equal to their `liquidationInitialAsk`.

```
if (potentialDebt > newStack[j].lien.details.liquidationInitialAsk) {
    revert InvalidState(InvalidStates.INITIAL_ASK_EXCEEDED);
  }
```

In `_appendStack()` and `_buyoutLien()`, this is performed by iterating through the stack backwards, totaling up the `potentialDebt`, and comparing it to each lien's `liquidationInitialAsk`:

```
for (uint256 i = stack.length; i > 0; ) {
      uint256 j = i - 1;
      newStack[j] = stack[j];
      if (block.timestamp >= newStack[j].point.end) {
        revert InvalidState(InvalidStates.EXPIRED_LIEN);
      }
      unchecked {
        potentialDebt += _getOwed(newStack[j], newStack[j].point.end);
      }
      if (potentialDebt > newStack[j].lien.details.liquidationInitialAsk) {
        revert InvalidState(InvalidStates.INITIAL_ASK_EXCEEDED);
      }

      unchecked {
        --i;
      }
    }
```
However, only the first item on the stack has a `liquidationInitialAsk` that matters. When a new auction is started on Seaport, `Router#liquidate()` uses `stack[0].lien.details.liquidationInitialAsk` as the starting price. The other values are meaningless, except in their ability to DOS future borrowers.

## Proof of Concept

- I set my `liquidationInitialAsk` to be exactly the value of my loan
- A borrower has already borrowed on their collateral, and the first loan on the stack will determine the auction price
- When they borrow from me, my `liquidationInitialAsk` is recorded
- Any future borrows will check that `futureBorrow + myBorrow <= myLiquidationInitialAsk`, which is not possible for any `futureBorrow > 0`
- The result is that the borrower will be DOS'd from all future borrows

This is made worse by the fact that `liquidationInitialAsk` is not a variable that can justify a refinance, so they'll need to either pay back the loan or find a refinancier who will beat one of the other terms (rate or duration) in order to get rid of this burden.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Get rid of all checks on `liquidationInitialAsk` except for comparing the total potential debt of the entire stack to the `liquidationInitialAsk` of the lien at position 0.