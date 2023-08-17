## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-02

# [First liquidity provider will suffer from revert or fund loss](https://github.com/code-423n4/2023-01-numoen-findings/issues/254) 

# Lines of code

https://github.com/code-423n4/2023-01-numoen/blob/2ad9a73d793ea23a25a381faadc86ae0c8cb5913/src/periphery/LiquidityManager.sol#L135


# Vulnerability details

## Impact
The first liquidity depositor should supply three input values `amount0Min, amount1Min, liquidity` via `AddLiquidityParams` but these three values should meet an accurate relationship, or else the depositor will suffer from revert or fund loss

## Proof of Concept
The LPs are supposed to use the function `LiquidityManager.addLiquidity(AddLiquidityParams calldata params)` to add liquidity.
When the pool is not empty, this function calculates the `amount0, amount1` according to the current total liquidity and the requested liquidity.
But when the pool is empty, these amounts are supposed to be provided by the caller.
```solidity
LiquidityManager.sol

120:   struct AddLiquidityParams {
121:     address token0;
122:     address token1;
123:     uint256 token0Exp;
124:     uint256 token1Exp;
125:     uint256 upperBound;
126:     uint256 liquidity;
127:     uint256 amount0Min;
128:     uint256 amount1Min;
129:     uint256 sizeMin;
130:     address recipient;
131:     uint256 deadline;
132:   }
133:
134:   /// @notice Add liquidity to a liquidity position
135:   function addLiquidity(AddLiquidityParams calldata params) external payable checkDeadline(params.deadline) {
136:     address lendgine = LendgineAddress.computeAddress(
137:       factory, params.token0, params.token1, params.token0Exp, params.token1Exp, params.upperBound
138:     );
139:
140:     uint256 r0 = ILendgine(lendgine).reserve0();
141:     uint256 r1 = ILendgine(lendgine).reserve1();
142:     uint256 totalLiquidity = ILendgine(lendgine).totalLiquidity();
143:
144:     uint256 amount0;
145:     uint256 amount1;
146:
147:     if (totalLiquidity == 0) {
148:       amount0 = params.amount0Min;//@audit-info caller specifies the actual reserve amount
149:       amount1 = params.amount1Min;//@audit-info
150:     } else {
151:       amount0 = FullMath.mulDivRoundingUp(params.liquidity, r0, totalLiquidity);
152:       amount1 = FullMath.mulDivRoundingUp(params.liquidity, r1, totalLiquidity);
153:     }
154:
155:     if (amount0 < params.amount0Min || amount1 < params.amount1Min) revert AmountError();
156:
157:     uint256 size = ILendgine(lendgine).deposit(
158:       address(this),
159:       params.liquidity,
160:       abi.encode(
161:         PairMintCallbackData({
162:           token0: params.token0,
163:           token1: params.token1,
164:           token0Exp: params.token0Exp,
165:           token1Exp: params.token1Exp,
166:           upperBound: params.upperBound,
167:           amount0: amount0,
168:           amount1: amount1,
169:           payer: msg.sender
170:         })
171:       )
172:     );
173:     if (size < params.sizeMin) revert AmountError();
174:
175:     Position memory position = positions[params.recipient][lendgine]; // SLOAD
176:
177:     (, uint256 rewardPerPositionPaid,) = ILendgine(lendgine).positions(address(this));
178:     position.tokensOwed += FullMath.mulDiv(position.size, rewardPerPositionPaid - position.rewardPerPositionPaid, 1e18);
179:     position.rewardPerPositionPaid = rewardPerPositionPaid;
180:     position.size += size;
181:
182:     positions[params.recipient][lendgine] = position; // SSTORE
183:
184:     emit AddLiquidity(msg.sender, lendgine, params.liquidity, size, amount0, amount1, params.recipient);
185:   }

```
Then how does the caller decides these amount? These values should be chosen very carefully as we explain below.

The whole protocol is based on its invariant that is defined in `Pair.invariant()`.
The invariant is actually ensuring that `a+b-c-d` stays not negative for all trades (interactions regarding reserve/liquidity).
Once `a+b-c-d` becomes strictly positive, anyone can call `swap()` function to pull the `token0` of that amount without any cost.
```solidity
Pair.sol
52:   /// @inheritdoc IPair
53:   function invariant(uint256 amount0, uint256 amount1, uint256 liquidity) public view override returns (bool) {
54:     if (liquidity == 0) return (amount0 == 0 && amount1 == 0);
55:
56:     uint256 scale0 = FullMath.mulDiv(amount0, 1e18, liquidity) * token0Scale;
57:     uint256 scale1 = FullMath.mulDiv(amount1, 1e18, liquidity) * token1Scale;
58:
59:     if (scale1 > 2 * upperBound) revert InvariantError();
60:
61:     uint256 a = scale0 * 1e18;
62:     uint256 b = scale1 * upperBound;
63:     uint256 c = (scale1 * scale1) / 4;
64:     uint256 d = upperBound * upperBound;
65:
66:     return a + b >= c + d;//@audit-info if strict inequality holds, anyone can pull token0 using swap()
67:   }
```

So going back to the question, if the LP choose the values `amount0, amount1, liquidity` not accurately, the transaction reverts or `a+b-c-d` becomes greater than zero.

Generally, liquidity providers do not specify the desired liquidity amount in other protocols.
During the conversation with the sponsor team, it is understood that they avoided the calculation of `liquidity` from `amount0, amount1` because it is too complicated.
Off-chain calculation will be necessary to help the situation, and this would limit the growth of the protocol.
If any other protocol is going to integrate Numoen, they will face the same problem.

I did some calculation and got the formula for the liquidity as below.

$
L = \frac{PCy+C^2x+\sqrt{2PC^3xy+C^4x^2}}{2P^2}
$
where $C=10^{18}$, $x$ is `amount0`, $y$ is `amount1`, $P$ is the `upperBound`, $L$ is the liquidity amount that should be used.

Because the LP will almost always suffer revert or fund loss without help of off-chain calculation, I submit this as a medium finding.
I would like to note that there still exists a mitigation (not that crazy).
As a side note, it would be very helpful to add new preview functions.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Add a functionality to calculate the liquidity for the first deposit on-chain.
And it is also recommended to add preview functions.