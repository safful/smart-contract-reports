## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-24

# [FlashAuction doesn't pass the initiator to the recipient ](https://github.com/code-423n4/2023-01-astaria-findings/issues/166) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/CollateralToken.sol#L307-L310


# Vulnerability details

## Impact
Existing flashloans pass the initiator to the recipient in the function's params, e.g. [AAVE](https://github.com/aave/aave-v3-core/blob/master/contracts/flashloan/interfaces/IFlashLoanReceiver.sol#L29) or [Uniswap](https://github.com/Uniswap/v2-core/blob/master/contracts/interfaces/IUniswapV2Callee.sol#L4). It's done to allow the recipient to verify the legitimacy of the call.

The CollateralToken's `flashAction()` function doesn't pass the initiator. That makes it more difficult to integrate the flashAction functionality. To protect yourself against other people executing the callback you have to:
- create a flag
- add an authorized function that sets the flag to true
- only allow `onFlashAction()` to be executed when the flag is true

Considering that most people won't bother with that there's a significant risk of this being abused. While this doesn't affect the Astaria protocol directly, it potentially affects its users and their funds. Because of that, I decided to rate it as MED.

## Proof of Concept
`onFlashAction()` is executed with the ERC721 token data (contract addr, ID, and return address) as well as the arbitrary `data` bytes.
```sol
receiver.onFlashAction(
        IFlashAction.Underlying(s.clearingHouse[collateralId], addr, tokenId),
        data
      )
```
## Tools Used
none

## Recommended Mitigation Steps
Add the initiator to the callback to allow easy authentication.