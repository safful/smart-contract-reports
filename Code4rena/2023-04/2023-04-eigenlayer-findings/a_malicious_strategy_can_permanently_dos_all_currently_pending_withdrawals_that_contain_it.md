## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-02

# [A malicious strategy can permanently DoS all currently pending withdrawals that contain it](https://github.com/code-423n4/2023-04-eigenlayer-findings/issues/132) 

# Lines of code

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L786-L789


# Vulnerability details

## Impact

A malicious strategy can permanently DoS all currently pending withdrawals that contain it. 

### Vulnerability details

In order to withdrawal funds from the project a user has to:
1. queue a withdrawal (via `queueWithdrawal`)
2. complete a withdrawal (via `completeQueuedWithdrawal(s)`)

Queuing a withdrawal, via `queueWithdrawal` modifies all internal accounting to reflect this:

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L345-L346
```Solidity
        // modify delegated shares accordingly, if applicable
        delegation.decreaseDelegatedShares(msg.sender, strategies, shares);
```

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L370
```Solidity
    if (_removeShares(msg.sender, strategyIndexes[strategyIndexIndex], strategies[i], shares[i])) {
```

and saves the withdrawl hash in order to be used in `completeQueuedWithdrawal(s)`

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L400-L415
```Solidity
            queuedWithdrawal = QueuedWithdrawal({
                strategies: strategies,
                shares: shares,
                depositor: msg.sender,
                withdrawerAndNonce: withdrawerAndNonce,
                withdrawalStartBlock: uint32(block.number),
                delegatedAddress: delegatedAddress
            });


        }


        // calculate the withdrawal root
        bytes32 withdrawalRoot = calculateWithdrawalRoot(queuedWithdrawal);


        // mark withdrawal as pending
        withdrawalRootPending[withdrawalRoot] = true;
```

In other words, it is final (as there is no `cancelWithdrawl` mechanism implemented).

When executing`completeQueuedWithdrawal(s)`, the `withdraw` function of the strategy is called (if `receiveAsTokens` is set to true).

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L786-L789
```Solidity
    // tell the strategy to send the appropriate amount of funds to the depositor
    queuedWithdrawal.strategies[i].withdraw(
        msg.sender, tokens[i], queuedWithdrawal.shares[i]
    );
```

In this case, a malicious strategy can always revert, blocking the user from retrieving his tokens.

If a user sets `receiveAsTokens` to `false`, the other case, then the tokens will be added as shares to the delegation contract

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L797-L800
```Solidity
    for (uint256 i = 0; i < strategiesLength;) {
        _addShares(msg.sender, queuedWithdrawal.strategies[i], queuedWithdrawal.shares[i]);
        unchecked {
            ++i;
```

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L646-L647
```Solidity
        // if applicable, increase delegated shares accordingly
        delegation.increaseDelegatedShares(depositor, strategy, shares);
```

This still poses a problem because the `increaseDelegatedShares` function's counterpart, `decreaseDelegatedShares` is only callable by the strategy manager (so as for users to indirectly get theier rewards worth back via this workaround)

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/DelegationManager.sol#L168-L179
```Solidity
    /**
     * @notice Decreases the `staker`'s delegated shares in each entry of `strategies` by its respective `shares[i]`, typically called when the staker withdraws from EigenLayer
     * @dev Callable only by the StrategyManager
     */
    function decreaseDelegatedShares(
        address staker,
        IStrategy[] calldata strategies,
        uint256[] calldata shares
    )
        external
        onlyStrategyManager
    {
```

but in `StrategyManager` there are only 3 cases where this is called, none of which is beneficial for the user:
 - [`recordOvercommittedBeaconChainETH`](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L205) - slashes user rewards in certain conditions
 - [`queueWithdrawal`](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L346) - accounting side effects have already been done, will fail in `_removeShares`
 - [`slashShares`](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L346) - slashes user rewards in certain conditions


In other words, the workaround (of setting `receiveAsTokens` to `false` in `completeQueuedWithdrawal(s)`) just leaves the funds accounted and stuck in `DelegationManager`.

In both cases all shares/tokens associated with other strategies in the pending withdrawal are permanently blocked.

## Proof of Concept

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L786-L789

## Tools Used

Manual analysis

## Recommended Mitigation Steps

Add a `cancelQueuedWithdrawl` function that will undo the accounting and cancel out any dependency on the malicious strategy. Although this solution may create a front-running opportunity for when their withdrawal will be slashed via `slashQueuedWithdrawal`, there may exist workarounds to this.

Another possibility is to implement a similar mechanism to how `slashQueuedWithdrawal` treats malicious strategies: adding a list of strategy indices to skip

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L532-L535
https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L560-L566
```Solidity
     * @param indicesToSkip Optional input parameter -- indices in the `strategies` array to skip (i.e. not call the 'withdraw' function on). This input exists
     * so that, e.g., if the slashed QueuedWithdrawal contains a malicious strategy in the `strategies` array which always reverts on calls to its 'withdraw' function,
     * then the malicious strategy can be skipped (with the shares in effect "burned"), while the non-malicious strategies are still called as normal.
     */
     
     ...

        for (uint256 i = 0; i < strategiesLength;) {
            // check if the index i matches one of the indices specified in the `indicesToSkip` array
            if (indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i) {
                unchecked {
                    ++indicesToSkipIndex;
                }
            } else {     
```


## Assessed type

DoS