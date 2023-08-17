## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-02

# [_harvestCore() roi calculation error](https://github.com/code-423n4/2023-02-ethos-findings/issues/752) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperStrategyGranarySupplyOnly.sol#L141


# Vulnerability details

## Impact
_harvestCore() roi calculation error,may double

## Proof of Concept
The _harvestCore() will calculate the roi and repayment values
The implementation code is as follows:
```solidity
    function _harvestCore(uint256 _debt) internal override returns (int256 roi, uint256 repayment) {
        _claimRewards();
        uint256 numSteps = steps.length;
        for (uint256 i = 0; i < numSteps; i = i.uncheckedInc()) {
            address[2] storage step = steps[i];
            IERC20Upgradeable startToken = IERC20Upgradeable(step[0]);
            uint256 amount = startToken.balanceOf(address(this));
            if (amount == 0) {
                continue;
            }
            _swapVelo(step[0], step[1], amount, VELO_ROUTER);
        }

        uint256 allocated = IVault(vault).strategies(address(this)).allocated;
        uint256 totalAssets = balanceOf();
        uint256 toFree = _debt;

        if (totalAssets > allocated) {
            uint256 profit = totalAssets - allocated;
            toFree += profit;
            roi = int256(profit);
        } else if (totalAssets < allocated) {
            roi = -int256(allocated - totalAssets);
        }

        (uint256 amountFreed, uint256 loss) = _liquidatePosition(toFree);
        repayment = MathUpgradeable.min(_debt, amountFreed);
        roi -= int256(loss);//<------这个地方可能会导致重复
    }
```
The last line may cause double counting of losses
For example, the current:
vault.allocated = 9
vault.strategy.allocBPS = 9000
strategy.totalAssets = 9

Suppose that after some time, strategy loses 2, then:
strategy.totalAssets = 9 - 2 = 7
Also the administrator sets vault.strategy.allocBPS = 0

This executes harvest()->_harvestCore(9) to get
roi = 4
repayment = 7

The actual loss of 2, but roi = 4 (double), test code as follows:

add to test/starter-test.js 'Vault Tests'

```solidity
    it.only('test_roi', async function () {
      const {vault, strategy, wantHolder, strategist} = await loadFixture(deployVaultAndStrategyAndGetSigners);
      const depositAmount = toWantUnit('10');
      await vault.connect(wantHolder)['deposit(uint256)'](depositAmount);
      await strategy.harvest();

      const balanceOf = await strategy.balanceOf();
      console.log(`strategy balanceOf: ${balanceOf}`);
      // allocated = 9
      // 1.loss 2, left 7
      await strategy.lossFortest(toWantUnit('2'));
      // 2.modify bps=>0
      await vault.connect(strategist).updateStrategyAllocBPS(strategy.address, 0);
      // 3.so debt = 9
      await strategy.harvest();

      const {allocated, losses, allocBPS} = await vault.strategies(strategy.address);
      console.log(`losses: ${losses}`);
      console.log(`allocated: ${allocated}`);
      console.log(`allocBPS: ${allocBPS}`);
    });

```
add to ReaperStrategyGranarySupplyOnly.sol
```solidity
    function lossFortest(uint256 amout) external{
        ILendingPool(ADDRESSES_PROVIDER.getLendingPool()).withdraw(address(want), amout, address(1));        
    }
```

```console
$ npx hardhat test test/starter-test.js

  Vaults
    Vault Tests
strategy balanceOf: 900000000
losses: 400000000     <--------will double
allocated: 0
allocBPS: 0
```

The last vault's allocated is correct, but the loss is wrong
Statistics and bpsChange of _reportLoss() will be wrong



## Tools Used

## Recommended Mitigation Steps

remove ` roi -= int256(loss);`