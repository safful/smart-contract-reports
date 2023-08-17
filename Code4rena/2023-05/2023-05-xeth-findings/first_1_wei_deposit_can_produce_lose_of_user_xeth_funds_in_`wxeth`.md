## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor acknowledged
- M-10

# [First 1 wei deposit can produce lose of user xETH funds in `wxETH`](https://github.com/code-423n4/2023-05-xeth-findings/issues/3) 

# Lines of code

https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/wxETH.sol#L94-L108


# Vulnerability details

## Description
The present implementation of the `wxETH::stake` functions permits the sending of tokens to the contract, even if the quantity of wxETH is zero. This can result in users losing funds, particularly when the initial deposit is only 1 wei, and the extent to which xETH is dripped (alongside its dripping period) is taken into consideration.

The issue arises when `xETHAmount * BASE_UNIT) < exchangeRate()` during the second deposit.

## Impact
There is a potential risk of losing funds in certain feasible scenarios due to the incorrect handling of round-down situations.

## POC
[Here a coded POC](https://gist.github.com/carlitox477/a5bd6345b4854da91fe5f80217a18aed) that shows what would happen if:
1. The protocol wants to drip 70 xETH in its first week as a way of promotion
2. Alice realized about this and decide to make the first deposit (by frontrunning every other deposits)
3. The second deposit of $104166666666666601\; wei$ happens 15 minutes after Alice deposit

Here the output of the coded POC:
```
Logs:
  Min amount to deposit to get a single share: 104166666666666601 wei
  Bob xETH before staking: 104166666666666600
  Bob wxETH after staking balance 0
```

### Other scenarios
Here are the numbers of other scenario:
|                               |     **1 eth in 1 year**     |                |  **10 eth in 1 year** |               | **100 eth in 1 year** |              | **1000 eth in 1 year** |             |
|:-----------------------------:|:---------------------------:|:--------------:|:---------------------:|:-------------:|:---------------------:|:------------:|:----------------------:|:-----------:|
| **Time until second deposit** | Min amount to deposit (ETH) |   Value(USD)   | Min amount to deposit |   Value(USD)  | Min amount to deposit |  Value(USD)  |  Min amount to deposit |  Value(USD) |
|            1 minute           |      0,000001902587519      | 0,003424657534 |    0,00001902587519   | 0,03424657534 |    0,0001902587519    | 0,3424657534 |     0,001902587519     | 3,424657534 |
|           15 minutes          |       0,00002853881279      |  0,05136986301 |    0,0002853881279    |  0,5136986301 |     0,002853881279    |  5,136986301 |      0,02853881279     | 51,36986301 |
|             1 hour            |       0,0001141552511       |  0,2054794521  |     0,001141552511    |  2,054794521  |     0,01141552511     |  20,54794521 |      0,1141552511      | 205,4794521 |
|            24 hours           |        0,002739726027       |   4,931506849  |     0,02739726027     |  49,31506849  |      0,2739726027     |  493,1506849 |       2,739726027      | 4931,506849 |
|             1 week            |        0,01917808219        |   34,52054795  |      0,1917808219     |  345,2054795  |      1,917808219      |  3452,054795 |       19,17808219      | 34520,54795 |

|                               |     **70 eth in 1 week**    |            |
|:-----------------------------:|:---------------------------:|:----------:|
| **Time until second deposit** | Min amount to deposit (ETH) | Value(USD) |
|            1 minute           |        0,006944444444       |    12,5    |
|           15 minutes          |         0,1041666667        |    187,5   |
|             1 hour            |         0,4166666667        |     750    |

## Mitigation steps
Check `mintAmount > 0` in function `wxETH::stake`. This means:
```diff
    function stake(uint256 xETHAmount) external drip returns (uint256) {
        /// @dev calculate the amount of wxETH to mint
        uint256 mintAmount = previewStake(xETHAmount);

+       if (mintAmount == 0){
+           revert ERROR_MINTING_0_SHARES();
+       }

        /// @dev transfer xETH from the user to the contract
        xETH.safeTransferFrom(msg.sender, address(this), xETHAmount);

        /// @dev emit event
        emit Stake(msg.sender, xETHAmount, mintAmount);

        /// @dev mint the wxETH to the user
        _mint(msg.sender, mintAmount);

        return mintAmount;
    }
```


## Assessed type

Other