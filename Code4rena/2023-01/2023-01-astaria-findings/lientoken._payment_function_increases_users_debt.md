## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-31

# [LienToken._payment function increases users debt](https://github.com/code-423n4/2023-01-astaria-findings/issues/96) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L790-L853


# Vulnerability details

## Impact
LienToken._payment function increases users debt. Every time when user pays not whole lien amount then interests are added to the loan amount, so next time user pays interests based on interests.

## Proof of Concept
`LienToken._payment` is used by `LienToken.makePayment` function that allows borrower to repay part or all his debt.
https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L790-L853
```solidity
  function _payment(
    LienStorage storage s,
    Stack[] memory activeStack,
    uint8 position,
    uint256 amount,
    address payer
  ) internal returns (Stack[] memory, uint256) {
    Stack memory stack = activeStack[position];
    uint256 lienId = stack.point.lienId;


    if (s.lienMeta[lienId].atLiquidation) {
      revert InvalidState(InvalidStates.COLLATERAL_AUCTION);
    }
    uint64 end = stack.point.end;
    // Blocking off payments for a lien that has exceeded the lien.end to prevent repayment unless the msg.sender() is the AuctionHouse
    if (block.timestamp >= end) {
      revert InvalidLoanState();
    }
    uint256 owed = _getOwed(stack, block.timestamp);
    address lienOwner = ownerOf(lienId);
    bool isPublicVault = _isPublicVault(s, lienOwner);


    address payee = _getPayee(s, lienId);


    if (amount > owed) amount = owed;
    if (isPublicVault) {
      IPublicVault(lienOwner).beforePayment(
        IPublicVault.BeforePaymentParams({
          interestOwed: owed - stack.point.amount,
          amount: stack.point.amount,
          lienSlope: calculateSlope(stack)
        })
      );
    }


    //bring the point up to block.timestamp, compute the owed
    stack.point.amount = owed.safeCastTo88();
    stack.point.last = block.timestamp.safeCastTo40();


    if (stack.point.amount > amount) {
      stack.point.amount -= amount.safeCastTo88();
      //      // slope does not need to be updated if paying off the rest, since we neutralize slope in beforePayment()
      if (isPublicVault) {
        IPublicVault(lienOwner).afterPayment(calculateSlope(stack));
      }
    } else {
      amount = stack.point.amount;
      if (isPublicVault) {
        // since the openLiens count is only positive when there are liens that haven't been paid off
        // that should be liquidated, this lien should not be counted anymore
        IPublicVault(lienOwner).decreaseEpochLienCount(
          IPublicVault(lienOwner).getLienEpoch(end)
        );
      }
      delete s.lienMeta[lienId]; //full delete of point data for the lien
      _burn(lienId);
      activeStack = _removeStackPosition(activeStack, position);
    }


    s.TRANSFER_PROXY.tokenTransferFrom(stack.lien.token, payer, payee, amount);


    emit Payment(lienId, amount);
    return (activeStack, amount);
  }
```

The main problem is in line 826. `stack.point.amount = owed.safeCastTo88();`
https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L826

Here `stack.point.amount` becomes `stack.point.amount` + accrued interests, because `owed` is loan amount + accrued interests by this time.

`stack.point.amount` is the amount that user borrowed. So actually that line has just increased user's debt. And in case if [he didn't pay all amount](https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L830) of lien, then next time he will pay more interests, because interests [depend on loan amount](https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L262).

In the docs Astaria protocol claims that: 
> All rates on the Astaria protocol are in simple interest and non-compounding.

https://docs.astaria.xyz/docs/protocol-mechanics/loanterms
You can check that inside "Interest rate" section.

However `_payment` function is compounding interests.
To avoid this, another field `interestsAccrued` should be introduced which will track already accrued interests.
## Tools Used
VsCode
## Recommended Mitigation Steps
You need store accrued interests by the moment of repayment to another variable `interestsAccrued`.
And calculate `_getInterest` functon like this.
```solidity
function _getInterest(Stack memory stack, uint256 timestamp)
    internal
    pure
    returns (uint256)
  {
    uint256 delta_t = timestamp - stack.point.last;

    return stack.point.interestsAccrued + (delta_t * stack.lien.details.rate).mulWadDown(stack.point.amount);
  }
```