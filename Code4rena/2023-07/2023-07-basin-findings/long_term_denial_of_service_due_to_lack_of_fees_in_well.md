## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- sponsor disputed
- M-04

# [Long term denial of service due to lack of fees in Well](https://github.com/code-423n4/2023-07-basin-findings/issues/255) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/Well.sol#L190


# Vulnerability details

## Description

The Well allows users to permissionless swap assets or add and remove liquidity. Users specify the intended slippage in `swapFrom`, in `minAmountOut`.

The ConstantProduct2 implementation ensures `Kend - Kstart >= 0`, where `K = Reserve1 * Reserve2`, and the delta should only be due to tiny precision errors.

Furthermore, the Well does not impose any fees to its users. This means that all conditions hold for a successful DOS of any swap transactions.
1. Token cost of sandwiching swaps is zero (no fees) - only gas cost
2. Price updates are instantenous through the billion dollar formula.
3. Swap transactions along with the max slippage can be viewed in the mempool

Note that such DOS attacks have serious adverse effects both on the protocol and the users. Protocol will use users due to disfunctional interactions. On the other side, users may opt to increment the max slippage in order for the TX to go through, which can be directly abused by the same MEV bots that could be performing the DOS.

## Impact

All swaps can be reverted at very little cost.

## POC

1. Evil bot sees swap TX, slippage=S
2. Bot submits a flashbot bundle, with the following TXs
	1. Swap TX in the same direction as victim, to bump slippage above S
	2. Victim TX, which will revert
	3. Swap TX in the opposite direction and velocity to TX (1). Because of the constant product formula, all tokens will be restored to the attacker.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Fees solve the problem described by making it too costly for attackers to DOS swaps. If DOS does takes place, liquidity providers are profiting a high APY to offset the inconvenience caused, and attract greater liquidity.


## Assessed type

DoS