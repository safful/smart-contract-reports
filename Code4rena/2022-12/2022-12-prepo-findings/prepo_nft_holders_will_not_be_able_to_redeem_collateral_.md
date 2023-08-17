## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- sponsor disputed
- M-04

# [PrePO NFT holders will not be able to redeem collateral ](https://github.com/code-423n4/2022-12-prepo-findings/issues/101) 

# Lines of code

https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/RedeemHook.sol#L18


# Vulnerability details

## Impact

The protocol has set a limitation on who can participate in the protocol activities.
1. Users who are included in an allowed list: `_accountList`.
2. Users who own specific NFTs that are supported by NFTScoreRequirement. These NFTs are PrePO NFTs that were minted to accounts that historically participated in PrePO activities.

Users who are #2 that deposited funds into the protocol are not able to redeem collateral tokens and withdraw their profits/funds from the market. (Loss of funds)

## Proof of Concept

When a user deposited, the protocol checks if the user is permitted to participate in the protocol activities by checking #1 and #2 from the Impact section. The check is done in the `hook` function in `DepositHook`:
https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/DepositHook.sol#L45
```
  function hook(address _sender, uint256 _amountBeforeFee, uint256 _amountAfterFee) external override onlyCollateral {
-----
    if (!_accountList.isIncluded(_sender)) require(_satisfiesScoreRequirement(_sender), "depositor not allowed");
-----
  }
```

After a user has deposited and received collateral tokens, he will trade it in uniswap pools to receive Long/Short tokens either manually or through the `depositAndTrade` function. 

When the user decided to `redeem` through the market in order to receive the collateral tokens and his funds/profits, the user will not be able to receive it  because only users that are in the account list (#1) will pass the checks. Users who participated because they own NFT (#2) will get a revert when calling the function.

`redeem` in `PrePOMarket`:
https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/PrePOMarket.sol#L96
```
  function redeem(uint256 _longAmount, uint256 _shortAmount) external override nonReentrant {
-----
      _redeemHook.hook(msg.sender, _collateralAmount, _collateralAmount - _expectedFee);
-----
  }
```

`hook` function in `RedeemHook`:
https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/RedeemHook.sol#L18
```
  function hook(address sender, uint256 amountBeforeFee, uint256 amountAfterFee) external virtual override onlyAllowedMsgSenders {
    require(_accountList.isIncluded(sender), "redeemer not allowed");
----
  }
```

As you can see above, only users that are in the account list will be able to redeem. NFT holders will receive a revert of "redeemer not allowed"

### Hardhat POC

There is an already implemented test that tests if a user that `hook` will revert if the user not in the allowed list:
https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/test/RedeemHook.test.ts#L128
```
    it('reverts if caller not allowed', async () => {
      msgSendersAllowlist.isIncluded.returns(false)
      expect(await msgSendersAllowlist.isIncluded(user.address)).to.be.false

      await expect(redeemHook.connect(user).hook(user.address, 1, 1)).to.be.revertedWith(
        'msg.sender not allowed'
      )
    })
```
## Tools Used

VS Code, hardhat

## Recommended Mitigation Steps

Add an additional check like in `DepositHook` to NFT holders through `NFTScoreRequirement`.