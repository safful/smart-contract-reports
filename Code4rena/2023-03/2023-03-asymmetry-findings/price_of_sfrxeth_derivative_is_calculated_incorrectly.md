## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- H-04

# [Price of sfrxEth derivative is calculated incorrectly](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/641) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L115-L116
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L74-L75
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L73
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L92


# Vulnerability details

## Impact
In the [ethPerDerivative()](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L111-L117), the calculated ```frxAmount``` is multiplied by (10 ** 18) and divided by ```price_oracle```, but it must be multiplied by ```price_oracle``` and divided by (10 ** 18).

The impact is severe as [ethPerDerivative()](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L111-L117) function is used in [stake()](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L63-L101), one of  two main functions a user will interact with. The value returned by [ethPerDerivative()](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L111-L117) affects the calculations of [```mintAmount```](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L98). The incorrect calculation may over or understate the amount of safEth received by the user.  

[ethPerDerivative()](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L111-L117) is also used in the [withdraw()](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L60-L88) function when calculating [```minOut```](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L74). So, incorrect calculation of ethPerDerivative() may increase/decrease slippage. This can cause unexpected losses or function revert. If [withdraw()](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L60-L88) function reverts, the function [unstake()](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L108-L129) is unavailable => assets are locked.

## Proof of Concept
We need to calculate: (10 ** 18) sfrxEth = X Eth.  
For example, we ```convertToAssets(10 ** 18)``` and get ```frxAmount``` = 1031226769652703996. ```price_oracle``` returns 998827832404234820. So, (10 ** 18) frxEth costs 998827832404234820 Eth. Thus, (10 ** 18) sfrxEth costs ```frxAmount * price_oracle / 10 ** 18``` = 1031226769652703996 * 998827832404234820 / 10 ** 18 Eth (1030017999049431492 Eth).  

But [this function](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L111-L117):
```
    function ethPerDerivative(uint256 _amount) public view returns (uint256) {
        uint256 frxAmount = IsFrxEth(SFRX_ETH_ADDRESS).convertToAssets(
            10 ** 18
        );
        return ((10 ** 18 * frxAmount) /
            IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).price_oracle());
    }
```
calculates the cost of sfrxEth as ```10 ** 18 * frxAmount / price_oracle ``` = 10 ** 18 * 1031226769652703996 / 998827832404234820 Eth (1032436958800480269 Eth). The current difference ~ 0.23% but it can be more/less.

## Tools Used
Manual review.

## Recommended Mitigation Steps
Change [these lines](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L115-L116):
```
        return ((10 ** 18 * frxAmount) /
            IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).price_oracle());
```
to:
```
        return (frxAmount * IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).price_oracle() / 10 ** 18);
```