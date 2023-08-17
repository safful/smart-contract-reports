## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-05

# [Pair price may be manipulated by direct transfers](https://github.com/code-423n4/2022-12-caviar-findings/issues/383) 

# Lines of code

https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L391
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L479-L480
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L384


# Vulnerability details

## Impact
An attacker may manipulate the price of a pair by transferring tokens directly to the pair. Since the `Pair` contract exposes the `price` function, it maybe be used as a price oracle in third-party integrations. Manipulating the price of a pair may allow an attacker to steal funds from such integrations.
## Proof of Concept
The `Pair` contract is a pool of two tokens, a base token and a fractional token. Its main purpose is to allow users to swap the tokens at a fair price. Since the price is calculated based on the reserves of a pair, it can only be changed in two cases:
1. when initial liquidity is added: the first liquidity provider sets the price of a pool ([Pair.sol#L85-L97](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L85-L97)); other liquidity providers cannot change the price ([Pair.sol#L421-L423](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L421-L423));
1. during trades: trading adds and removes tokens from a pool, ensuring the K constant invariant is respected ([Pair.sol#L194-L204](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L194-L204), [Pair.sol#L161-L173](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L161-L173)).

However, the Pair contract calculates the price using the current token balances of the contract ([Pair.sol#L379-L385](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L379-L385), [Pair.sol#L477-L481](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L477-L481)):
```solidity
function baseTokenReserves() public view returns (uint256) {
    return _baseTokenReserves();
}

function _baseTokenReserves() internal view returns (uint256) {
    return baseToken == address(0)
        ? address(this).balance - msg.value // subtract the msg.value if the base token is ETH
        : ERC20(baseToken).balanceOf(address(this));
}

function fractionalTokenReserves() public view returns (uint256) {
    return balanceOf[address(this)];
}
```

This allows an attacker to change the price of a pool and skip the K constant invariant check that's enforced on new liquidity ([Pair.sol#L421-L423](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L421-L423)).
## Tools Used
Manual review
## Recommended Mitigation Steps
Consider tracking pair's reserves internally, using state variables, similarly to how Uniswap V2 does that:
- [UniswapV2Pair.sol#L22-L23](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L22-L23):
```solidity
uint112 private reserve0;           // uses single storage slot, accessible via getReserves
uint112 private reserve1;           // uses single storage slot, accessible via getReserves
```

- [UniswapV2Pair.sol#L38-L42](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L38-L42):
```solidity
function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1, uint32 _blockTimestampLast) {
    _reserve0 = reserve0;
    _reserve1 = reserve1;
    _blockTimestampLast = blockTimestampLast;
}
```

- [UniswapV2Pair.sol#L38-L42](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L38-L42):
```solidity
// update reserves and, on the first call per block, price accumulators
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
    require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
    uint32 blockTimestamp = uint32(block.timestamp % 2**32);
    uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
        // * never overflows, and + overflow is desired
        price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
        price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
    }
    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
    blockTimestampLast = blockTimestamp;
    emit Sync(reserve0, reserve1);
}
```