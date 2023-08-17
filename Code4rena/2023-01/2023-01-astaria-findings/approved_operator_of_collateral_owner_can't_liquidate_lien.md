## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-27

# [Approved operator of collateral owner can't liquidate lien](https://github.com/code-423n4/2023-01-astaria-findings/issues/133) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/AstariaRouter.sol#L611-L619
https://github.com/code-423n4/2023-01-astaria/blob/main/src/VaultImplementation.sol#L237-L244


# Vulnerability details

## Impact
Approved operator of collateral owner can't liquidate lien

## Proof of Concept
If someone wants to liquidate lien then `canLiquidate` function [is called](https://github.com/code-423n4/2023-01-astaria/blob/main/src/AstariaRouter.sol#L625-L627) to check if it's possible.
https://github.com/code-423n4/2023-01-astaria/blob/main/src/AstariaRouter.sol#L611-L619
```soldiity
  function canLiquidate(ILienToken.Stack memory stack)
    public
    view
    returns (bool)
  {
    RouterStorage storage s = _loadRouterSlot();
    return (stack.point.end <= block.timestamp ||
      msg.sender == s.COLLATERAL_TOKEN.ownerOf(stack.lien.collateralId));
  }
```

As you can see owner of collateral token can liquidate lien in any moment.
However approved operators of owner can't do that, however they should.

As while validating commitment it's allowed for approved operator [to request a loan](https://github.com/code-423n4/2023-01-astaria/blob/main/src/VaultImplementation.sol#L237-L244).
That means that owner of collateral token can approve some operators to allow them to work with their debts.
So they should be able to liquidate loan as well.
## Tools Used
VsCode
## Recommended Mitigation Steps
Add ability for approved operators to liqiudate lien.