## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-14

# [User underpay for the remote call execution gas on root chain](https://github.com/code-423n4/2023-05-maia-findings/issues/612) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L836
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1066
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1032
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L811


# Vulnerability details

## Impact
User underpay for the remote call execution gas, meaning Incorrect **minExecCost** that being deposited at `_replenishGas` call inside `_payExecutionGas` function.

## Proof of Concept
*Multi chain contracts - anycall v7 lines*
https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Upgradeable.sol#L265

https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Upgradeable.sol#L167

https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Upgradeable.sol#L276

*ulysses-omnichain contract lines*
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L811

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L851

The user is paying incorrect minimum execution cost for Anycall Mutlichain [L820](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L820), the value of `minExecCost` is calculated incorrectly. AnycallV7 protocol considers a premium fee `_feeData.premium` on top of the TX gas price which is not considered here.

let's get into the flow from the start, the `anyExec` call that being called by the executer [L265](https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Upgradeable.sol#L265) when an anycall request comes from a source chain includes `chargeDestFee` modifier
```Solidity
    function anyExec(
        address _to,
        bytes calldata _data,
        string calldata _appID,
        RequestContext calldata _ctx,
        bytes calldata _extdata
    )
        external
        virtual
        lock
        whenNotPaused
        chargeDestFee(_to, _ctx.flags)
        onlyMPC
    {
        IAnycallConfig(config).checkExec(_appID, _ctx.from, _to);
```

now, chargeDestFee modifier will call `chargeFeeOnDestChain` function as well at  [L167](https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Upgradeable.sol#L167) 
```Solidity
/// @dev Charge an account for execution costs on this chain
/// @param _from The account to charge for execution costs
    modifier chargeDestFee(address _from, uint256 _flags) {
        if (_isSet(_flags, AnycallFlags.FLAG_PAY_FEE_ON_DEST)) {
            uint256 _prevGasLeft = gasleft();
            _;
            IAnycallConfig(config).chargeFeeOnDestChain(_from, _prevGasLeft);
        } else {
            _;
        }
    }
```

as you see [L198-L210](https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Config.sol#L198C1-L210), inside chargeFeeOnDestChain function is including`_feeData.premium` for the execution cost `totalCost`.
```Solidity
function chargeFeeOnDestChain(address _from, uint256 _prevGasLeft)
        external
        onlyAnycallContract
    {
        if (!_isSet(mode, FREE_MODE)) {
            uint256 gasUsed = _prevGasLeft + EXECUTION_OVERHEAD - gasleft();
            uint256 totalCost = gasUsed * (tx.gasprice + _feeData.premium);
            uint256 budget = executionBudget[_from];
            require(budget > totalCost, "no enough budget");
            executionBudget[_from] = budget - totalCost;
            _feeData.accruedFees += uint128(totalCost);
        }
    }
```


The conclusion: `minExecCost`  calculation doesn't include `_feeData.premium` at [L811](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L811) according to multichain AnycallV7 protocol.

You should include `_feeData.premium` as well in `minExecCost`  same as [L204](https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Config.sol#L204)

```
uint256 totalCost = gasUsed * (tx.gasprice + _feeData.premium);
```


Note: This also applicable on:
_payFallbackGas() in RootBridgeAgent at [L836](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L836) 
_payFallbackGas() in BranchBridgeAgent at [L1066](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1066)
_payExecutionGas in BranchBridgeAgent at [L1032](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1032)



## Tools Used
Manual Review

## Recommended Mitigation Steps
add `_feeData.premium` to `minExecCost`  at  `_payExecutionGas` function [L811](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L811)

you need to get _feeData.premium first from AnycallV7Config by premium() function at [L286-L288](https://github.com/anyswap/multichain-smart-contracts/blob/645d0053d22ed63005b9414b5610879094932304/contracts/anycall/v7/AnycallV7Config.sol#L286-L288)
```
uint256 minExecCost = (tx.gasprice  + _feeData.premium) * (MIN_EXECUTION_OVERHEAD + _initialGas - gasleft()));

```



## Assessed type

Other