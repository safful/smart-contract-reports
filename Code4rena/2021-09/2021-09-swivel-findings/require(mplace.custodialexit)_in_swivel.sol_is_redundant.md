## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [require(mPlace.custodialExit) in Swivel.sol is redundant](https://github.com/code-423n4/2021-09-swivel-findings/issues/128) 

# Handle

GalloDaSballo


# Vulnerability details

## Impact
The function `custodialExit` in Marketplace.sol: https://github.com/Swivel-Finance/gost/blob/5fb7ad62f1f3a962c7bf5348560fe88de0618bae/test/marketplace/MarketPlace.sol#L194

Always returns true or reverts

In swivel.sol the check 
`require(mPlace.custodialExit(o.underlying, o.maturity, o.maker, msg.sender, a), 'custodial exit failed');
`
will always pass, unless the function `custodialExit` reverts

This extra require is not necessary, and provides no additional guarantees as `custodialExit` will always return true or revert


## Recommended Mitigation Steps
Replace
`
require(mPlace.custodialExit(o.underlying, o.maturity, o.maker, msg.sender, a), 'custodial exit failed');
`

With
`
mPlace.custodialExit(o.underlying, o.maturity, o.maker, msg.sender, a)
`

