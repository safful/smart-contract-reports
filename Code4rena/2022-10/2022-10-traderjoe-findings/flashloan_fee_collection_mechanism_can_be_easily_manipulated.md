## Tags

- bug
- 2 (Med Risk)
- primary issue
- sponsor acknowledged
- edited-by-warden
- selected for report
- M-03

# [Flashloan fee collection mechanism can be easily manipulated](https://github.com/code-423n4/2022-10-traderjoe-findings/issues/136) 

# Lines of code

https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L452-L453


# Vulnerability details

## Impact

[`LBPair.flashLoan()`](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L415-L456) utilizes an “unfair” fee mechanism in which the whole pair liquidity can be loaned but only the liquidity providers of the active bin receive the fees. Although one can argue that this unfair structure is to incentivize greater liquidity around the active price range, it nonetheless opens up a way to easily manipulate fees. The current structure allows a user to provide liquidity to an active bin right before a flashloan to receive most of the fees. This trick can be used both by the borrower themselves, or by a third party miner or a node operator frontrunning the flashloan transactions. In either case, this is in detriment to the liquidity providers, who would be providing the bulk of the flashloan, but receiving a much less fraction of the fees.

## Proof of Concept

`LBPair.flashLoan()` function [enables borrowing the entire balance of a pair](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L435-L436).

```solidity
        tokenX.safeTransfer(_to, _amountXOut);
        tokenY.safeTransfer(_to, _amountYOut);
```

This means that a liquidity provider’s tokens can be used regardless of which bin their liquidity is in. However, the loan fee [is only paid to the active bin’s liquidity providers](https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L452-L453).

```solidity
        _bins[_id].accTokenXPerShare += _feesX.getTokenPerShare(_totalSupply);
        _bins[_id].accTokenYPerShare += _feesY.getTokenPerShare(_totalSupply);
```

Based on this, someone can frontrun a flashloan by adding liquidity to the active bin to receive the most of the flashloan fees, even if the active bin constitutes a small percentage of the loaned amount. Alternatively, a borrower can also atomically add and remove liquidity to the active bin before and after the flashloan, respectively. This way the borrower essentially would be using flashloans with fraction of the intended fee.

### Example Scenario

Imagine a highly volatile market, in which the liquidity providers lag behind the price action, resulting in active bin to constitute a very small percent of the liquidity.  The following snippet shows the liquidity in each bin of the eleven bins of this imaginary market. The active bin, marked with an in-line comment, has 1 token X and 1 token Y in its reserves. You can appreciate that such a liquidity composition is highly probable when the price of the asset increases rapidly, leaving the concentrated liquidity behind. Even when this type of a distribution is not readily available, the price can be manipulated to create a similar distribution allowing the profitable execution of the described trick to steal flashloan fees.

```js
[
	[ 0, 10 ],
	[ 0, 30 ],
	[ 0, 40 ],
	[ 0, 90 ],
	[ 0, 80 ],
	[ 0, 50 ],
	[ 0, 30 ],
	[ 0, 10 ],
	[ 0,  9 ],
	[ 1,  1 ], // Active bin
	[ 1,  0 ]
]
``` 

Here, a flashloan user can do the following transactions atomically (note: function arguments are simplified to portray the idea):

```solidity
    LBRouter.addLiquidity({
        binId: ACTIVE_BIN,
        tokenXAmount: 9,
        tokenYAmount: 9
    }); 
    LBPair.flashLoan({
        tokenXAmount: 359,
        tokenYAmount: 0
    });
    LBRouter.removeLiquidity({
        binId: ACTIVE_BIN,
        tokenXAmount: 9,
        tokenYAmount: 9
    }); 
```

With this method, the borrower will receive 90% of the liquidity provider fees of the flashloan, even though they only had 2.5% of the token X liquidity. This method is very likely to cause majority of the flashloan fees leaking out of the protocol, denying liquidity providers their revenue. Even if the average flashloan borrower does not utilize this strategy to reduce the effective fee they pay, the trick would surely be used by MEV users sandwiching the flashloan transactions to receive the majority of the fees.

## Tools Used

Manual review

## Recommended Mitigation Steps

The ideal remediation would be to have a separate global fee structure for a given pair, and record token X and token Y fees per share separately. So if user A has only token X, they should only receive fee when token X is loaned, and not when token Y is loaned. But user A should receive their fee regardless of which bin their liquidity is in. However, given that users’ token reserves change dynamically, there appears to be no simple way of achieving this.

If the ideal remediation cannot be achieved, a compromise would be to distribute the flashloan fees from the -nth populated bin to the +nth populated bin, where `n` is respective to the active bin. `n` could be an adjustable market parameter. A higher value of `n` would make the flashloan gas cost higher, but it would reduce the feasibility of the issues described in this finding. Admittedly, this is not a perfect solution: As long as an incomplete subset of liquidity providers or bins receive the flashloan fees, there will be bias hence a risk of inequitable or exploitable fee distribution.

If the compromise is not deemed sufficient, the alternative would be to remove the flashloan feature from the protocol altogether. If the liquidity providers cannot equitably receive their flashloan fees, it might not worth to expose them to the risks of flashloans.