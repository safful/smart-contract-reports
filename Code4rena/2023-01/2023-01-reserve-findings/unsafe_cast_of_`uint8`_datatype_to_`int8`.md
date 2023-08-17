## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-14

# [Unsafe cast of `uint8` datatype to `int8`](https://github.com/code-423n4/2023-01-reserve-findings/issues/265) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BackingManager.sol#L228
https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BasketHandler.sol#L421


# Vulnerability details

## Impact
Converting uint8 to int8 can have unexpected consequences when done unsafely. This issue affects the quote function in BasketHandler.sol and handoutExcessAssets in BackingManager.sol. While there is some risk here, the issue is unlikely to be exploited as ERC-20 tokens generally don't have a decimals value over 18, nevertheless one over 127. 

## Proof of Concept

➜ int8(uint8(127))
Type: int
├ Hex: 0x7f
└ Decimal: 127
➜ int8(uint8(128))
Type: int
├ Hex: 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80
└ Decimal: -128

## Tools Used
Chisel

## Recommended Mitigation Steps
Validate that the decimals value is within an acceptable upper-bound before attempting to cast it to a signed integer. 