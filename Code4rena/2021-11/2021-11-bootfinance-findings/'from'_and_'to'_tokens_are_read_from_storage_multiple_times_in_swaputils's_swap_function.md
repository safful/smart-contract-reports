## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# ['From' and 'to' tokens are read from storage multiple times in SwapUtils's swap function](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/190) 

# Handle

hyh


# Vulnerability details

## Impact

Gas is overspent due to excessive storage reads

## Proof of Concept

SwapUtils's swap: saving self.pooledTokens[tokenIndexFrom], which do not change, to memory and reusing will reduce gas costs.
https://github.com/code-423n4/2021-11-bootfinance/blob/main/customswap/contracts/SwapUtils.sol#L1098

## Recommended Mitigation Steps

Now:
self.pooledTokens[tokenIndexFrom].balanceOf(msg.sender)...
...
uint256 beforeBalance = self.pooledTokens[tokenIndexFrom].balanceOf(address(this));
self.pooledTokens[tokenIndexFrom].safeTransferFrom(
...
uint256 transferredDx = self.pooledTokens[tokenIndexFrom].balanceOf(address(this)).sub(beforeBalance);
To be:
IERC20 memory fromToken = self.pooledTokens[tokenIndexFrom];
fromToken.balanceOf(msg.sender)...
...
uint256 beforeBalance = fromToken.balanceOf(address(this));
fromToken.safeTransferFrom(
...
uint256 transferredDx = fromToken.balanceOf(address(this)).sub(beforeBalance);

