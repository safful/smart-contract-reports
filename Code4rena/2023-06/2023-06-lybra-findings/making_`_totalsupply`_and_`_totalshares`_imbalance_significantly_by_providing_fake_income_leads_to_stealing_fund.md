## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- H-05

# [Making `_totalSupply` and `_totalShares` imbalance significantly by providing fake income leads to stealing fund](https://github.com/code-423n4/2023-06-lybra-findings/issues/340) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L62


# Vulnerability details

## Impact

If the project is just started, a malicious user can make the `_totalSupply` and `_totalShares` imbalance significantly by providing fake income. By doing so, later, when innocent users deposit and mint, the malicious user can steal protocol's stETH without burning any share. Moreover, the protocol's income can be stolen as well.

## Proof of Concept

Suppose nothing is deposited in the protocol (it is day 0).

Bob (a malicious user) deposits $1000 worth of ether (equal to 1 ETH, assuming ETH price is $1000) to mint `200e18 + 1` eUSD. The state will be:
 - `shares[Bob] = 200e18 + 1`
 - `_totalShares = 200e18 + 1`
 - `_totalSupply = 200e18 + 1`
 - `borrowed[Bob] = 200e18 + 1`
 - `poolTotalEUSDCirculation = 200e18 + 1`
 - `depositAsset[Bob] = 1e18`
 - `totalDepositedAsset = 1e18`
 - `stETH.balanceOf(protocol) = 1e18`
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L37

Then, Bob transfers directly `0.2stETH` (worth $200) to the protocol. By doing so, Bob is providing a fake excess income in the protocol. So, the state will be:
 - `shares[Bob] = 200e18 + 1`
 - `_totalShares = 200e18 + 1`
 - `_totalSupply = 200e18 + 1`
 - `borrowed[Bob] = 200e18 + 1`
 - `poolTotalEUSDCirculation = 200e18 + 1`
 - `depositAsset[Bob] = 1e18`
 - `totalDepositedAsset = 1e18`
 - `stETH.balanceOf(protocol) = 1e18 + 2e17`

Then, Bob calls `excessIncomeDistribution` to buy this excess income. As you see in line 63, the `excessIncome` is equal to the difference of `stETH.balanceOf(protocol)` and `totalDepositedAsset`. So, the `excessAmount = 2e17`.
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L63

Then, in line 66, this amount `2e17` is converted to eUSD amount based on the price of stETH. Since, we assumed  ETH is $1000, we have:
```
uint256 payAmount = (((realAmount * getAssetPrice()) / 1e18) * getDutchAuctionDiscountPrice()) / 10000 = 2e17 * 1000e18 / 1e18 = 200e18
```
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L66C9-L66C112

Since the protocol is just started, there is no `feeStored`, so the income is equal to zero.
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L68

In line 75, we have: 
```
uint256 sharesAmount = _EUSDAmount.mul(_totalShares).div(totalMintedEUSD) = 200e18 * (200e18 + 1) / (200e18 + 1) = 200e18
```
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L75C13-L75C35

In line 81, this amount of `sharesAmount` will be burned from Bob, and then in line 93, `2e17` stETH will be transferred to Bob. So, the state will be:
 - `shares[Bob] = 200e18 + 1 - 200e18 = 1`
 - `_totalShares = 200e18 + 1 - 200e18 = 1`
 - `_totalSupply = 200e18 + 1`
 - `borrowed[Bob] = 200e18 + 1`
 - `poolTotalEUSDCirculation = 200e18 + 1`
 - `depositAsset[Bob] = 1e18`
 - `totalDepositedAsset = 1e18`
 - `stETH.balanceOf(protocol) = 1e18 + 2e17 - 2e17 = 1e18`
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L81
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L93

**Please note** that currently we have `_totalSupply = 200e18 + 1` and `_totalShares = 1`.

Suppose, Alice (an innocent user) deposits 10ETH, and mints 4000e18 eUSD. So, the amount of shares minted to Alice will be:
```
sharesAmount = _EUSDAmount.mul(_totalShares).div(totalMintedEUSD) = 4000e18 * 1 / (200e18 + 1) = 19
```
So, the state will be:
 - `shares[Bob] = 1`
 - `_totalShares = 1 + 19 = 20`
 - `_totalSupply = 200e18 + 1 + 4000e18 = 4200e18 + 1`
 - `borrowed[Bob] = 200e18 + 1`
 - `poolTotalEUSDCirculation = 200e18 + 1 + 4000e18 = 4200e18 + 1`
 - `depositAsset[Bob] = 1e18`
 - `totalDepositedAsset = 1e18 + 10e18 = 11e18`
 - `stETH.balanceOf(protocol) = 1e18 + 10e18 = 11e18`
 - `shares[Alice] = 19`
 - `borrowed[Alice] = 4000e18`
 - `depositAsset[Alice] = 10e18`

Now, different issues can happen leading to loss/steal of funds:

 - **First Scenario:** If Alice is a provider, Bob can redeem eUSD multiple of times to receive stETH without burning any share by calling `rigidRedemption`. To be more accurate, Bob should call this function with `eusdAmount` as parameter equal to `_totalSupply / _totalShares`. For example:
   - first call: `rigidRedemption (Alice, 210e18)`, so we will have:
      - `shares[Bob] = 1`
      - `_totalShares  = 20`
      - `_totalSupply = 4200e18 + 1 - 210e18 = 3990e18 + 1`
      - `borrowed[Bob] = 200e18 + 1`
      - `poolTotalEUSDCirculation = 4200e18 + 1 - 210e18 = 3990e18 + 1`
      - `depositAsset[Bob] = 1e18`
      - `totalDepositedAsset = 11e18 - 21e16`
      - `stETH.balanceOf(protocol) = 11e18 - 21e16`
      - `shares[Alice] = 19`
      - `borrowed[Alice] = 4000e18 - 210e18 = 3790e18`
      - `depositAsset[Alice] = 10e18 - 21e16`
      Please note that no shares are burned from Bob, because `getSharesByMintedEUSD` returns zero as `210e18 * 20 / (4200e18 + 1) = 0`. It means, Bob receives 0.21 stETH by burning no shares.

   - second call: `rigidRedemption (Alice, 199e18)`, so we will have:
      - `shares[Bob] = 1`
      - `_totalShares  = 20`
      - `_totalSupply = 3990e18 + 1 - 199e18 = 3791e18 + 1`
      - `borrowed[Bob] = 200e18 + 1`
      - `poolTotalEUSDCirculation = 3990e18 + 1 - 199e18 = 3791e18 + 1`
      - `depositAsset[Bob] = 1e18`
      - `totalDepositedAsset = 11e18 - 210e15 - 199e15 = 11e18 - 409e15`
      - `stETH.balanceOf(protocol) = 11e18 - 210e15 - 199e15 = 11e18 - 409e15`
      - `shares[Alice] = 19`
      - `borrowed[Alice] = 3790e18 - 199e18 = 3591e18`
      - `depositAsset[Alice] = 10e18 - 210e15 - 199e15 = 10e18 - 409e15`
      Please note that no shares are burned from Bob, because `getSharesByMintedEUSD` returns zero as `199e18 * 20 / (3990e18 + 1) = 0`. It means, Bob receives 0.199 stETH by burning no shares.

   - third call: `rigidRedemption (Alice, 189e18)`, so we will have:
      - `shares[Bob] = 1`
      - `_totalShares  = 20`
      - `_totalSupply = 3791e18 + 1 - 189e18 = 3602e18 + 1`
      - `borrowed[Bob] = 200e18 + 1`
      - `poolTotalEUSDCirculation = 3791e18 + 1 - 189e18 = 3602e18 + 1`
      - `depositAsset[Bob] = 1e18`
      - `totalDepositedAsset = 11e18 - 409e15 - 189e15 = 11e18 - 598e15`
      - `stETH.balanceOf(protocol) = 11e18 - 409e15 - 189e15 = 11e18 - 598e15`
      - `shares[Alice] = 19`
      - `borrowed[Alice] = 3591e18 - 189e18 = 3402e18`
      - `depositAsset[Alice] = 10e18 - 409e15 - 189e15 = 10e18 - 598e15`
      Please note that no shares are burned from Bob, because `getSharesByMintedEUSD` returns zero as `189e18 * 20 / (3791e18 + 1) = 0`. It means, Bob receives 0.189 stETH by burning no shares.

   So far, by just calling the function `rigidRedemption` three times, Bob received `0.21 + 0.199 + 0.189 = 0.598` stETH (worths $598). If, bob continues calling this function, his gain will increase more and more to the point that `_totalSupply` and `_totalShares` become closer to each other.

   A simple calculation shows that if Bob calls this function 60 times (for sure each time the input parameter should be adjusted based on the `_totalSupply` and `_totalShares`), the state will be:
      - `shares[Bob] = 1`
      - `_totalShares  = 20`
      - `_totalSupply = 203.7e18`
      - `borrowed[Bob] = 200e18 + 1`
      - `poolTotalEUSDCirculation = 203.7e18`
      - `depositAsset[Bob] = 1e18`
      - `totalDepositedAsset = 7e18`
      - `stETH.balanceOf(protocol) = 7e18`
      - `shares[Alice] = 19`
      - `borrowed[Alice] = 3.7e18`
      - `depositAsset[Alice] = 6e18`
   It shows that almost the gain of Bob is 4 stETH (worth $4000).

   The following code simply shows that how this repetitive calling of `rigidRedemption` works:
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract LybraPoC {
    mapping(address => uint256) public borrowed;
    mapping(address => uint256) public shares;
    address public Alice = address(1);
    address public Bob = address(2);
    uint256 public bobGain;
    uint256 public num;

    function redeem() public {
        uint256 toBeRedeemed;
        uint256 requiredShares;
        uint256 _totalSupply = 4200e18 + 1;
        uint256 _totalShares = 20;
        shares[Bob] = 1;
        shares[Alice] = 19;
        borrowed[Bob] = 200e18 + 1;
        borrowed[Alice] = 4000e18;
        while (true) {
            num++;
            toBeRedeemed = (_totalSupply % _totalShares == 0)
                ? (_totalSupply / _totalShares) - 1
                : (_totalSupply / _totalShares);
            requiredShares = (toBeRedeemed * _totalShares) / _totalSupply;
            if (toBeRedeemed > borrowed[Alice]) {
                break;
            }
            borrowed[Alice] -= toBeRedeemed;
            _totalSupply -= toBeRedeemed;
            _totalShares -= requiredShares;
            shares[Bob] -= requiredShares;
            bobGain += toBeRedeemed;
        }
    }
}

```

   Please note that, Bob does not have enough share to repay his borrow and release all his collateral. So, assuming safe collateral rate is 160%, Bob at most can withdraw `1 ETH - 1.6 * (200e18 + 1) = $680`. He also gained $4000 by redeeming Alice 60 times, so Bob's balance now is: `$680 + $4000 = $4680` which means $3680 is his total gain that is stolen from the protocol. In other words, protocol has minted some shares without enough stETH backed. 

   Please note that Bob can now start to repay his borrow to reduce `borrowed[Bob]` step by step, without burning any share. For example, first repays 10e18 eUSD, second repays 9e18 eUSD. But, for simplicity, I ignored this calculation, and just focused on redeeming Alice to steal big fund. By repaying multiple of times, `_totalSupply` and `_totalShares` become closer to each other. Then again it is possible to make it imbalance by providing fake income and attack the next users. So, this attack can be applied multiple of times without any restriction.

   Please note that Alice is just an example of all the providers in the protocol. If there are other non-provider users also, this scenario is still valid. 

 - **Second Scenario:** If Alice is liquidated, Bob can liquidate her without burning share again similar to the mechanism described during redeeming.

 - **Third Scenario:** Please note that if another innocent user (Eve) is also involved in our example, she will lose fund as well. So, let's say that Eve deposited 20 ETH, and also minted 10000e18 eUSD. So, the state will be:
   - `shares[Bob] = 1`
   - `_totalShares = 20 + 47 = 67`
   - `_totalSupply = 4200e18 + 1 + 10000e18 = 14200e18 + 1`
   - `borrowed[Bob] = 200e18 + 1`
   - `poolTotalEUSDCirculation = 4200e18 + 1 + 10000e18 = 14200e18 + 1`
   - `depositAsset[Bob] = 1e18`
   - `totalDepositedAsset = 11e18 + 20e18 = 31e18`
   - `stETH.balanceOf(protocol) = 11e18 + 20e18 = 31e18`
   - `shares[Alice] = 19`
   - `borrowed[Alice] = 4000e18`
   - `depositAsset[Alice] = 10e18`
   - `shares[Eve] = 47`
   - `borrowed[Eve] = 10000e18`
   - `depositAsset[Eve] = 20e18`
  Now, suppose only Alice is provider, and Eve is not. So, we can redeem Alice by using the same mechanism we describe in the first scenario. Using the same piece of code for repetitive redemption, we have:
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract LybraPoC {
    mapping(address => uint256) public borrowed;
    mapping(address => uint256) public shares;
    address public Alice = address(1);
    address public Bob = address(2);
    address public Eve = address(3);
    uint256 public bobGain;
    uint256 public num;
    uint256 public _totalSupply;
    uint256 public _totalShares;

    function redeem() public {
        uint256 toBeRedeemed;
        uint256 requiredShares;
        _totalSupply = 14200e18 + 1;
        _totalShares = 67;
        shares[Bob] = 1;
        shares[Alice] = 19;
        shares[Eve] = 47;
        borrowed[Bob] = 200e18 + 1;
        borrowed[Alice] = 4000e18;
        borrowed[Eve] = 10000e18;

        while (true) {
            num++;
            toBeRedeemed = (_totalSupply % _totalShares == 0)
                ? (_totalSupply / _totalShares) - 1
                : (_totalSupply / _totalShares);
            requiredShares = (toBeRedeemed * _totalShares) / _totalSupply;
            if (toBeRedeemed > borrowed[Alice]) {
                break;
            }
            borrowed[Alice] -= toBeRedeemed;
            _totalSupply -= toBeRedeemed;
            _totalShares -= requiredShares;
            shares[Bob] -= requiredShares;
            bobGain += toBeRedeemed;
        }
    }
}

```
After redeeming Alice 23 times, the state will be:
   - `shares[Bob] = 1`
   - `_totalShares = 20 + 47 = 67`
   - `_totalSupply = 10200.2e18`
   - `borrowed[Bob] = 200e18 + 1`
   - `poolTotalEUSDCirculation = 10200.2e18 + 1`
   - `depositAsset[Bob] = 1e18`
   - `totalDepositedAsset = 27e18`
   - `stETH.balanceOf(protocol) = 27e18`
   - `shares[Alice] = 19`
   - `borrowed[Alice] = 2.1e17`
   - `depositAsset[Alice] = 6e18`
   - `shares[Eve] = 47`
   - `borrowed[Eve] = 10000e18`
   - `depositAsset[Eve] = 20e18`

  Now if Eve wants to repay her whole borrowed amount, she should burn almost 65 shares: `10000e18 * 67 / 10200e18`, but she has only 47 shares. So, she can only repay at most 7155e18 of her borrow. It means that Eve's fund is stolen by Bob. In other words, the collateralized ETH are taken by Bob without burning any shares, so the left shares do not have enough ETH backed.

  This scenario shows that Bob made `_totalSupply` and `_totalShares` imbalance, then two innocent users deposited in the protocol and borrowed some eUSD. Since the difference between these two `_totalSupply` and `_totalShares` is large, small amount of shares are minted. Then, Bob redeemed some amount through the user who was provider. By doing so, the values of `_totalSupply` and `_totalShares` become closer to each other. Now if the second user intends to repay her borrow, she should burn more shares that she owns (because the difference of the values `_totalSupply` and `_totalShares` is now smaller).

  - **Fourth Scenario:** Alice can not transfer less than 210e18 eUSD. Because, in the function `_trasnfer`, `_sharesToTransfer = 209e18 * 20 / (4200e18 + 1) = 0`
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/EUSD.sol#L153
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/EUSD.sol#L349

 - **Fifth Scenario:** If protocol stETH balance increases by 0.1stETH through LSD after some time. Bob can buy this income without burning any share, in other words Bob steals the income of the protocol. The flow is as follows:
   - Bob calls `excessIncomeDistribution(1e17)`.
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L62C14-L62C38
   - The `payAmount` will be `100e18`.
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L66
   - If `income >= payAmount`, then `payAmount` should be transferred from Bob to the configurator address.
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L85
   - In the `_transfer`, `100e18` will be converted to shares: `_sharesToTransfer = 100e18 * 20 / (4200e18 + 1) = 0`. So, 0 shares will be deducted from Bob, but 0.1 stETH will be transferred to him.
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/EUSD.sol#L348




Please note that for sake of simplicity the fees related to the redemption/liquidation are ignored. So, considering those into our calculation does not make the scenarios invalid. 


**In Summary:**

Bob makes `_totalSupply` and `_totalShares` imbalance significantly, by just providing fake income in the protocol at day 0. Now that it is imbalanced, he can redeem, or liquidate users without burning any shares. He can also steal protocol's income fund without burning any shares. 


## Tools Used

## Recommended Mitigation Steps
**First Fix:**
During the `_repay`, it should return the amount of burned shares.
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L279

So that in the functions `liquidation`, `superLiquidation`, and `rigidRedemption`, again the burned shares should be converted to eUSD, and this amount should be used for the rest of calculations.
```
function rigidRedemption(address provider, uint256 eusdAmount) external virtual {
        // ...
        uint256 brnedShares = _repay(msg.sender, provider, eusdAmount);
        eusdAmount = getMintedEUSDByShares(brnedShares);
        //...
    }
```
**Second Fix:**
In the `excessIncomeDistribution`, the same check 
```
uint256 sharesAmount = EUSD.getSharesByMintedEUSD(payAmount - income);
            if (sharesAmount == 0) {
                //EUSD totalSupply is 0: assume that shares correspond to EUSD 1-to-1
                sharesAmount = (payAmount - income);
            }
```
should be included in the else body as well.
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraStETHVault.sol#L75-L79























## Assessed type

Context