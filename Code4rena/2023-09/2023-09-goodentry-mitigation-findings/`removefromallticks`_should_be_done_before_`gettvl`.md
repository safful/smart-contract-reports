## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- MR-H-02

# [`removeFromAllTicks` should be done before `getTVL`](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/57) 

# Lines of code

https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/GeVault.sol#L265-L293


# Vulnerability details

After the mitigation, the TR fee is directly sent to GE vault. Suppose 0.1 eth trading fee has accumulated in TR.

    uint vaultValueX8 = getTVL();   
    uint adjBaseFee = getAdjustedBaseFee(token == address(token0));
    // Wrap if necessary and deposit here
    if (msg.value > 0){
      require(token == address(WETH), "GEV: Invalid Weth");
      // wraps ETH by sending to the wrapper that sends back WETH
      WETH.deposit{value: msg.value}();
      amount = msg.value;
    }
    else { 
      ERC20(token).safeTransferFrom(msg.sender, address(this), amount);
    }
    
    // Send deposit fee to treasury
    uint fee = amount * adjBaseFee / 1e4;
    ERC20(token).safeTransfer(treasury, fee);
    uint valueX8 = oracle.getAssetPrice(token) * (amount - fee) / 10**ERC20(token).decimals();


    require(tvlCap > valueX8 + vaultValueX8, "GEV: Max Cap Reached");


    uint tSupply = totalSupply();
    // initial liquidity at 1e18 token ~ $1
    if (tSupply == 0 || vaultValueX8 == 0)
      liquidity = valueX8 * 1e10;
    else {
      liquidity = tSupply * valueX8 / vaultValueX8;
    }
    
    rebalance();

As above, when depositing, the 0.1 eth fee is not reflected in `getTVL`. Only after `removeFromAllTicks`(in `rebalance`) will the fee be collected and sent to GE vault. Therefore, attacker can take a flashloan, deposit and then withdraw to steal almost all of the 0.1 eth trading fee. (the process is similar to what H-04 has described)

When withdrawing, similarly, user will incur loss since latest trading fee is not accounted.


## Assessed type

Context