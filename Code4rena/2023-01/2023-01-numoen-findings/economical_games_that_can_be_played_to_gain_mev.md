## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-03

# [Economical games that can be played to gain MEV](https://github.com/code-423n4/2023-01-numoen-findings/issues/242) 

# Lines of code

https://github.com/code-423n4/2023-01-numoen/blob/2ad9a73d793ea23a25a381faadc86ae0c8cb5913/src/core/Pair.sol#L53-L140


# Vulnerability details

_Disclaimer: Developers did an extremely good job writing the protocol, however, these are some aspects that I think are missed in the design stage and can be considered. Look at it as a food for thought in future designs._
## Impact
### How the invariant works
The invariant of the project is a power formula that follows:
```
k = x − (p_0 + (-1/2) * y)^2
```
Where it is implemented by the code below:
```solidity
  function invariant(uint256 amount0, uint256 amount1, uint256 liquidity) public view override returns (bool) {
    if (liquidity == 0) return (amount0 == 0 && amount1 == 0);

    uint256 scale0 = FullMath.mulDiv(amount0, 1e18, liquidity) * token0Scale;
    uint256 scale1 = FullMath.mulDiv(amount1, 1e18, liquidity) * token1Scale;

    if (scale1 > 2 * upperBound) revert InvariantError();

    uint256 a = scale0 * 1e18;
    uint256 b = scale1 * upperBound;
    uint256 c = (scale1 * scale1) / 4;
    uint256 d = upperBound * upperBound;

    return a + b >= c + d;
  }
```
Where `x` is equal to `scale0`, y is equal to `scale1` and `p0` is the upper bound. The graph that draws the acceptable point by the invariant is shown below:

[invariant image](https://user-images.githubusercontent.com/18353616/216128438-a43afa14-c96d-413e-9fc5-5279da6fc852.png)

We can see that the `scale1` does not put a hard cap on `scale0`, but `scale0` does. Also the upper half of the plot is not acceptable by the plot. Overall, it is expected by the protocol that the `(scale0, scale1) stays on the curve on the bottom. 
The derivative of the equation is:
```
dx/dy = x/2 - p0
or
d(scale0)/d(scale1) = scale1 / 2 - p0
```

Which means whenever the price of the two tokens is different than this derivative, there is an arbitrage opportunity. The reason `p0` is called upper bound is that the protocol only anticipates the price fraction until the price of asset1 is p0 times the price of asset0 (The curve needs scale1 to be less than zero to support lower prices which is not possible). when `scale1 = 2*p0`, the price of the scale1/scale0 is zero and scale0 is infinite times more valuable than scale1 token. This is the value used to make sure a position is never undercollateralized.

### The problem
The liquidity market of a `lendgine` is revolved around `upperbound`. Liquidity providers are looking for the highest `upperbound` possible where the borrowers are providing the most collateral. And the borrowers are looking for the lowest `upperbound` possible where they lock in the least collateral for the most liquidity. Therefore, there should be a middle ground reached by the two sides. The middle ground for the both side is an uppderbound that is far away from the current price, where the liquidity providers feel safe, and is close enough to the actual price that the borrowers find the fees they pay logical. However, if the `upperbound` is a function of how close it is to the actual price, and the actual relative price of the two tokens is volatile, accepted `upperbound` will change through time as well. Therefore we can expect that for two tokens, the liquidity will be moving from one market to another as the accepted `upperbound` value changes. This means that if a lendgine is busy one day, might not be so busy the other day with the price change. This is not a problem by itself, but can leave some liquidity providers behind in locked markets which is explained in the proof of concept.
The second problem comes from the fact that while the lendgine algorithm makes sure a position is never undercollateralized, it does not value bigger markets more than the smaller ones. This means that a lender while lending, only cares about the smallest `upperbound` possible and the liquidity market would be basically a set of price bids, if a borrwer wants to borrow amount `B` from the whole market, starts from the smallest `upperbound` and if there is not enough liquidity in the smaller one, it makes its way up until he has `B` borrowed. (of course he will consider the fee that he should pay) Therefore, this would cause the liquidity providing market to be extremely scattered, and for each lendgine, liquidity providing is highly centralized (since many lendgines can be made and the upper bound value can be controversial).

## Proof of concept
Lets consider several cases: (These also happen in other markets, but can get exaggerated here)
- Imagine a liquidity market which has up to a considerable percentage of its liquidity borrowed, if the safe `upperbound` for the liquidators starts to move down the protocol allows the earliest liquidators to opt-out, creating a certain kind of MEV for the fastest liquidators. While the remaining providers will get more fees, the protocol favors the fastest actors to decide when to opt-out. In the extreme case of base token crashing down, there would be a race between borrowers to lock the money and the earliest liquidity providers to get out.
- Liquidity providers might mint some shares for themselves in times of uncertainty, just to have the option to quickly opt-out of the protocol if they need. They can give back the borrowed amount and withdraw the said amount in one transaction. while if they do not lock the funds, they either have to take the funds out or someone else might come and get lock the funds.

## Recommended Mitigation Steps
There are two things that could be done in the future to mitigate issues:
- value the bigger markets more than the smaller ones, where users are incentivized to use the bigger markets.
- Use an aggregator that crawls over several markets and let liquidity providers to stake in a range of liquidity.