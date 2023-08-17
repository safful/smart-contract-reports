## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-09

# [MuteBond.sol: When maxPayout is lowered the contract can end up DOSed](https://github.com/code-423n4/2023-03-mute-findings/issues/14) 

# Lines of code

https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L119-L123
https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L153-L200


# Vulnerability details

## Impact
The [`maxPayout`](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L34) variable in the `MuteBond` contract specifies the amount of `MUTE` that is paid out in one epoch before the next epoch is entered.  

The variable is initialized in the constructor and can then be changed via the [`setMaxPayout`](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L119-L123) function.  

The issue occurs when `maxPayout` is lowered.  

So say `maxPayout` is currently `10,000 MUTE` and the `owner` wants to reduce it to `5,000 MUTE`.  

Before this transaction to lower `maxPayout` is executed, another transaction might be executed which increases the current payout to `> 5,000 MUTE`.  

This means that calls to [`MuteBond.deposit`](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L153-L200) revert and no new epoch can be entered. Thereby the `MuteBond` contracts becomes unable to provide bonds.  

The DOS is not permanent. The `owner` can increase `maxPayout` such that the current payout is smaller than `maxPayout` again and the contract will work as intended.  

So the impact is a temporary DOS of the `MuteBond` contract.  

The issue can be solved by requiring in the `setMaxPayout` function that `maxPayout` must be bigger than the payout in the current epoch.  

## Proof of Concept
Add the following test to the `bonds.ts` test file:  

```javascript
it('POC maxPayout below current payout causes DOS', async function () {
    // owner wants to set maxPayout to 9 * 10**18 
    // However a transaction is executed first that puts the payout in the current epoch above that value
    // all further deposits revert

    // make a payout
    await bondContract.connect(owner).deposit(new BigNumber(10).times(Math.pow(10,18)).toFixed(), owner.address, false)
    
    // set maxPayout below currentPayout
    await bondContract.connect(owner).setMaxPayout(new BigNumber(9).times(Math.pow(10,18)).toFixed())

    // deposit reverts due to underflow
    await bondContract.connect(owner).deposit("0", owner.address, true)
  })
```

## Tools Used
VSCode

## Recommended Mitigation Steps
I recommend that the `setMaxPayout` function checks that `maxPayout` is set to a value bigger than the payout in the current epoch:  

```diff
diff --git a/contracts/bonds/MuteBond.sol b/contracts/bonds/MuteBond.sol
index 96ee755..4af01d7 100644
--- a/contracts/bonds/MuteBond.sol
+++ b/contracts/bonds/MuteBond.sol
@@ -118,6 +118,7 @@ contract MuteBond {
      */
     function setMaxPayout(uint _payout) external {
         require(msg.sender == customTreasury.owner());
+        require(_payout > terms[epoch].payoutTotal, "_payout too small");
         maxPayout = _payout;
         emit MaxPayoutChanged(_payout);
     }
```
