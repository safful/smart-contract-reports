## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-04

# [Function `_execBuyNftFromMarket()` Fails to Check the Actual ETH Balance in the Contract After Executing the Trade](https://github.com/code-423n4/2023-05-particle-findings/issues/7) 

# Lines of code

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L420


# Vulnerability details

## Impact
In the function `_execBuyNftFromMarket()`, if the user chooses to use `WETH`, the function deposits ETH and approves the `amount` of WETH to the marketplace. After executing the trade at the marketplace, the function checks that the balance decrease is correct in the end. However, this check only accounts for ETH changes, not WETH changes, which is incorrect. If the trade did not use the full amount of WETH approved to the marketplace, some leftover WETH will remain in the contract. This amount of WETH/ETH will be locked in the contract, even though it should belong to the borrower who was able to get a good offer to buy the NFT at a lower price.

```solidity
if (useToken == 0) {
    // use ETH
    // solhint-disable-next-line avoid-low-level-calls
    (success, ) = marketplace.call{value: amount}(tradeData);
} else if (useToken == 1) {
    // use WETH
    weth.deposit{value: amount}();
    weth.approve(marketplace, amount);
    // solhint-disable-next-line avoid-low-level-calls
    
    // @audit might not use all amount approved, cause ETH to locked in the protocol
    (success, ) = marketplace.call(tradeData); 
}

if (!success) {
    revert Errors.MartketplaceFailedToTrade();
}

// verify that the declared NFT is acquired and the balance decrease is correct
if (IERC721(collection).ownerOf(tokenId) != address(this) || balanceBefore - address(this).balance != amount) {
    revert Errors.InvalidNFTBuy();
}
```

## Proof of Concept
Consider the following scenario:

1. Alice (borrower 1) calls `buyNftFromMarket()` to acquire an NFT with a price of 100 WETH. However, she sets the amount to 105 WETH, so the contract deposits and approves 105 WETH to the marketplace. After the trade, there is still 5 WETH approved to the marketplace.
2. Bob (borrower 2) sees the opportunity. He has a much cheaper lien, so he also calls `buyNftFromMarket()` to acquire an NFT with a price of 5 WETH. He specifies the `useToken = 0`. However, he sets the `amount = 0` and actually uses the 5 WETH left in step 1 of Alice to acquire the NFT. The result is Bob is able to steal 5 WETH approved to the marketplace.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider accounting for the WETH when checking balance changes in `_execBuyNftFromMarket()`.



## Assessed type

Invalid Validation