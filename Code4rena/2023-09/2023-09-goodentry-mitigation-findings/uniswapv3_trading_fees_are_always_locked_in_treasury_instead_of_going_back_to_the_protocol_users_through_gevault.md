## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report

# [UniswapV3 trading fees are always locked in treasury instead of going back to the protocol users through GeVault](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/18) 

# Lines of code

https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/TokenisableRange.sol#L179


# Vulnerability details

TokenisableRange was redesigned to redirect collected fees to a pre-defined GeVault, where the protocol stakers can benefit from the added value.

However, the use of an incorrect variable makes this distribution of the fees impossible to happen, and the fees will necessarily be sent to the the protocol treasury:

- In TokenisableRange.sol, where fees are distributed, the vault lookup checks for a vault of (token0, token0) instead of (token0, token1):
```Solidity
      try RoeRouter(roerouter).getVault(address(TOKEN0.token), address(TOKEN0.token)) returns (address _vault) {
        vault = _vault;
      }
```
- Because of the validation happening in RoeRouter, no GeVault can possibly be configured for a pair of the same address to work around the problem once the protocol is live:
```Solidity
  function setVault(address token0, address token1, address vault) public onlyOwner {
    require(token0 < token1, "Invalid Order");
```

- This will lead to fee funds being temporarily locked in Treasury instead of being distributed to the protocol users through the designed vault

## Impact
Users will see the fees generated from their participation to the protocol taken away by the protocol instead of redistributed to them.

The funds are only temporarily locked because the protocol team can act to forward them to the appropriate GeVault, but users may perceive this issue as the protocol stealing trading fees from them.

## Proof of Concept
- Have a random account as `vault` - it can be an EOA for the sake of this PoC
- Call `RoeRouter.setVault(USDC, WETH, vault)`
- Create and fund a TokenisableRange for USDC/WETH
- Mock Uniswap V3 to arrange some fees for this TokenisableRange
- call the range's `claimFee()` function
- check the token balance of `vault` - this will be unchanged
- check the token balance of the TokenisableRange treasury - this will have received the fees

## Tools Used
Code review, Foundry

## Recommended Mitigation Steps
Consider fixing the `getVault` call as follows:
```diff
-      try RoeRouter(roerouter).getVault(address(TOKEN0.token), address(TOKEN0.token)) returns (address _vault) {
+      try RoeRouter(roerouter).getVault(address(TOKEN0.token), address(TOKEN1.token)) returns (address _vault) {
        vault = _vault;
      }
```

Additionally, you may want to make `RoeRouter` configurable at least in the `TokenisableRange` constructor, because the version deployed at the currently hardcoded address `0x22Cc3f665ba4C898226353B672c5123c58751692` does not have a `getVault` method and is not upgradeable.


## Assessed type

Token-Transfer