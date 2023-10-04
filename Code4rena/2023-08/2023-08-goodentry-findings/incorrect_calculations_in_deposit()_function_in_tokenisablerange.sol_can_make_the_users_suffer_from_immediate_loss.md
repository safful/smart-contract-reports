## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- satisfactory
- selected for report
- sponsor acknowledged
- M-07

# [Incorrect calculations in deposit() function in TokenisableRange.sol can make the users suffer from immediate loss](https://github.com/code-423n4/2023-08-goodentry-findings/issues/202) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/TokenisableRange.sol#L218-L285


# Vulnerability details

## Impact
Calculations of uncompounded fee in deposit() in TokenisableRange.sol is incorrect, this can make a potential immediate fund loss after a user make a deposit.

## Proof of Concept
Generally speaking, functions in a protocol should be designed symmetrically. For example in a DEX, when swap X to Y and then immediately swap Y to X, when excluding fees, the user won't loss any funds. In this TokenisableRange.sol case, when a user deposits some funds, if the user withdraw it immediately, and when there is no market condition change and exclude fees, the amount the user can obtained should be same as the deposit value. 

However, this is not the case in deposit() function. In deposit(), a user will input the amount he/she want to deposit, a uncompound fee may subtracted from that amount, and the rest of the amount will be deposited, and LP tokens will be transferred to the user. When withdraw, the user burn the LP token, and obtain the corresponding liquidity and uncompounded fee. As stated above, when no market change and exclude any fees, the liquidity and uncompounded fee a user can get should be the same as paid in deposit(). The issue is in these lines:

    if (fee0 + fee1 > 0 && (n0 > 0 || fee0 == 0) && (n1 > 0 || fee1 == 0)) {
            address pool = V3_FACTORY.getPool(
                address(TOKEN0.token),
                address(TOKEN1.token),
                feeTier * 100
            );
            (uint160 sqrtPriceX96, , , , , , ) = IUniswapV3Pool(pool).slot0();
            (uint256 token0Amount, uint256 token1Amount) = LiquidityAmounts
                .getAmountsForLiquidity(
                    sqrtPriceX96,
                    TickMath.getSqrtRatioAtTick(lowerTick),
                    TickMath.getSqrtRatioAtTick(upperTick),
                    liquidity
                );
            if (token0Amount + fee0 > 0)
                newFee0 = (n0 * fee0) / (token0Amount + fee0);
            if (token1Amount + fee1 > 0)
                newFee1 = (n1 * fee1) / (token1Amount + fee1);
            fee0 += newFee0;
            fee1 += newFee1;
            n0 -= newFee0;
            n1 -= newFee1;
        }

The issue here is, the deducted uncompounded fee newFee0 and newFee1 are computed based on users input n0 and n1, but the user input n0 and n1 may not be as the same ratio correspond to the current market ratio. So in conclusion, the uncompounded fees is computed based on user input n0 and n1, the ratio between that n0 and n1 may not be the current ratio under current market condition, but later the actual deposit amounts are based on the current ratio and the minted LP token also based on that ratio, the current ratio is also used in withdraw, so it is this difference that result in a potential loss in uncompounded fee.

This may not obvious so let's look at an example with solid numbers.

We assume such a condition. User input n0 = 415, n1 = 100, uncompounded fee0 f0 = 20, fee1 f1 = 5, under current market price and tick range token0Amount t0 = 4000, token1Amount t1 = 1000, t0 and t1 correspond to a liquidity L of 1000, and LP token total supply T is 2000.

Since f0 + f1 = 25 > 0 && n0 = 415 > 0 && n1 = 100 > 0, we will enter the first if statement to compute the deducted uncompounded fee. newFee0 = 415\*20/(4000+20) = 415/201, newFee1 = 100\*5/(1000+5) = 100/201, updated n0 = 415 - 415/201 = 412.9353234, updated n1 = 100 - 100/201 = 20000/201. 

We will then make the deposit. Under current assumed market condition we can deposit n1 of 20000/201 and n0 of 80000/201 and obtain a new liquidity newL of 20000/201. The LP token amount we can get is newL/L\*T = 40000/201. Note here n1 will all be deposited, deposited n0 is 80000/201 , which is larger than 95% of (n1 - newFee0), which is 95% of 412.9353234. So slippage check is satisfied.

In conclusion, in this deposit(), user specified n0 of 415 and n1 of 100, user have an actual deposit of n0 = 80000/201 and n1 of 20000/201, this corresponding to a liquidity of 20000/201, user also deposit a uncompounded fee, newFee0 is 415/201 and newFee1 is 100/201. User get back 40000/201 LP token. Now updated fee0 = 20 + 415/201 = 1475/67, and updated fee1 = 5 + 100/201 = 1105/201. Updated total liquidity is 1000 + 20000/201 = 1099.502488, and updated LP token supply is 442000/201.

Now user wants to withdraw. He will burn all his LP token, so the removedLiquidity he can get is (40000/201)/(442000/201)\*1099.502488 = 20000/201, this is same as the liquidity got in deposit(). For uncompounded fee, obtained fee0 = (40000/201)/(442000/201)\*1475/67 = 1.9923, obtained fee1 = (40000/201)/(442000/201)\*1105/201 = 100/201. We can see the fee1 got back is exactly the same, but the fee0 different, deposited 415/201 = 2.0647 but only get back 1.9923, so the user will incur a loss. 

As mentioned before, the loss in uncompounded fee here is due to use user input n0 and n1 here, which may not be proportional to the actual deposit amount.

## Tools Used
VS Code, Manual Review

## Recommended Mitigation Steps
The protocol should first calculate a proportioned n0 and n1 based on user inputs, then compute uncompounded fee based on that. Take the example above, user inputs n0 = 415 and n1 = 100, the protocol should calculate that the proportioned n0 = 400 and n1 = 100 under current condition. Then the fee should be calculated based on the n0 of 400 and n1 of 100. 


## Assessed type

Other