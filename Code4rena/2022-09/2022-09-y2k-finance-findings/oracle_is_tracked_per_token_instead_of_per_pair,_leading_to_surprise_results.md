## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [Oracle is tracked per token instead of per pair, leading to surprise results](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/100) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/2175c044af98509261e4147edeb48e1036773771/src/VaultFactory.sol#L121
https://github.com/code-423n4/2022-09-y2k-finance/blob/2175c044af98509261e4147edeb48e1036773771/src/VaultFactory.sol#L221-L223


# Vulnerability details

## Impact
Oracles are tracked by individual token, instead of by pair. Because for some tokens (ie BTC) it is unclear which implementation is the canonical one to compare others to, this can result in a situation where different pairs (ie ibBTC-wBTC and ibBC-renBTC) using the same oracle, which will be incorrect for one of them.

## Proof of Concept
The protocol assumes that there is a canonical asset to compare pegged assets to, so oracles are tracked only by the pegged asset. However, for some assets (like BTC), there is no clear canonical asset, and the result is that tracking `tokenToOracle` is not sufficient.

When there is a conflict in `tokenToOracle`, the protocol responds by skipping the assignment and keeping the old value:
```
if (tokenToOracle[_token] == address(0)) {
    tokenToOracle[_token] = _oracle;
}
```

The result of this is that the protocol may define a new pair with a new oracle, and have it silently skip it and use a non-matching oracle. As an example:
- The admins start with an implementation of ibBTC-wBTC, deploying the oracle
- This is set as `tokenToOracle[ibBTC]`
- Later, the admins create a new pair for ibBTC-renBTC, deploying a new oracle
- The protocol silently skips this assignment and uses the ibBTC-wBTC oracle

This can produce incorrect results, for example if wBTC appreciates relative to the other two, or if both ibBTC and renBTC depeg.

## Tools Used

Manual Review, Foundry

## Recommended Mitigation Steps

Change `tokenToOracle` to represent the pair of tokens, either by creating a `Pair` struct as the key, or by nesting a mapping inside of another mapping.