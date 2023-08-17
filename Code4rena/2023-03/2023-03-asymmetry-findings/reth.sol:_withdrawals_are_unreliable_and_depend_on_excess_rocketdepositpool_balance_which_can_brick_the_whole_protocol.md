## Tags

- bug
- 3 (High Risk)
- disagree with severity
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- H-07

# [Reth.sol: Withdrawals are unreliable and depend on excess RocketDepositPool balance which can brick the whole protocol](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/210) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L107-L114


# Vulnerability details

## Impact
The Asymmetry protocol promises that a user can call [`SafETH.unstake`](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L108-L129) at all times. What I mean by that is that a user should be able at all times to burn his `SafETH` tokens and receive `ETH` in return. This requires that the derivatives held by the protocol can at all times be withdrawn (i.e. converted to `ETH`).  

Also the [`rebalanceToWeights`](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L138-L155) functionality requires that the derivatives can be withdrawn at all times. If a derivative cannot be withdrawn then the `rebalanceToWeights` function cannot be executed which means that the protocol cannot be adjusted to use different derivatives.  

For the `WstEth` and `SfrxEth` derivatives this is achieved by swapping the derivative in a Curve pool for `ETH`. The liquidity in the respective Curve pool ensures that withdrawals can be processed at all times.  

The `Reth` derivative works differently.  
Withdrawals are made by calling the `RocketTokenRETH.burn` function:  

[Link](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L107-L114)  
```solidity
function withdraw(uint256 amount) external onlyOwner {
    // @audit this is how rETH is converted to ETH
    RocketTokenRETHInterface(rethAddress()).burn(amount);
    // solhint-disable-next-line
    (bool sent, ) = address(msg.sender).call{value: address(this).balance}(
        ""
    );
    require(sent, "Failed to send Ether");
}
```

The issue with this is that the `RocketTokenRETH.burn` function only allows for _excess balance_ to be withdrawn. I.e. ETH that has been deposited by stakers but that is not yet staked on the Ethereum beacon chain. So Rocketpool allows users to burn `rETH` and withdraw `ETH` as long as the excess balance is sufficient.  

The issue is obvious now: If there is no excess balance because enough users burn `rETH` or the Minipool capacity increases, the Asymmetry protocol is bascially unable to operate.  

Withdrawals are then impossible which bricks `SafEth.unstake` and `SafEth.rebalanceToWeights`.  

## Proof of Concept
I show in this section how the current withdrawal flow for the `Reth` derivative is dependend on there being _excess balance_ in the RocketDepositPool.  

The current withdrawal flow calls `RocketTokenRETH.burn` which executes this code:  

[Link](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/token/RocketTokenRETH.sol#L106-L123)  
```solidity
function burn(uint256 _rethAmount) override external {
    // Check rETH amount
    require(_rethAmount > 0, "Invalid token burn amount");
    require(balanceOf(msg.sender) >= _rethAmount, "Insufficient rETH balance");
    // Get ETH amount
    uint256 ethAmount = getEthValue(_rethAmount);
    // Get & check ETH balance
    uint256 ethBalance = getTotalCollateral();
    require(ethBalance >= ethAmount, "Insufficient ETH balance for exchange");
    // Update balance & supply
    _burn(msg.sender, _rethAmount);
    // Withdraw ETH from deposit pool if required
    withdrawDepositCollateral(ethAmount);
    // Transfer ETH to sender
    msg.sender.transfer(ethAmount);
    // Emit tokens burned event
    emit TokensBurned(msg.sender, _rethAmount, ethAmount, block.timestamp);
}
```

This executes `withdrawDepositCollateral(ethAmount)`:  

[Link](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/token/RocketTokenRETH.sol#L126-L133)  

```solidity
function withdrawDepositCollateral(uint256 _ethRequired) private {
    // Check rETH contract balance
    uint256 ethBalance = address(this).balance;
    if (ethBalance >= _ethRequired) { return; }
    // Withdraw
    RocketDepositPoolInterface rocketDepositPool = RocketDepositPoolInterface(getContractAddress("rocketDepositPool"));
    rocketDepositPool.withdrawExcessBalance(_ethRequired.sub(ethBalance));
}
```

This then calls `rocketDepositPool.withdrawExcessBalance(_ethRequired.sub(ethBalance))` to get the `ETH` from the _excess balance_:  

[Link](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/deposit/RocketDepositPool.sol#L194-L206)  
```solidity
function withdrawExcessBalance(uint256 _amount) override external onlyThisLatestContract onlyLatestContract("rocketTokenRETH", msg.sender) {
    // Load contracts
    RocketTokenRETHInterface rocketTokenRETH = RocketTokenRETHInterface(getContractAddress("rocketTokenRETH"));
    RocketVaultInterface rocketVault = RocketVaultInterface(getContractAddress("rocketVault"));
    // Check amount
    require(_amount <= getExcessBalance(), "Insufficient excess balance for withdrawal");
    // Withdraw ETH from vault
    rocketVault.withdrawEther(_amount);
    // Transfer to rETH contract
    rocketTokenRETH.depositExcess{value: _amount}();
    // Emit excess withdrawn event
    emit ExcessWithdrawn(msg.sender, _amount, block.timestamp);
}
```

And this function reverts if the _excess balance_ is insufficient which you can see in the `require(_amount <= getExcessBalance(), "Insufficient excess balance for withdrawal");` check.  

## Tools Used
VSCode

## Recommended Mitigation Steps
The solution for this issue is to have an alternative withdrawal mechanism in case the _excess balance_ in the RocketDepositPool is insufficient to handle the withdrawal.  

The alternative withdrawal mechanism is to sell the `rETH` tokens via the Uniswap pool.  

You can use the [`RocketDepositPool.getExcessBalance`](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/deposit/RocketDepositPool.sol#L59-L67) to check if there is sufficient excess `ETH` to withdraw from Rocketpool or if the withdrawal must be made via Uniswap.  

The pseudocode of the new withdraw flow looks like this:  
```
function withdraw(uint256 amount) external onlyOwner {
    if (rocketDepositPool excess balance is sufficient) {
        RocketTokenRETHInterface(rethAddress()).burn(amount);
        // solhint-disable-next-line
        (bool sent, ) = address(msg.sender).call{value: address(this).balance}(
            ""
        );
        require(sent, "Failed to send Ether");
    } else {
        // swap rETH for ETH via Uniswap pool
    }
}
```

I also wrote the code for the changes that I suggest:  
```diff
diff --git a/contracts/SafEth/derivatives/Reth.sol b/contracts/SafEth/derivatives/Reth.sol
index b6e0694..b699d5c 100644
--- a/contracts/SafEth/derivatives/Reth.sol
+++ b/contracts/SafEth/derivatives/Reth.sol
@@ -105,11 +105,24 @@ contract Reth is IDerivative, Initializable, OwnableUpgradeable {
         @notice - Convert derivative into ETH
      */
     function withdraw(uint256 amount) external onlyOwner {
-        RocketTokenRETHInterface(rethAddress()).burn(amount);
-        // solhint-disable-next-line
-        (bool sent, ) = address(msg.sender).call{value: address(this).balance}(
-            ""
-        );
+        if (canWithdrawFromRocketPool(amount)) {
+            RocketTokenRETHInterface(rethAddress()).burn(amount);
+            // solhint-disable-next-line
+        } else {
+
+            uint256 minOut = ((((poolPrice() * amount) / 10 ** 18) *
+                ((10 ** 18 - maxSlippage))) / 10 ** 18);
+
+            IWETH(W_ETH_ADDRESS).deposit{value: msg.value}();
+            swapExactInputSingleHop(
+                rethAddress(),
+                W_ETH_ADDRESS,
+                500,
+                amount,
+                minOut
+            );
+        }
+        (bool sent, ) = address(msg.sender).call{value: address(this).balance}("");
         require(sent, "Failed to send Ether");
     }
 
@@ -149,6 +162,21 @@ contract Reth is IDerivative, Initializable, OwnableUpgradeable {
             _amount >= rocketDAOProtocolSettingsDeposit.getMinimumDeposit();
     }
 
+    function canWithdrawFromRocketPool(uint256 _amount) private view returns (bool) {
+        address rocketDepositPoolAddress = RocketStorageInterface(
+            ROCKET_STORAGE_ADDRESS
+        ).getAddress(
+                keccak256(
+                    abi.encodePacked("contract.address", "rocketDepositPool")
+                )
+            );
+        RocketDepositPoolInterface rocketDepositPool = RocketDepositPoolInterface(
+                rocketDepositPoolAddress
+            );
+        uint256 _ethAmount = RocketTokenRETHInterface(rethAddress()).getEthValue(_amount);
+        return rocketDepositPool.getExcessBalance() >= _ethAmount;
+    }
+
```