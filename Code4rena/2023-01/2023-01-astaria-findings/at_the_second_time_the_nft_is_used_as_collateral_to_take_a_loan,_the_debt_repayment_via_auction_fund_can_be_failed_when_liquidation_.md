## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- H-09

# [At the second time the nft is used as collateral to take a loan, the debt repayment via auction fund can be failed when liquidation ](https://github.com/code-423n4/2023-01-astaria-findings/issues/379) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L143-L146


# Vulnerability details

## Proof of concept 
When a user transfer an NFT to `CollateralToken` contract, it will toggle the function `CollateralToken.onERC721Received()`. In this function if there didn't exist any `clearingHouse` for the `collateralId`, it will create a new one for that collateral. 
```solidity=
if (s.clearingHouse[collateralId] == address(0)) {
    address clearingHouse = ClonesWithImmutableArgs.clone(
      s.ASTARIA_ROUTER.BEACON_PROXY_IMPLEMENTATION(),
      abi.encodePacked(
        address(s.ASTARIA_ROUTER),
        uint8(IAstariaRouter.ImplementationType.ClearingHouse),
        collateralId
      )
    );

    s.clearingHouse[collateralId] = clearingHouse;
  }
```
The interesting thing of this technique is: there will be **just one `clearingHouse`** be used for each collateral no matter how many times the collateral is transferred to the contract. Even when the lien is liquidated / fully repayed, the `s.clearingHouse[collateralId]` remain unchanged. 

The question here is any stale datas in `clearingHouse` from the previous time that the nft was used as collateral can affect the behavior of protocol when the nft was transfered to CollateralToken again ? 
Let take a look at the function [`ClearingHouse._execute()`](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L114). In this function, the implementation uses `safeApprove()` to approve `payment - liquidatorPayment` amount for the `TRANSFER_PROXY`. 
```solidity=
ERC20(paymentToken).safeApprove(
  address(ASTARIA_ROUTER.TRANSFER_PROXY()),
  payment - liquidatorPayment
);
```
the `safeApprove` function will revert if the allowance was set from non-zero value to non-zero value. This will incur some potential risk for the function like example below: 
1. NFT x is transferred to `CollateralToken` to take loans and then it is liquidated.
2. At time 10, function `ClearingHouse._execute()` was called and the `payment - liquidatorPayment > totalDebt`. This will the `paymentToken.allowance[clearingHouse][TRANSFER_PROXY] > 0` after the function ended. 
3. NFT x is transferred to `CollateralToken` for the second time to take a loans and then it is liquidated again. 
4. At time 15 (> 10), function `ClearingHouse._execute()` was called, but at this time, the `safeApprove` will revert since the previous allowance is different from 0 

## Impact
The debt can be repayed by auction funds when liquidation.

## Tools Used
Manual review 
  
## Recommended Mitigation Steps
Consider to use `approve` instead of `safeApprove`.