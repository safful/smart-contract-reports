## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- sponsor disputed
- M-22

# [Unreleased locks cause the reward distribution to be flawed in BondNFT](https://github.com/code-423n4/2022-12-tigris-findings/issues/630) 

# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/BondNFT.sol#L150
https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/BondNFT.sol#L225


# Vulnerability details

## Impact
After a lock has expired, it doesn't get any rewards distributed to it. But, unreleased locks cause other existing bonds to not receive the full amount of tokens either. The issue is that as long as the bond is not released, the `totalShares` value isn't updated. Everybody receives a smaller cut of the distribution. Thus, bond owners receive less rewards than they should.

A bond can be released after it expired by the owner of it. If the owner doesn't release it for 7 days, anybody else can release it as well. As long as the owner doesn't release it, the issue will be in effect for at least 7 epochs.

Since this causes a loss of funds for every bond holder I rate it as HIGH. It's likely to be an issue since you can't guarantee that bonds will be released the day they expire.

## Proof of Concept
Here's a test showcasing the issue:

```js
// 09.Bonds.js

    it.only("test", async function () {
      await stabletoken.connect(owner).mintFor(owner.address, ethers.utils.parseEther("100"));
      await lock.connect(owner).lock(StableToken.address, ethers.utils.parseEther("100"), 100);
      await stabletoken.connect(owner).mintFor(user.address, ethers.utils.parseEther("1000"));
      await lock.connect(user).lock(StableToken.address, ethers.utils.parseEther("1000"), 10);
      await stabletoken.connect(owner).mintFor(owner.address, ethers.utils.parseEther("1000"));
      await bond.distribute(stabletoken.address, ethers.utils.parseEther("1000"));

      await network.provider.send("evm_increaseTime", [864000]); // Skip 10 days
      await network.provider.send("evm_mine");

      [,,,,,,,pending,,,] = await bond.idToBond(1);
      expect(pending).to.be.equals("499999999999999999986");
      [,,,,,,,pending,,,] = await bond.idToBond(2);
      expect(pending).to.be.equals("499999999999999999986");


      await stabletoken.connect(owner).mintFor(owner.address, ethers.utils.parseEther("1000"));
      await bond.distribute(stabletoken.address, ethers.utils.parseEther("1000"));

      await network.provider.send("evm_increaseTime", [86400 * 3]); // Skip 3 days
      await network.provider.send("evm_mine");

      // Bond 2 expired, so it doesn't receive any of the new tokens that were distributed
      [,,,,,,,pending,,,] = await bond.idToBond(2);
      expect(pending).to.be.equals("499999999999999999986");

      // Thus, Bond 1 should get all the tokens, increasing its pending value to 1499999999999999999960
      // But, because bond 2 wasn't released (`totalShares` wasn't updated), bond 1 receives less tokens than it should.
      // Thus, the following check below fails
      [,,,,,,,pending,,,] = await bond.idToBond(1);
      expect(pending).to.be.equals("1499999999999999999960");

      await lock.connect(user).release(2);

      expect(await stabletoken.balanceOf(user.address)).to.be.equals("1499999999999999999986");

    });
```

The `totalShares` value is only updated after a lock is released:

```sol
    function release(
        uint _id,
        address _releaser
    ) external onlyManager() returns(uint amount, uint lockAmount, address asset, address _owner) {
        Bond memory bond = idToBond(_id);
        require(bond.expired, "!expire");
        if (_releaser != bond.owner) {
            unchecked {
                require(bond.expireEpoch + 7 < epoch[bond.asset], "Bond owner priority");
            }
        }
        amount = bond.amount;
        unchecked {
            totalShares[bond.asset] -= bond.shares;
        // ... 
```
## Tools Used
none

## Recommended Mitigation Steps
Only shares belonging to an active bond should be used for the distribution logic.