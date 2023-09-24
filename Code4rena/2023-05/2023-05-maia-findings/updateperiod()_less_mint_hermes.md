## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-08

# [updatePeriod() less mint HERMES](https://github.com/code-423n4/2023-05-maia-findings/issues/737) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/hermes/minters/BaseV2Minter.sol#L139-L141


# Vulnerability details

## Impact
If there is a `weekly` that has not been taken, it may result in insufficient mint HERMES

## Proof of Concept
In `updatePeriod()`, mint new `HERMES` every week with a certain percentage `weeklyEmission`.


The code is as follows.

```solidity
    function updatePeriod() public returns (uint256) {
        uint256 _period = activePeriod;
        // only trigger if new week
        if (block.timestamp >= _period + week && initializer == address(0)) {
            _period = (block.timestamp / week) * week;
            activePeriod = _period;
            uint256 newWeeklyEmission = weeklyEmission();
@>          weekly += newWeeklyEmission;
            uint256 _circulatingSupply = circulatingSupply();

            uint256 _growth = calculateGrowth(newWeeklyEmission);
            uint256 _required = _growth + newWeeklyEmission;
            /// @dev share of newWeeklyEmission emissions sent to DAO.
            uint256 share = (_required * daoShare) / base;
            _required += share;
            uint256 _balanceOf = underlying.balanceOf(address(this));          
@>          if (_balanceOf < _required) {
                HERMES(underlying).mint(address(this), _required - _balanceOf);
            }

            underlying.safeTransfer(address(vault), _growth);

            if (dao != address(0)) underlying.safeTransfer(dao, share);

            emit Mint(msg.sender, newWeeklyEmission, _circulatingSupply, _growth, share);

            /// @dev queue rewards for the cycle, anyone can call if fails
            ///      queueRewardsForCycle will call this function but won't enter
            ///      here because activePeriod was updated
            try flywheelGaugeRewards.queueRewardsForCycle() {} catch {}
        }
        return _period;
    }
```

The above code will first determine if the balance of the current contract is less than `_required` and if it is less, then mint new `HERMES`, so that there will be enough `HERMES` for the distribution.

But there is a problem that the current balance of contract may contain the last `weekly` HERMES,that  `flywheelGaugeRewards` have not yet been taken:  (e.g. last week's allocation of weeklyEmission)

Because the `gaugeCycle` of `flywheelGaugeRewards` may be greater than one week. So it is possible that the last `weekly`  HERMES has not yet been taken.

So we can't use the current balance to compare with `_required` directly, we need to Consider  the `weekly` staying in the contract, it hasn't been taken, to avoid not having enough balance when `flywheelGaugeRewards` comes to take `weekly`.


## Tools Used

## Recommended Mitigation Steps

```solidity
    function updatePeriod() public returns (uint256) {
        uint256 _period = activePeriod;
        // only trigger if new week
        if (block.timestamp >= _period + week && initializer == address(0)) {
            _period = (block.timestamp / week) * week;
            activePeriod = _period;
            uint256 newWeeklyEmission = weeklyEmission();
            weekly += newWeeklyEmission;
            uint256 _circulatingSupply = circulatingSupply();

            uint256 _growth = calculateGrowth(newWeeklyEmission);
            uint256 _required = _growth + newWeeklyEmission;
            /// @dev share of newWeeklyEmission emissions sent to DAO.
            uint256 share = (_required * daoShare) / base;
            _required += share;
            uint256 _balanceOf = underlying.balanceOf(address(this));          
-           if (_balanceOf < _required) {
+           if (_balanceOf < weekly + _growth + share ) {
-              HERMES(underlying).mint(address(this), _required - _balanceOf);
+              HERMES(underlying).mint(address(this),weekly + _growth + share - _balanceOf);
            }

            underlying.safeTransfer(address(vault), _growth);

            if (dao != address(0)) underlying.safeTransfer(dao, share);

            emit Mint(msg.sender, newWeeklyEmission, _circulatingSupply, _growth, share);

            /// @dev queue rewards for the cycle, anyone can call if fails
            ///      queueRewardsForCycle will call this function but won't enter
            ///      here because activePeriod was updated
            try flywheelGaugeRewards.queueRewardsForCycle() {} catch {}
        }
        return _period;
    }
```


## Assessed type

Context