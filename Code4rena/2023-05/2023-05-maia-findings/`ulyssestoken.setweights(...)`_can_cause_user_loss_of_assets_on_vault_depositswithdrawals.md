## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-34

# [`UlyssesToken.setWeights(...)` can cause user loss of assets on vault deposits/withdrawals](https://github.com/code-423n4/2023-05-maia-findings/issues/281) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-4626/ERC4626MultiToken.sol#L200-L207
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-4626/ERC4626MultiToken.sol#L250-L256


# Vulnerability details

## Impact
The [ERC-4626](https://eips.ethereum.org/EIPS/eip-4626) paradigm of deposit/mint and withdraw/redeem, where a **single** underlying asset amount can always be converted to a number of vault shares and vice-versa, breaks as soon as there are **multiple weighted** underlying assets involved.  
While it's easy to convert from shares to asset amounts using the weight factors, it's hard to convert from asset amounts to shares in case they are not exactly distributed according to the weight factors.  

In the Ulysses protocol this was solved the following way:
* On `UlyssesToken.deposit(...)` every asset amount is converted to shares and the **smallest** of them is the one received for the deposit, see [ERC4626MultiToken.convertToShares(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-4626/ERC4626MultiToken.sol#L200-L207). As a consequence, excess assets provided on deposit are lost for the user and cannot be redeemed with the received shares.
* On `UlyssesToken.withdraw(...)` every asset amount is converted to shares and the **greatest** of them is the one required to withdraw the given asset amounts, see [ERC4626MultiToken.previewWithdraw(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-4626/ERC4626MultiToken.sol#L250-L255). As a consequence, less assets than entitled to according to the share count can be withdrawn from the vault incuring a loss for the user.  

One might argue that this issue is of low severity due to user error and the user being responsible to only use asset amounts in accordance with the vault's asset weights. However, the asset weights are not fixed and can be changed at any time by the ower of the `UlyssesToken` contract via the [setWeights(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesToken.sol#L88-L105) method. That is what makes this an actual issue.  
Consider the scenario when a user is about to deposit/withdraw assets not knowing their respective weights have changed, or even worse the deposit/withdraw transaction is already in the **mempool** but the call to `setWeights(...)` is executed before. Depending on the new asset weights, this will inevitably lead to a loss for the user.

## Proof of Concept

The following in-line documented PoC demonstrates the above claims for the deposit case. Just add the new test case below to `test/ulysses-amm/UlyssesTokenTest.t.sol:InvariantUlyssesToken` and run it with `forge test -vv --match-test test_UlyssesToken`.

```solidity
function test_UlyssesTokenSetWeightsDepositLoss() public {
    UlyssesToken token = UlyssesToken(_vault_);

    // initialize asset amounts according to weights, mint tokens & give approval to UlyssesToken vault
    uint256[] memory assetsAmounts = new uint256[](NUM_ASSETS);
    for (uint256 i = 0; i < NUM_ASSETS; i++) {
        assetsAmounts[i] = 1000 * token.weights(i);
        MockERC20(token.assets(i)).mint(address(this), 1e18);
        MockERC20(token.assets(i)).approve(address(token), 1e18);
    }

    // deposit assets & check if we got the expected number of shares
    uint256 expectedShares = token.previewDeposit(assetsAmounts);
    uint256 receivedShares = token.deposit(assetsAmounts, address(this));
    assertEq(expectedShares, receivedShares); // OK

    // check if we can redeem the same asset amounts as we deposited
    uint256[] memory redeemAssetsAmounts = token.previewRedeem(receivedShares);
    assertEq(assetsAmounts, redeemAssetsAmounts); // OK

    // assuming everything is fine, we submit another deposit transaction to the mempool
    // meanwhile the UlyssesToken owner changes the asset weights
    uint256[] memory weights = new uint256[](NUM_ASSETS);
    for (uint256 i = 0; i < NUM_ASSETS; i++) {
        weights[i] = token.weights(i);
    }
    weights[0] *= 2; // double the weight of first asset
    token.setWeights(weights);

    // now our deposit transaction gets executed, but due to the changed asset weights
    // we got less shares than expected while sending too many assets (except for asset[0])
    receivedShares = token.deposit(assetsAmounts, address(this));
    assertEq(expectedShares, receivedShares, "got less shares than expected");

    // due to the reduced share amount we cannot redeem all the assets we deposited,
    // we lost the excess assets we have deposited (except for asset[0])
    redeemAssetsAmounts = token.previewRedeem(receivedShares);
    assertEq(assetsAmounts, redeemAssetsAmounts, "can redeem less assets than deposited");
}
```

The test case shows that less shares than expected are received in case of changed weights and any deposited excess assets cannot be redeemed anymore:
```
Running 1 test for test/ulysses-amm/UlyssesTokenTest.t.sol:InvariantUlyssesToken
[FAIL. Reason: Assertion failed.] test_UlyssesTokenSetWeightsDepositLoss() (gas: 631678)
Logs:
  Error: got less shares than expected
  Error: a == b not satisfied [uint]
        Left: 45000
       Right: 27500
  Error: can redeem less assets than deposited
  Error: a == b not satisfied [uint[]]
        Left: [10000, 10000, 20000, 5000]
       Right: [10000, 5000, 10000, 2500]
```

*For the sake of simplicity, the test for the withdrawal case was skipped since it's exactly the same problem just in the reverse direction.*

## Tools Used
VS Code, Foundry

## Recommended Mitigation Steps
* On `UlyssesToken.deposit(...)`, only transfer the necessary token amounts (according to the computed share count) from the sender, like the `UlyssesToken.mint(...)` method does.
* On `UlyssesToken.withdraw(...)`, transfer all the asset amounts the sender is entitled to (according to the computed share count) to the receiver, like the `UlyssesToken.redeem(...)` method does.



## Assessed type

Rug-Pull