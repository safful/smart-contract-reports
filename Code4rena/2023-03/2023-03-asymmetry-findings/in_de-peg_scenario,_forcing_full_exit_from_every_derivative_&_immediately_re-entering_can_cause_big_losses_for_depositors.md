## Tags

- bug
- 2 (Med Risk)
- judge review requested
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-07

# [In de-peg scenario, forcing full exit from every derivative & immediately re-entering can cause big losses for depositors](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/765) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L142


# Vulnerability details

## Impact
In a de-peg scenario, there will be a general flight to safety (`ETH`) from users across the board. All pools will have a uni-directional sell-pressure where users prefer to exchange derivative token for WETH. 

There are 3 sources of losses in this scenario

- Protocol currently doesn't localize exit & force-exits all positions. So even non de-pegged assets are forced to exit causing further sell-side pressure that can further widen slippages

 - Protocol immediately tries to re-enter the position based on new weights. Since de-peg in one asset can trigger de-peg in another (eg. USDT de-pegged immediately after UST collapse), immediately entering into another position after exiting one might cause more. A better approach would be to simply exit stressed positions & waiting out for the market/gas prices to settle down. Separating exit & re-entry functions can save depositors from high execution costs.

 - Protocol is inefficiently exiting & re-entering the positions. Instead of taking marginal positions to rebalance, current implementation first fully exits & then re-enters back based on new weights (see POC below). Since any slippage losses are borne by depositors, a better implementation can save losses to users


## Proof of Concept

- Assume positions are split in following ratio by value: 10% frax-Eth, 70% stEth and 20% rEth 
- Now frax-Eth starts to de-peg, forcing protocol to exit frax-Eth and rebalance to say, 80% stEth and 20% rEth
- Current rebalancing first exits 70% stEth, 20% rEth and then re-enters 80% stEth and 20% rEth
- A marginal re-balancing would have only needed protocol to exit 10% frax-Eth and divert that 10% to stEth

By executing huge, unnecessary txns, protocol is exposing depositors to high slippage costs on the entire pool.

## Tools Used
Manual Review

## Recommended Mitigation Steps

Consider following improvements to `rebalanceToWeights`:

- Separate exits & entries. Split the functionality to `exit` and `re-enter`. In stressed times or fast evolving de-peg scenarios, protocol owners should first ensure an orderly exit. And then wait for markets to settle down before re-entering new positions

- Localize exits, ie. if derivative A is de-pegging, first try to exit that derivative position before incurring exit costs on derivative B and C

- Implement a marginal re-balancing - for protocol's whose weights have increased, avoid exit and re-entry. Instead just increment/decrement based on marginal changes in net positions