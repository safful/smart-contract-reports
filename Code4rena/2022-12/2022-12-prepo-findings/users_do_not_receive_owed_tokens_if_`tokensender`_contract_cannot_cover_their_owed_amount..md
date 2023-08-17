## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- M-07

# [Users do not receive owed tokens if `TokenSender` contract cannot cover their owed amount.](https://github.com/code-423n4/2022-12-prepo-findings/issues/257) 

# Lines of code

https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/TokenSender.sol#L41


# Vulnerability details

### Description
​
The `TokenSender.send()` method is called during the course of users withdrawing or redeeming tokens from the protocol. The method is called via `DepositHook.hook()`, `RedeemHook.hook()`, and `WithdrawHook.hook()`. These in turn are called in `prePOMarket.redeem()` or `Collateral.deposit()|.withdraw()`
​
`TokenSender.send()` contains some logic to return early without sending any of the "outputToken", such as if the price of the outputToken has fallen below an adjustable lower bound, or if the amount would be 0.
​
However, it also checks its own balance to see if it can cover the required amount. If it cannot, it simply doesn't send tokens. These tokens are intended to be a compensation for fees paid elsewhere in the process, and thus do represent a value loss.
​
```
function send(address recipient, uint256 unconvertedAmount) external override onlyAllowedMsgSenders { 
    uint256 scaledPrice = (_price.get() * _priceMultiplier) / MULTIPLIER_DENOMINATOR;
    if (scaledPrice <= _scaledPriceLowerBound) return; 
    uint256 outputAmount = (unconvertedAmount * _outputTokenDecimalsFactor) / scaledPrice;
    if (outputAmount == 0) return;
    if (outputAmount > _outputToken.balanceOf(address(this))) return; // don't send if not enough balance
    _outputToken.transfer(recipient, outputAmount);
}
```
​
The documentation in `ITokenSender.sol` states this is so the protocol doesn't halt the redeem and deposit/withdraw actions.
​
### Severity Rationalization
​
The warden agrees that the protocol halting is generally undesirable. 
​
However, there isn't any facility in the code for the user who triggered the overage amount to be able to later receive their tokens when the contract is topped up. They must rely upon governance to send them any owed tokens. This increases centralization risks and isn't necessary.
​
Since the contract makes no attempt to track the tokens that should have been sent, manually reviewing and verifying owed tokens becomes a non-trivial task if any more than a handful of users were affected.
​
Since the user did receive their underlying collateral in any case and the loss isn't necessarily permanent, medium seems to be the right severity for this issue.
​
​
### Proof of Concept
​
Bob wants to redeem his long and short tokens via `PrePOMarket.redeem()`. However, Alice's redemption prior to his significantly drained the `TokenSender` contract of its tokens. As a result, Bob's redemption fails to benefit him in the amount of the outputToken he should have received in compensation for the fees paid. 

Because the quantity of tokens paid to Bob is partially dependent upon the token's price at the time of redemption, the protocol might shoulder more  downside loss (token price dropped compared to when Bob redeemed, must pay out more tokens) or Bob might suffer upside loss (price went up compared to time of redemption, Bob looses the difference).
​
Bob's recourse is to contact the project administrators and try to have his tokens sent to him manually. Agreeing to a value adds friction to the process.
​
### Mitigation
​
The `TokenSender` contract should track users whose balance wasn't covered in a mapping, as well as a function for them to manually claim tokens later on if the contract's balance is topped up. Such a function might record the price at redemption time, or it might calculate it with the current price.