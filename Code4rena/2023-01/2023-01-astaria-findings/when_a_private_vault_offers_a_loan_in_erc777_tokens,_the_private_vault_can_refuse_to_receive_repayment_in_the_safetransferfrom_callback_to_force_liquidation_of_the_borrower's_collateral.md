## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-21

# [When a private vault offers a loan in ERC777 tokens, the private vault can refuse to receive repayment in the safeTransferFrom callback to force liquidation of the borrower's collateral](https://github.com/code-423n4/2023-01-astaria-findings/issues/247) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L849-L850


# Vulnerability details

## Impact
When the borrower calls LienToken.makePayment to repay the loan, safeTransferFrom is used to send tokens to the recipient of the vault, which in the case of a private vault is the owner of the private vault.
```solidity
    s.TRANSFER_PROXY.tokenTransferFrom(stack.lien.token, payer, payee, amount);
...
  function tokenTransferFrom(
    address token,
    address from,
    address to,
    uint256 amount
  ) external requiresAuth {
    ERC20(token).safeTransferFrom(from, to, amount);
  }
...
  function recipient() public view returns (address) {
    if (IMPL_TYPE() == uint8(IAstariaRouter.ImplementationType.PublicVault)) {
      return address(this);
    } else {
      return owner();
    }
  }
```
If the token for the loan is an ERC777 token, a malicious private vault owner can refuse to receive repayment in the callback, which results in the borrower not being able to repay the loan and the borrower's collateral being auctioned off when the loan expires.
## Proof of Concept
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L849-L850
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/TransferProxy.sol#L28-L35
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L900-L909
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L366-L372
## Tools Used
None
## Recommended Mitigation Steps
For private vaults, when the borrower repays, sending tokens to the vault, and the private vault owner claims it later