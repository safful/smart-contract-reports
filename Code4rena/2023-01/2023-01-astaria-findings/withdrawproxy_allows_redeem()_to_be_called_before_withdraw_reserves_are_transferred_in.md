## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-16

# [WithdrawProxy allows redeem() to be called before withdraw reserves are transferred in](https://github.com/code-423n4/2023-01-astaria-findings/issues/358) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/WithdrawProxy.sol#L152-L161


# Vulnerability details

## Impact
The WithdrawProxy contract has the `onlyWhenNoActiveAuction` modifier on the `withdraw()` and `redeem()` functions. This modifier stops these functions from being called when an auction is active:

```
modifier onlyWhenNoActiveAuction() {
  WPStorage storage s = _loadSlot();
  if (s.finalAuctionEnd != 0) {
    revert InvalidState(InvalidStates.NOT_CLAIMED);
  }
  _;
}
```
Furthermore, both `withdraw()` and `redeem()` can only be called when `totalAssets() > 0` based on logic within those functions.

The intention of these two checks is that the WithdrawProxy shares can only be cashed out after the PublicVault has called `transferWithdrawReserve`.

However, because `s.finalAuctionEnd == 0` before an auction has started, and `totalAssets()` is calculated by taking the balance of the contract directly (`ERC20(asset()).balanceOf(address(this));`), a user may redeem their shares before the vault has been fully funded, and take less than their share of the balance, benefiting the other withdrawer.

## Proof of Concept

- A depositor decides to withdraw from the PublicVault and receives WithdrawProxy shares in return
- A malicious actor deposits a small amount of the underlying asset into the WithdrawProxy, making `totalAssets() > 0`
- The depositor accidentally redeems, or is tricked into redeeming, from the WithdrawProxy, getting only a share of the small amount of the underlying asset rather than their share of the full withdrawal
- PublicVault properly processes epoch and full withdrawReserve is sent to WithdrawProxy
- All remaining holders of WithdrawProxy shares receive an outsized share of the withdrawReserve

## Tools Used

Manual Review

## Recommended Mitigation Steps

Add an additional storage variable that is explicitly switched to `open` when it is safe to withdraw funds.