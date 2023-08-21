## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-11

# [Understatement of `poolTotalPeUSDCirculation` amounts due to incorrect accounting after function `_repay` is called](https://github.com/code-423n4/2023-06-lybra-findings/issues/532) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L192-L210


# Vulnerability details

## Impact
Incorrect accounting of `poolTotalPeUSDCirculation`, resulting in an understatement of `poolTotalPeUSDCirculation` amounts. This causes inaccurate bookkeeping and in turn affects any other functions dependent on the use of `poolTotalPeUSDCirculation`. 

## Proof of Concept
We look at function `_repay` of `LybraPeUSDVaultBase.sol` as [follows](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L192-L210):

```solidity
 function _repay(address _provider, address _onBehalfOf, uint256 _amount) internal virtual {
     try configurator.refreshMintReward(_onBehalfOf) {} catch {}
     _updateFee(_onBehalfOf);
     uint256 totalFee = feeStored[_onBehalfOf];
     uint256 amount = borrowed[_onBehalfOf] + totalFee >= _amount ? _amount : borrowed[_onBehalfOf] + totalFee;
     if(amount >= totalFee) {
         feeStored[_onBehalfOf] = 0;
         PeUSD.transferFrom(_provider, address(configurator), totalFee);
         PeUSD.burn(_provider, amount - totalFee);
     } else {
         feeStored[_onBehalfOf] = totalFee - amount;
         PeUSD.transferFrom(_provider, address(configurator), amount);
     }
     try configurator.distributeRewards() {} catch {}
     borrowed[_onBehalfOf] -= amount;
     poolTotalPeUSDCirculation -= amount;


     emit Burn(_provider, _onBehalfOf, amount, block.timestamp);
 }
```

In particular, note the accounting of `poolTotalPeUSDCirculation` after repayment as [follows](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L207):

```solidity
        poolTotalPeUSDCirculation -= amount;
```

Consider a scenario per below for user Alice, where:

amount borrowed = 200
totalFee = 2

| Repay Scenario (PeUSD)        |         |
|-------------------------------|--------:|
| _amount input                 |     100 |
| totalFee                      |       2 |
| amount (repay)                |     100 |
| Fees left                     |       0 |
| PeUSD transfer to config addr |       2 |
| PeUSD burnt                   |      98 |
| borrowed[Alice]               |     100 |
| poolTotalPeUSDCirculation (X) | X - 100 | 

- Based on the accounting flow of the function, the fees incurred are transferred to `address(configurator)`. 
- The amount burned is `amount - totalFee`.
- However, we see that `poolTotalPeUSDCirculation` reduces the entire `amount` where it should be `amount - totalFee` reduced. 
- This results in an understatement of `poolTotalPeUSDCirculation` amounts. 

## Tools Used
Manual review

## Recommended Mitigation Steps

Correct the accounting as follows:

```solidity
   -     poolTotalPeUSDCirculation -= amount;
   +     poolTotalPeUSDCirculation -= (amount - totalFee);
```


## Assessed type

Error