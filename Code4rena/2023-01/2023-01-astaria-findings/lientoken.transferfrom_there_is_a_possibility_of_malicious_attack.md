## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-04

# [LienToken.transferFrom There is a possibility of malicious attack](https://github.com/code-423n4/2023-01-astaria-findings/issues/571) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L366-L368


# Vulnerability details

## Impact
Corrupt multiple key properties of public vault, causing vault not to function properly

## Proof of Concept

When LienToken.makePayment()/buyoutLien()/payDebtViaClearingHouse()
If it corresponds to PublicVault, it will make multiple changes to the vault, such as: yIntercept, slope, last, epochData, etc.

If LienToken corresponds to PublicVault, then ownerOf(lienId) = PublicVault address

When the LienToken is a private vault, it is possible to transfer the owner of the LienToken.

As the above seems, if the private vault is transferred to the PublicVault address will result in the wrong modification of the yIntercept, slope, last, epochData, etc.

So we restrict the to in transferFrom to not be a PublicVault address

```solidity

  function transferFrom(
    address from,
    address to,
    uint256 id
  ) public override(ERC721, IERC721) {

    if (_isPublicVault(s, to)) {  //***@audit when to == PublicVault address , will revert
      revert InvalidState(InvalidStates.PUBLIC_VAULT_RECIPIENT);  
    }  
```

However, such a restriction does not prevent an attacker from transferring PrivateVault's LienToken to PublicVault
Because the address is predictable when the vault contract is created, a malicious user can predict the vault address, front-run, and transfer PrivateVault's LienToken to the predicted PublicVault address before the public vault is created, thus bypassing this restriction

Assumptions:
1. alice creates PrivateVault, and creates multiple PrivateVault's LienToken
2. alice monitors bob's creation of the PublicVault transaction, i.e., AstariaRouter.newPublicVault(), and then predicts the address of the newly generated treasure chest
Note: newPublicVAult() although the use of create(), but still can predict the address

see：https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed
```
The address for an Ethereum contract is deterministically computed from the address of its creator (sender) and how many transactions the creator has sent (nonce). The sender and nonce are RLP encoded and then hashed with Keccak-256.

```
3.front-run , and transfer LienToken to public vault predict address
4.bob's public vault created success and do some loan
5.alice do makePayment() to Corrupt bob's public vault



## Tools Used

## Recommended Mitigation Steps

The corresponding vault address is stored in s.lienMeta[id].orginOwner when the LienToken is created, this is not modified. Get the vault address from this variable, not from ownerOf(id).