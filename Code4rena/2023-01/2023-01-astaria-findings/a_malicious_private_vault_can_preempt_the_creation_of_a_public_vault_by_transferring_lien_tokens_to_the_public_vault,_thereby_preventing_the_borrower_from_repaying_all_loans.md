## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- H-14

# [A malicious private vault can preempt the creation of a public vault by transferring lien tokens to the public vault, thereby preventing the borrower from repaying all loans](https://github.com/code-423n4/2023-01-astaria-findings/issues/246) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L360-L375


# Vulnerability details

## Impact
In LienToken.transferFrom, transferring lien tokens to the public vault is prohibited because variables such as liensOpenForEpoch are not updated when the public vault receives a lien token, which would prevent the borrower from repaying the loan in that lien token.
```solidity
  function transferFrom(
    address from,
    address to,
    uint256 id
  ) public override(ERC721, IERC721) {
    LienStorage storage s = _loadLienStorageSlot();
    if (_isPublicVault(s, to)) {
      revert InvalidState(InvalidStates.PUBLIC_VAULT_RECIPIENT);
    }
    if (s.lienMeta[id].atLiquidation) {
      revert InvalidState(InvalidStates.COLLATERAL_AUCTION);
    }
    delete s.lienMeta[id].payee;
    emit PayeeChanged(id, address(0));
    super.transferFrom(from, to, id);
  }
```
However, public vaults are created using the ClonesWithImmutableArgs.clone function, which uses the `create` opcode, which allows the address of the public vault to be predicted before it is created.

https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed
```solidity
            assembly {
                instance := create(0, ptr, creationSize)
            }
```
This allows a malicious private vault to transfer lien tokens to the predicted public vault address in advance, and then call AstariaRouter.newPublicVault to create the public vault, which has a liensOpenForEpoch of 0.
When the borrower repays the loan via LienToken.makePayment, decreaseEpochLienCount fails due to overflow in _payment, resulting in the liquidation of the borrower's collateral
```solidity
    } else {
      amount = stack.point.amount;
      if (isPublicVault) {
        // since the openLiens count is only positive when there are liens that haven't been paid off
        // that should be liquidated, this lien should not be counted anymore
        IPublicVault(lienOwner).decreaseEpochLienCount(
          IPublicVault(lienOwner).getLienEpoch(end)
        );
      }
```

Consider the following scenario where private vault A provides a loan of 1 ETH to the borrower, who deposits NFT worth 2 ETH and borrows 1 ETH.
Private Vault A creates Public Vault B using the account alice and predicts the address of Public Vault B before it is created and transfers the lien tokens to it.
The borrower calls LienToken.makePayment to repay the loan, but fails due to overflow.
The borrower is unable to repay the loan, and when the loan expires, the NFTs used as collateral are auctioned

## Proof of Concept
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L360-L375
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L835-L847
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L731-L742
https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed
## Tools Used
None
## Recommended Mitigation Steps
In LienToken.transferFrom, require to.code.length >0, thus preventing the transfer of lien tokens to uncreated public vaults

```diff
  function transferFrom(
    address from,
    address to,
    uint256 id
  ) public override(ERC721, IERC721) {
    LienStorage storage s = _loadLienStorageSlot();
    if (_isPublicVault(s, to)) {
      revert InvalidState(InvalidStates.PUBLIC_VAULT_RECIPIENT);
    }
+  require(to.code.length > 0);
    if (s.lienMeta[id].atLiquidation) {
      revert InvalidState(InvalidStates.COLLATERAL_AUCTION);
    }
    delete s.lienMeta[id].payee;
    emit PayeeChanged(id, address(0));
    super.transferFrom(from, to, id);
  }
```