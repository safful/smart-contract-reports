## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-12

# [ReaperVaultERC4626 is not EIP-4626 compliant and integrations can result in loss of funds](https://github.com/code-423n4/2023-02-ethos-findings/issues/247) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L368-L408


# Vulnerability details

## Impact
Contracts that integrate with the `ReaperVaultERC4626` vault (including Ethos contracts) may wrongly assume that the functions are EIP-4626 compliant, which it might cause integration problems in the future, that can lead to a wide range of issues for both parties, including loss of funds.

## Proof of Concept

This PoC describes this issue for the `withdraw` function, but there is the same problem with the `redeem` function.

EIP-4626 specification says that the `withdraw` function:

```
// Burns shares from owner and sends exactly assets of underlying tokens to receiver.

// MUST revert if all of assets cannot be withdrawn (due to withdrawal limit being reached, slippage,
// the owner not having enough shares, etc).
```

This is the `withdraw` function:

```solidity
File: Ethos-Vault\contracts\ReaperVaultERC4626.sol

202:     function withdraw(
203:         uint256 assets,
204:         address receiver,
205:         address owner
206:     ) external override returns (uint256 shares) {
207:         shares = previewWithdraw(assets); // previewWithdraw() rounds up so exactly "assets" are withdrawn and not 1 wei less
208:         if (msg.sender != owner) _spendAllowance(owner, msg.sender, shares);
209:         _withdraw(shares, receiver, owner);
210:     }
```

When the internal `_withdraw` is called, the `value` represents the total amount of assets that will be transferred to the receiver. 
There is a special case where there could be a withdrawal that exceeds the total balance of the vault:

```solidity
File: Ethos-Vault\contracts\ReaperVaultV2.sol

368:         if (value > token.balanceOf(address(this))) {
369:             uint256 totalLoss = 0;
370:             uint256 queueLength = withdrawalQueue.length;
371:             uint256 vaultBalance = 0;
372:             for (uint256 i = 0; i < queueLength; i = i.uncheckedInc()) {
373:                 vaultBalance = token.balanceOf(address(this));
374:                 if (value <= vaultBalance) {
375:                     break;
376:                 }
377: 
378:                 address stratAddr = withdrawalQueue[i];
379:                 uint256 strategyBal = strategies[stratAddr].allocated;
380:                 if (strategyBal == 0) {
381:                     continue;
382:                 }
383: 
384:                 uint256 remaining = value - vaultBalance;
385:                 uint256 loss = IStrategy(stratAddr).withdraw(Math.min(remaining, strategyBal));
386:                 uint256 actualWithdrawn = token.balanceOf(address(this)) - vaultBalance;
387: 
388:                 // Withdrawer incurs any losses from withdrawing as reported by strat
389:                 if (loss != 0) {
390:                     value -= loss;
391:                     totalLoss += loss;
392:                     _reportLoss(stratAddr, loss);
393:                 }
394: 
395:                 strategies[stratAddr].allocated -= actualWithdrawn;
396:                 totalAllocated -= actualWithdrawn;
397:             }
398: 
399:             vaultBalance = token.balanceOf(address(this));
400:             if (value > vaultBalance) {
401:                 value = vaultBalance;
402:             }
403: 
404:             require(
405:                 totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR,
406:                 "Withdraw loss exceeds slippage"
407:             );
408:         }
409: 
410:         token.safeTransfer(_receiver, value);
```
In this case, if a strategy incurs any losses, the actual withdrawal amount will NOT be the exact same specified when calling the `withdraw` function, as it will be less than that, as **the loss is detracted from the withdrawn value**:

```solidity
File: Ethos-Vault\contracts\ReaperVaultV2.sol

388:                 // Withdrawer incurs any losses from withdrawing as reported by strat
389:                 if (loss != 0) {
390:                     value -= loss;
391:                     totalLoss += loss;
392:                     _reportLoss(stratAddr, loss);
393:                 }
```
If that happens, then `assets requested > assets received`.

As the specification says that the withdrawal process `MUST revert if all of assets cannot be withdrawn (due to withdrawal limit being reached, slippage, the owner not having enough shares, etc)`, this makes the `ReaperVaultERC4626` non EIP-4626 compliant.

This might cause integration problems in the future, which can lead to a wide range of issues, including loss of funds.

Because EIP-4626 is aimed to create a consistent and robust implementation pattern for Tokenized Vaults, and even a slight deviation from the standard would break composability (and potentially lead to a loss of funds), this is categorized as a high risk.

## Tools Used
Manual review

## Recommended Mitigation Steps
The `withdraw` and `redeem` functions should be modified to meet the specifications of EIP-4626: the `value` transferred must always be equal to the `assets` withdrawn. In case this is not true, the transaction must be reverted.