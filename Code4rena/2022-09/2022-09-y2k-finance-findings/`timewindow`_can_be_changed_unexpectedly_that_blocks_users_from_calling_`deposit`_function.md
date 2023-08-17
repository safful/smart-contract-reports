## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [`timewindow` can be changed unexpectedly that blocks users from calling `deposit` function](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/483) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Vault.sol#L87-L91
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Vault.sol#L152-L174
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/VaultFactory.sol#L327-L338
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Vault.sol#L287-L289


# Vulnerability details

## Impact
As shown by the following `epochHasNotStarted` modifier, which is used by the `deposit` function below, users can only deposit when `block.timestamp <= idEpochBegin[id] - timewindow` holds true. Before depositing, a user can check if this relationship is true at that moment; if so, she or he can call the `deposit` function. However, just before the user's `deposit` function call is executed, the admin unexpectedly calls the `VaultFactory.changeTimewindow` function below, which further calls the `Vault.changeTimewindow` function below, to increase the `timewindow`. Since the admin's `VaultFactory.changeTimewindow` transaction is executed before the user's `deposit` transaction and the `timewindow` change takes effect immediately, it is possible that the user's `deposit` function call will revert. Besides wasting gas, the user can feel confused and unfair because her or his `deposit` transaction should be executed successfully if `VaultFactory.changeTimewindow` is not called unexpectedly. 

https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Vault.sol#L87-L91
```solidity
    modifier epochHasNotStarted(uint256 id) {
        if(block.timestamp > idEpochBegin[id] - timewindow)
            revert EpochAlreadyStarted();
        _;
    }
```

https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Vault.sol#L152-L174
```solidity
    function deposit(
        uint256 id,
        uint256 assets,
        address receiver
    )
        public
        override
        marketExists(id)
        epochHasNotStarted(id)
        nonReentrant
        returns (uint256 shares)
    {
        // Check for rounding error since we round down in previewDeposit.
        require((shares = previewDeposit(id, assets)) != 0, "ZeroValue");

        asset.transferFrom(msg.sender, address(this), shares);

        _mint(receiver, id, shares, EMPTY);

        emit Deposit(msg.sender, receiver, id, shares, shares);

        return shares;
    }
```

https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/VaultFactory.sol#L327-L338
```solidity
    function changeTimewindow(uint256 _marketIndex, uint256 _timewindow)
        public
        onlyAdmin
    {
        address[] memory vaults = indexVaults[_marketIndex];
        Vault insr = Vault(vaults[0]);
        Vault risk = Vault(vaults[1]);
        insr.changeTimewindow(_timewindow);
        risk.changeTimewindow(_timewindow);

        emit changedTimeWindow(_marketIndex, _timewindow);
    }
```

https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Vault.sol#L287-L289
```solidity
    function changeTimewindow(uint256 _timewindow) public onlyFactory {
        timewindow = _timewindow;
    }
```

## Proof of Concept
Please add the following `error` and append the following test in `test\AssertTest.t.sol`. This test will pass to demonstrate the described scenario.

```solidity
    error EpochAlreadyStarted();

    function testChangeTimeWindowUnexpectedly() public {
        vm.deal(alice, AMOUNT);
        vm.deal(chad, AMOUNT * CHAD_MULTIPLIER);

        vm.startPrank(admin);
        FakeOracle fakeOracle = new FakeOracle(oracleFRAX, STRIKE_PRICE_FAKE_ORACLE);
        vaultFactory.createNewMarket(FEE, tokenFRAX, DEPEG_AAA, beginEpoch, endEpoch, address(fakeOracle), "y2kFRAX_99*");
        vm.stopPrank();

        address hedge = vaultFactory.getVaults(1)[0];
        address risk = vaultFactory.getVaults(1)[1];
        
        Vault vHedge = Vault(hedge);
        Vault vRisk = Vault(risk);

        // alice is able to deposit in hedge vault before the time window change
        vm.startPrank(alice);
        ERC20(WETH).approve(hedge, AMOUNT);
        vHedge.depositETH{value: AMOUNT}(endEpoch, alice);
        vm.stopPrank();

        // admin changes time window unexpectedly, which takes effect immediately
        vm.startPrank(admin);
        vaultFactory.changeTimewindow(1, 5 days);
        vm.stopPrank();

        // chad is unable to deposit in risk vault after the time window change
        vm.startPrank(chad);
        ERC20(WETH).approve(risk, AMOUNT * CHAD_MULTIPLIER);

        vm.expectRevert(EpochAlreadyStarted.selector);
        vRisk.depositETH{value: AMOUNT * CHAD_MULTIPLIER}(endEpoch, chad);
        vm.stopPrank();
    }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
When calling the `VaultFactory.createNewMarket` or `VaultFactory.deployMoreAssets` function, the `timewindow`, which is configured for that moment, can be taken into account in the created asset's `epochBegin`.

Then, https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Vault.sol#L87-L91 can be updated to the following code.
```solidity
    modifier epochHasNotStarted(uint256 id) {
        if(block.timestamp > idEpochBegin[id])
            revert EpochAlreadyStarted();
        _;
    }
```