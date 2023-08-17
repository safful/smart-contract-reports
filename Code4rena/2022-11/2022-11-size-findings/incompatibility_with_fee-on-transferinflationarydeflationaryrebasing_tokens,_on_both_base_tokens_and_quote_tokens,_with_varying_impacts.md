## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- edited-by-warden
- M-01

# [Incompatibility with fee-on-transfer/inflationary/deflationary/rebasing tokens, on both base tokens and quote tokens, with varying impacts](https://github.com/code-423n4/2022-11-size-findings/issues/47) 

# Lines of code

https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L163
https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L96-L105


# Vulnerability details

The following report describes two issues with how the `SizeSealed` contract incorrectly handles several so-called "weird ERC20" tokens, in which the token's balance can change unexpectedly:
- How the contract cannot handle fee-on-transfer base tokens, and
- How the contract incorrectly handles unusual ERC20 tokens in general, with stated impact.

## Description

### Base tokens

Let us first note how the contract attempts to handle sudden balance change to the `baseToken`:

https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L96-L105

```solidity
uint256 balanceBeforeTransfer = ERC20(auctionParams.baseToken).balanceOf(address(this));

SafeTransferLib.safeTransferFrom(
    ERC20(auctionParams.baseToken), msg.sender, address(this), auctionParams.totalBaseAmount
);

uint256 balanceAfterTransfer = ERC20(auctionParams.baseToken).balanceOf(address(this));
if (balanceAfterTransfer - balanceBeforeTransfer != auctionParams.totalBaseAmount) {
    revert UnexpectedBalanceChange();
}
```

The effect is that the operation will revert with the error `UnexpectedBalanceChange()` if the received amount is different from what was transferred. 

### Quote tokens

Unlike base tokens, there is no such check when transferring the `quoteToken` from the bidder:

https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L163

```solidity
SafeTransferLib.safeTransferFrom(ERC20(a.params.quoteToken), msg.sender, address(this), quoteAmount);
```

Since [line 150](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L150) stores the `quoteAmount` as a state variable, which will be used later on during outgoing transfers (see following lines), this results in incorrect state handling.

It is worth noting that this will effect **all** functions with outgoing transfers, due to reliance on storage values.
- https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L327
- https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L351
- https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L381
- https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L439

## Proof of concept

Consider the following scenario, describing the issue with both token types:

1. Alice places an auction, her base token is a rebasing token (e.g. aToken from Aave).
2. Bob places a bid with `1e18` quote tokens. This quote token turns out to be a fee-on-transfer token, and the contract actually received `1e18 - fee`.
3. Some time later, Alice cancels the auction and withdraws her aTokens.
4. Bob is now unable to withdraw his bid, due to failed transfers for the contract not having enough balance.

For Alice's situation, she is able to recover her original amount. However, since aTokens increases one's own balance over time, the "interest" amount is permanently locked in the contract.

Bob's situation is somewhat more recoverable, as he can simply send more tokens to the contract itself, so that the contract's balance is sufficient to refund Bob his tokens. However, given that Bob now has to incur more transfer fees to get his own funds back, I consider this a leakage of value.

It is worth noting that similar impacts **will happen to successful auctions** as well, although it is a little more complicated, and that it varies in the matter of who is affected.

## Impact

For the base token:
- For fee-on-transfer tokens, the contract is simply unusable on these tokens. 
    - There exists [many](https://github.com/d-xo/weird-erc20#fee-on-transfer) popular fee-on-transfer tokens, and potential tokens where the fees may be switched on in the future.
- For rebasing tokens, part of the funds may be permanently locked in the contract.

For the quote token:
- If the token is a fee-on-transfer, or anything that results in the contract holding lower amount than stored, then this actually results in a denial-of-service, since outgoing transfers in e.g. `withdraw()` will fail due to insufficient balance. 
    - This may get costly to recover if a certain auction is popular, but is cancelled, so the last bidder to withdraw takes all the damage.
    - This can also be considered fund loss due to quote tokens getting stuck in the contract.
- For rebasing tokens, part of the funds may be permanently locked in the contract (same effect as base token).


## Remarks 

While technically two separate issues are described, they do have many overlappings, and both comes down to correct handling of unusual ERC20 tokens, hence I have decided to combine these into a single report.

Another intention was to highlight the similarities and differences between balance handling of base tokens and quote tokens, which actually has given rise to part of the issue itself.

## Tools Used

Manual review

## Recommended Mitigation Steps

For both issues: 
- Consider warning the users about the possibility of fund loss when using rebasing tokens as the quote token.
- Adding ERC20 recovering functions is an option, however this should be done carefully, so as not to accidentally pull out base or quote tokens from ongoing auctions.

For the base token issue:
- Consider using two parameters for transferring base tokens: e.g. `amountToTransferIn` for the amount to be pulled in, and `auctionParams.totalBaseAmount` for the actual amount received, then check the received amount is appropriate.
    - The calculation for these two amount should be the auction creator's responsibility, however this can be made a little easier for them by checking amount received is at least `auctionParams.totalBaseAmount` instead of exact, and possibly transferring the surplus amount back to the creator.

For the quote token issue:
- Consider adding a check for `quoteAmount` to be correctly transferred, or check the actual amount transferred in the contract instead of using the function parameter, or something similar to the base token's mitigation.

Additionally, consider using a separate internal function for pulling tokens, to ensure correct transfers.
