## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-03

# [Overflow can still happened when calculating `priceX8` inside `poolMatchesOracle` operation](https://github.com/code-423n4/2023-08-goodentry-findings/issues/140) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/GeVault.sol#L367-L378


# Vulnerability details

## Impact
`poolMatchesOracle` is used to compare price calculated from uniswap v3 pool and chainlink oracle and decide whether rebalance should happened or not. `priceX8` will be holding price information calculated using `sqrtPriceX96` and when operations is performed, it will try to scale down using `2 ** 12`. However, the scale down is not enough and overflow can still happened.

## Proof of Concept
Consider this scenario, The GeVault is using WBTC for `token0` and WETH for `token1`.

These are information for the WBTC/WETH from uniswap v3 pool (0x4585FE77225b41b697C938B018E2Ac67Ac5a20c0):

slot0 data (at current time) :

```
sqrtPriceX96   uint160 :  31520141554881197083247204479961147
```

`token0` (WBTC) decimals is 8 and `token1` (WETH) decimals is 18.

Using these information, try to reproduce the `priceX8` calculation :

```solidity
    function testOraclePrice() public {
        uint160 sqrtPriceX96 = 31520141554881197083247204479961147;
        // decimals0 is 8
        uint priceX8 = 10 ** 8;
        // Overflow if dont scale down the sqrtPrice before div 2*192 
        // @audit - the overflow still possible
        priceX8 =
            (priceX8 * uint(sqrtPriceX96 / 2 ** 12) ** 2 * 1e8) /
            2 ** 168;
        // decimals1 is 18
        priceX8 = priceX8 / 10 ** 18;
        assertEq(true, true);
    }
```

the test result in overflow :

```
[FAIL. Reason: Arithmetic over/underflow] testOraclePrice() 
```

This will cause calculation still overflow, even using the widely used WBTC/WETH pair 

## Tools Used

Manual review

## Recommended Mitigation Steps

Consider to change the scale down using the recommended value from uniswap v3 library:

https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol#L49-L69

or change the scale down similar to the one used inside library

```diff
  function poolMatchesOracle() public view returns (bool matches){
    (uint160 sqrtPriceX96,,,,,,) = uniswapPool.slot0();
    
    uint decimals0 = token0.decimals();
    uint decimals1 = token1.decimals();
    uint priceX8 = 10**decimals0;
    // Overflow if dont scale down the sqrtPrice before div 2*192
-    priceX8 = priceX8 * uint(sqrtPriceX96 / 2 ** 12) ** 2 * 1e8 / 2**168;
+    priceX8 = priceX8 * (uint(sqrtPriceX96) ** 2 / 2 ** 64) * 1e8 / 2**128;
    priceX8 = priceX8 / 10**decimals1;
    uint oraclePrice = 1e8 * oracle.getAssetPrice(address(token0)) / oracle.getAssetPrice(address(token1));
    if (oraclePrice < priceX8 * 101 / 100 && oraclePrice > priceX8 * 99 / 100) matches = true;
  }
```








## Assessed type

Math