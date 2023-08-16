## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Redundant `require` in Swivel.sol](https://github.com/code-423n4/2021-09-swivel-findings/issues/129) 

# Handle

GalloDaSballo


# Vulnerability details

## Impact
The line:
    require(MarketPlace(marketPlace).p2pZcTokenExchange(o.underlying, o.maturity, o.maker, msg.sender, a), 'zcToken exchange failed');

in Swivel.sol
https://github.com/Swivel-Finance/gost/blob/5fb7ad62f1f3a962c7bf5348560fe88de0618bae/test/swivel/Swivel.sol#L171

Is checking for the return value of `MarketPlace.p2pZcTokenExchange`, however `p2pZcTokenExchange` will always return true or revert

As such the require is not necessary, and doesn't provide any additional guarantees.

## Recommended Mitigation Steps

Replace
`
    require(MarketPlace(marketPlace).p2pZcTokenExchange(o.underlying, o.maturity, o.maker, msg.sender, a), 'zcToken exchange failed');
`

With

`
MarketPlace(marketPlace).p2pZcTokenExchange(o.underlying, o.maturity, o.maker, msg.sender, a)
`

