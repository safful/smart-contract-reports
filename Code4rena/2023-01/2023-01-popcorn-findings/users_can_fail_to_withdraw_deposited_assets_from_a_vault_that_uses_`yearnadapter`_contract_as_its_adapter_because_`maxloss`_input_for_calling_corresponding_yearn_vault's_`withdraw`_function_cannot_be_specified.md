## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-07

# [Users can fail to withdraw deposited assets from a vault that uses `YearnAdapter` contract as its adapter because `maxLoss` input for calling corresponding Yearn vault's `withdraw` function cannot be specified](https://github.com/code-423n4/2023-01-popcorn-findings/issues/581) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L210-L235
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/yearn/YearnAdapter.sol#L166-L172
https://github.com/yearn/yearn-vaults/blob/master/contracts/Vault.vy#L1028-L1167


# Vulnerability details

## Impact
For a vault that uses the `YearnAdapter` contract as its adapter, calling the `Vault.withdraw` or `Vault.redeem` function will eventually call the `AdapterBase._withdraw` and `YearnAdapter._protocolWithdraw` functions below when the adapter is not paused. When the `YearnAdapter._protocolWithdraw` function executes `yVault.withdraw(convertToUnderlyingShares(assets, shares))`, the `maxLoss` input is not specified when calling the Yearn vault's `withdraw` function below. Thus, the Yearn vault's `withdraw` function will be called with its default `maxLoss` input value that is 0.01%. If the total loss incurred during the withdrawal is more than 0.01%, calling the Yearn vault's `withdraw` function that executes `assert totalLoss <= maxLoss * (value + totalLoss) / MAX_BPS` will revert. In a bear market, it is possible that the Yearn vault's strategies do not perform well so the total loss can be more than 0.01% permanently. In this situation, calling the `Vault.withdraw` or `Vault.redeem` function will always revert because calling the Yearn vault's `withdraw` function without specifying the `maxLoss` input reverts. As a result, users lose the deposited assets that they are unable to withdraw.

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L210-L235
```solidity
    function _withdraw(
        address caller,
        address receiver,
        address owner,
        uint256 assets,
        uint256 shares
    ) internal virtual override {
        ...

        if (!paused()) {
            uint256 underlyingBalance_ = _underlyingBalance();  
            _protocolWithdraw(assets, shares);
            // Update the underlying balance to prevent inflation attacks
            underlyingBalance -= underlyingBalance_ - _underlyingBalance();
        }

        ...
    }
```

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/yearn/YearnAdapter.sol#L166-L172
```solidity
    function _protocolWithdraw(uint256 assets, uint256 shares)
        internal
        virtual
        override
    {
        yVault.withdraw(convertToUnderlyingShares(assets, shares));
    }
```

https://github.com/yearn/yearn-vaults/blob/master/contracts/Vault.vy#L1028-L1167
```vyper
@external
@nonreentrant("withdraw")
def withdraw(
    maxShares: uint256 = MAX_UINT256,
    recipient: address = msg.sender,
    maxLoss: uint256 = 1,  # 0.01% [BPS]
) -> uint256:
    """
    ...
    @param maxLoss
        The maximum acceptable loss to sustain on withdrawal. Defaults to 0.01%.
        If a loss is specified, up to that amount of shares may be burnt to cover losses on withdrawal.
    @return The quantity of tokens redeemed for `_shares`.
    """
    shares: uint256 = maxShares  # May reduce this number below

    # Max Loss is <=100%, revert otherwise
    assert maxLoss <= MAX_BPS

    # If _shares not specified, transfer full share balance
    if shares == MAX_UINT256:
        shares = self.balanceOf[msg.sender]

    # Limit to only the shares they own
    assert shares <= self.balanceOf[msg.sender]

    # Ensure we are withdrawing something
    assert shares > 0

    # See @dev note, above.
    value: uint256 = self._shareValue(shares)
    vault_balance: uint256 = self.totalIdle

    if value > vault_balance:
        totalLoss: uint256 = 0
        # We need to go get some from our strategies in the withdrawal queue
        # NOTE: This performs forced withdrawals from each Strategy. During
        #       forced withdrawal, a Strategy may realize a loss. That loss
        #       is reported back to the Vault, and the will affect the amount
        #       of tokens that the withdrawer receives for their shares. They
        #       can optionally specify the maximum acceptable loss (in BPS)
        #       to prevent excessive losses on their withdrawals (which may
        #       happen in certain edge cases where Strategies realize a loss)
        for strategy in self.withdrawalQueue:
            if strategy == ZERO_ADDRESS:
                break  # We've exhausted the queue

            if value <= vault_balance:
                break  # We're done withdrawing

            amountNeeded: uint256 = value - vault_balance

            # NOTE: Don't withdraw more than the debt so that Strategy can still
            #       continue to work based on the profits it has
            # NOTE: This means that user will lose out on any profits that each
            #       Strategy in the queue would return on next harvest, benefiting others
            amountNeeded = min(amountNeeded, self.strategies[strategy].totalDebt)
            if amountNeeded == 0:
                continue  # Nothing to withdraw from this Strategy, try the next one

            # Force withdraw amount from each Strategy in the order set by governance
            preBalance: uint256 = self.token.balanceOf(self)
            loss: uint256 = Strategy(strategy).withdraw(amountNeeded)
            withdrawn: uint256 = self.token.balanceOf(self) - preBalance
            vault_balance += withdrawn

            # NOTE: Withdrawer incurs any losses from liquidation
            if loss > 0:
                value -= loss
                totalLoss += loss
                self._reportLoss(strategy, loss)

            # Reduce the Strategy's debt by the amount withdrawn ("realized returns")
            # NOTE: This doesn't add to returns as it's not earned by "normal means"
            self.strategies[strategy].totalDebt -= withdrawn
            self.totalDebt -= withdrawn
            log WithdrawFromStrategy(strategy, self.strategies[strategy].totalDebt, loss)

        self.totalIdle = vault_balance
        # NOTE: We have withdrawn everything possible out of the withdrawal queue
        #       but we still don't have enough to fully pay them back, so adjust
        #       to the total amount we've freed up through forced withdrawals
        if value > vault_balance:
            value = vault_balance
            # NOTE: Burn # of shares that corresponds to what Vault has on-hand,
            #       including the losses that were incurred above during withdrawals
            shares = self._sharesForAmount(value + totalLoss)

        # NOTE: This loss protection is put in place to revert if losses from
        #       withdrawing are more than what is considered acceptable.
        assert totalLoss <= maxLoss * (value + totalLoss) / MAX_BPS

    # Burn shares (full value of what is being withdrawn)
    self.totalSupply -= shares
    self.balanceOf[msg.sender] -= shares
    log Transfer(msg.sender, ZERO_ADDRESS, shares)
    
    self.totalIdle -= value
    # Withdraw remaining balance to _recipient (may be different to msg.sender) (minus fee)
    self.erc20_safe_transfer(self.token.address, recipient, value)
    log Withdraw(recipient, shares, value)
    
    return value
```

## Proof of Concept
The following steps can occur for the described scenario.
1. A vault that uses the `YearnAdapter` contract as its adapter exists.
2. A user calls the `Vault.deposit` function to deposit some asset tokens in the corresponding Yearn vault.
3. A bear market starts so the Yearn vault's strategies do not perform well, and the total loss is more than 0.01% consistently.
4. Calling the `Vault.withdraw` or `Vault.redeem` function always reverts because the user cannot specify the `maxLoss` input for calling the Yearn vault's `withdraw` function. As a result, the user loses the deposited asset tokens.

## Tools Used
VSCode

## Recommended Mitigation Steps
The `YearnAdapter._protocolWithdraw` function can be updated to add an additional input that would be used as the `maxLoss` input for calling the Yearn vault's `withdraw` function. The other functions that call the `YearnAdapter._protocolWithdraw` function need to add this input as well.