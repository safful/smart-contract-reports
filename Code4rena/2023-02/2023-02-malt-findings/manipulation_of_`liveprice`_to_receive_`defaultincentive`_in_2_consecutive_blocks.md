## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- H-03

# [Manipulation of `livePrice` to receive `defaultIncentive` in 2 consecutive blocks](https://github.com/code-423n4/2023-02-malt-findings/issues/14) 

# Lines of code

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L312


# Vulnerability details

## Impact
In StabilizerNode, the default behaviour when twap is below the lower peg threshold all transfers to the amm pool are blocked. However when `usePrimedWindow = true`, it will only block transfers for `primedWindow = 10` blocks. After 10 blocks, the block automatically stops and allow free market trading. 

The first one call to start this priming will receive `defaultIncentive` Malt and set `primedBlock` to start the priming. However, function `_validateSwingTraderTrigger()` which used to validate and start the priming using `livePrice` which is easy to be manipulated. Attacker can manipulate it to receive `defaultIncentive` in 2 consecutive blocks.

## Proof of Concept

Consider the scenario
1. Block i, twap is below the value returned from `maltDataLab.getSwingTraderEntryPrice()`, attacker call `stabilize()` and receive `defaultIncentive`. `primedBlock = block.number`. 
2. Block i+1, call to `_validateSwingTraderTrigger()` return `true` and trigger swing trader to bring the price back to peg. It's also reset `primedBlock = 0` (stop blocking transfer to AMM pool)
3. Since only 1 block pass, let's assume twap is still below the value returned from `maltDataLab.getSwingTraderEntryPrice()` (because twap moves slowly and will not change immediately to current price)
4. Now attacker can use flash loan to manipulate the `livePrice` to be larger than `entryPrice` (tranfer to AMM is not blocked) and call `stabilize()` to receive incentive again then repay the flash loan. 

Attacker cost is only flash loan fee, since his call will start an auction but not trigger swing trader so the state of AMM pool when he repay the flash loan is still the same (only added flash loan fee).

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L312-L334

```solidity
function _validateSwingTraderTrigger(uint256 livePrice, uint256 entryPrice)
    internal
    returns (bool)
  {
    if (usePrimedWindow) {
      if (livePrice > entryPrice) {
        return false;
      }

      if (block.number > primedBlock + primedWindow) {
        primedBlock = block.number;
        malt.mint(msg.sender, defaultIncentive * (10**malt.decimals()));
        emit MintMalt(defaultIncentive * (10**malt.decimals()));
        return false;
      }

      if (primedBlock == block.number) {
        return false;
      }
    }

    return true;
  }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider not giving incentives for caller or reset the `primedBlock` at least after `primedWindow` blocks.
