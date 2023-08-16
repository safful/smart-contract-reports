## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Unnecessary Multiple Return Statements (PrizePool.sol)](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/15) 

# Handle

ye0lde


# Vulnerability details

## Impact

Gas savings and code clarity

## Proof of Concept

PrizePool.sol:
https://github.com/pooltogether/v4-core/blob/35b00f710db422a6193131b7dc2de5202dc4677c/contracts/prize-pool/PrizePool.sol#L383-L387

## Tools Used
Visual Studio Code, Remix

## Recommended Mitigation Steps
Replace this
https://github.com/pooltogether/v4-core/blob/35b00f710db422a6193131b7dc2de5202dc4677c/contracts/prize-pool/PrizePool.sol#L383-L387
with
<code>
return (ticket == _controlledToken) 
</code>

