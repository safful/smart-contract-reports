## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- judge review requested
- selected for report
- sponsor acknowledged
- M-14

# [`BondNFT.sol#claim()` needs to correct all the missing epochs](https://github.com/code-423n4/2022-12-tigris-findings/issues/392) 

# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/BondNFT.sol#L177-L183
https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/BondNFT.sol#L235-L242


# Vulnerability details

## Impact

In `BondNFT.sol#claim()`, `accRewardsPerShare[][]` is amended to reflect the expired shares. But only `accRewardsPerShare[bond.asset][epoch[bond.asset]]` is updated. All the epochs between `bond.expireEpoch-1` and `epoch[bond.asset]` are missed. However, some users claimable rewards calculation could be based on the missed epochs. As a result, the impact might be:
- `accRewardsPerShare` be inaccurate for the epochs in between.
- some users could lose reward due to wrong `accRewardsPerShare`, some users might receive undeserved rewards.
- some rewards will be locked in the contract.


## Proof of Concept

The rationale behind the unchecked block below seems to take into account the shares of reward of the expired bond. However, if only update the latest epoch data, the epochs in between could have errors and lead to loss of other users.

```solidity
File: contracts/BondNFT.sol
168:     function claim(
169:         uint _id,
170:         address _claimer
171:     ) public onlyManager() returns(uint amount, address tigAsset) {
    
177:             if (bond.expired) {
178:                 uint _pendingDelta = (bond.shares * accRewardsPerShare[bond.asset][epoch[bond.asset]] / 1e18 - bondPaid[_id][bond.asset]) - (bond.shares * accRewardsPerShare[bond.asset][bond.expireEpoch-1] / 1e18 - bondPaid[_id][bond.asset]);
179:                 if (totalShares[bond.asset] > 0) {
180:                     accRewardsPerShare[bond.asset][epoch[bond.asset]] += _pendingDelta*1e18/totalShares[bond.asset];
181:                 }
182:             }
183:             bondPaid[_id][bond.asset] += amount;
```

Users can claim rewards up to the expiry time, based on `accRewardsPerShare[tigAsset][bond.expireEpoch-1]`:
```solidity
235:     function idToBond(uint256 _id) public view returns (Bond memory bond) {
    
238:         bond.expired = bond.expireEpoch <= epoch[bond.asset] ? true : false;
239:         unchecked {
240:             uint _accRewardsPerShare = accRewardsPerShare[bond.asset][bond.expired ? bond.expireEpoch-1 : epoch[bond.asset]];
241:             bond.pending = bond.shares * _accRewardsPerShare / 1e18 - bondPaid[_id][bond.asset];

```
