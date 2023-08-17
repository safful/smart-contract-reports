## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- H-02

# [User can lose up to whole stake on vault withdrawal when there are funds locked in the strategy](https://github.com/code-423n4/2023-02-ethos-findings/issues/288) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L399-L407


# Vulnerability details

ReaperVaultV2's `withdrawMaxLoss` isn't honoured when there are any locked funds in the strategy. Locked funds mean that there is a gap between requested and returned amount other than the loss reported. This is valid behavior of a strategy, but in this case realized loss is miscalculated in _withdraw() and a withdrawing user will receive less funds, while having all the shares burned.

## Impact

Users can lose up to the whole asset amount due as all their requested shares can be burned, while only available amount be transferred to them. This amount can be arbitrary low.

The behaviour is not controlled by `withdrawMaxLoss` limit and is conditional only on a strategy having some funds locked (i.e. strategy experiencing liquidity squeeze).

## Proof of Concept

_withdraw() resets `value` to be `token.balanceOf(address(this))` when the balance isn't enough for withdrawal:

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L357-L412

```solidity
    // Internal helper function to burn {_shares} of vault shares belonging to {_owner}
    // and return corresponding assets to {_receiver}. Returns the number of assets that were returned.
    function _withdraw(
        uint256 _shares,
        address _receiver,
        address _owner
    ) internal nonReentrant returns (uint256 value) {
        ...

            vaultBalance = token.balanceOf(address(this));
            if (value > vaultBalance) {
                value = vaultBalance;
            }

            require(
                totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR,
                "Withdraw loss exceeds slippage"
            );
        }

        token.safeTransfer(_receiver, value);
        emit Withdraw(msg.sender, _receiver, _owner, value, _shares);
    }
```

Each strategy can return less than `requested - loss` as some funds can be temporary frozen:

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L90-L103

```solidity
    /**
     * @dev Withdraws funds and sends them back to the vault. Can only
     *      be called by the vault. _amount must be valid and security fee
     *      is deducted up-front.
     */
    function withdraw(uint256 _amount) external override returns (uint256 loss) {
        require(msg.sender == vault, "Only vault can withdraw");
        require(_amount != 0, "Amount cannot be zero");
        require(_amount <= balanceOf(), "Ammount must be less than balance");

        uint256 amountFreed = 0;
        (amountFreed, loss) = _liquidatePosition(_amount);
        IERC20Upgradeable(want).safeTransfer(vault, amountFreed);
    }
```

The invariant there is `liquidatedAmount + loss <= _amountNeeded`, so `liquidatedAmount + loss < _amountNeeded` is a valid state (due to the funds locked):

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L230-L243

```solidity
    /**
     * Liquidate up to `_amountNeeded` of `want` of this strategy's positions,
     * irregardless of slippage. Any excess will be re-invested with `_adjustPosition()`.
     * This function should return the amount of `want` tokens made available by the
     * liquidation. If there is a difference between them, `loss` indicates whether the
     * difference is due to a realized loss, or if there is some other sitution at play
     * (e.g. locked funds) where the amount made available is less than what is needed.
     *
     * NOTE: The invariant `liquidatedAmount + loss <= _amountNeeded` should always be maintained
     */
    function _liquidatePosition(uint256 _amountNeeded)
        internal
        virtual
        returns (uint256 liquidatedAmount, uint256 loss);
```

_liquidatePosition() is called in strategy withdraw():

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L90-L103

```solidity
    /**
     * @dev Withdraws funds and sends them back to the vault. Can only
     *      be called by the vault. _amount must be valid and security fee
     *      is deducted up-front.
     */
    function withdraw(uint256 _amount) external override returns (uint256 loss) {
        require(msg.sender == vault, "Only vault can withdraw");
        require(_amount != 0, "Amount cannot be zero");
        require(_amount <= balanceOf(), "Ammount must be less than balance");

        uint256 amountFreed = 0;
        (amountFreed, loss) = _liquidatePosition(_amount);
        IERC20Upgradeable(want).safeTransfer(vault, amountFreed);
    }
```

This way there can be `lockedAmount = _amountNeeded - (liquidatedAmount + loss) >= 0`, which is neither a loss, nor withdraw-able at the moment.

As ReaperVaultV2's _withdraw() updates `value` per `if (value > vaultBalance) {value = vaultBalance;}`, the following `totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR` check do not control for the real loss and allows user to lose up to the whole amount due as _withdraw() first burns the full amount of the `_shares` requested and this total loss check for the *rebased* `value` is the only guard in place:

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L359-L412

```solidity
    function _withdraw(
        uint256 _shares,
        address _receiver,
        address _owner
    ) internal nonReentrant returns (uint256 value) {
        require(_shares != 0, "Invalid amount");
        value = (_freeFunds() * _shares) / totalSupply();
        _burn(_owner, _shares);

        if (value > token.balanceOf(address(this))) {
            ...

            vaultBalance = token.balanceOf(address(this));
            if (value > vaultBalance) {
                value = vaultBalance;
            }

            require(
                totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR,
                "Withdraw loss exceeds slippage"
            );
        }

        token.safeTransfer(_receiver, value);
        emit Withdraw(msg.sender, _receiver, _owner, value, _shares);
    }
```

Suppose there is only one strategy and `90` of the `100` tokens requested is locked at the moment, and there is no loss, just a temporal liquidity squeeze. Say there is no tokens on the vault balance before strategy withdrawal.

ReaperBaseStrategyv4's withdraw() will transfer `10`, report `0` loss, `0 = totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR = (10 + 0) * withdrawMaxLoss / PERCENT_DIVISOR` check will be satisfied for any viable `withdrawMaxLoss` setting.

Bob the withdrawing user will receive `10` tokens and have `100` tokens worth of the shares burned.

## Recommended Mitigation Steps

Consider rewriting the controlling logic so the check be based on initial value:

Now:

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L399-L407

```solidity
            vaultBalance = token.balanceOf(address(this));
            if (value > vaultBalance) {
                value = vaultBalance;
            }

            require(
                totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR,
                "Withdraw loss exceeds slippage"
            );
```

To be, as an example, if treat the loss attributed to the current user only as they have requested the withdrawal:

```solidity
            require(
                totalLoss <= (value * withdrawMaxLoss) / PERCENT_DIVISOR,
                "Withdraw loss exceeds slippage"
            );

            value -= totalLoss;

            vaultBalance = token.balanceOf(address(this));
            require(
                value <= vaultBalance,
                "Not enough funds"
            );
```

Also, `shares` can be updated according to the real value obtained as it is done in yearn:

https://github.com/yearn/yearn-vaults/blob/master/contracts/Vault.vy#L1147-L1151

```solidity
        if value > vault_balance:
            value = vault_balance
            # NOTE: Burn # of shares that corresponds to what Vault has on-hand,
            #       including the losses that were incurred above during withdrawals
            shares = self._sharesForAmount(value + totalLoss)
```