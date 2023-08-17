## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- H-04

# [transfer position ownership to `addr(0)` to DoS `end()` challenge](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/670) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/MintingHub.sol#L268


# Vulnerability details

## Impact

If some challenge is about to succeed, the position owner will lose the collateral. Seeing the unavoidable loss, the owner can transfer the position ownership to `addr(0)`, fail the `end()` call of the challenge. At the end, the DoS in `end()` will have these impacts:
- the successful bidder will lose bid fund. 
- the challenger's collateral will be locked, and lose the challenge reward.


## Proof of Concept

Assuming, the position has `minimumCollateral` of 600 zchf, the position owner minted 1,000 zchf against some collateral worth of 1,100 zchf, the highest bid for the collateral was 1,060 zchf, the challenge reward being 50. Then in `Position.sol#notifyChallengeSucceeded()`, the `repayment` will be 1,000, but `effectiveBid` worth of 1,060. The `fundNeeded` will be 1,000 + 50 = 1,050, and results in excess of 1,060 - 1,050 = 10 to refund the position owner in line 268 `MintingHub.sol`. In addition, due to the `minimumCollateral` limit, this challenge cannot be split into smaller ones.

```solidity
File: contracts/MintingHub.sol
252:     function end(uint256 _challengeNumber, bool postponeCollateralReturn) public {

260:         (address owner, uint256 effectiveBid, uint256 volume, uint256 repayment, uint32 reservePPM) = challenge.position.notifyChallengeSucceeded(recipient, challenge.bid, challenge.size);
261:         if (effectiveBid < challenge.bid) {
262:             // overbid, return excess amount
263:             IERC20(zchf).transfer(challenge.bidder, challenge.bid - effectiveBid);
264:         }
265:         uint256 reward = (volume * CHALLENGER_REWARD) / 1000_000;
266:         uint256 fundsNeeded = reward + repayment;
267:         if (effectiveBid > fundsNeeded){
268:             zchf.transfer(owner, effectiveBid - fundsNeeded);

File: contracts/Position.sol
329:     function notifyChallengeSucceeded(address _bidder, uint256 _bid, uint256 _size) external onlyHub returns (address, uint256, uint256, uint256, uint32) {

349:         uint256 repayment = minted < volumeZCHF ? minted : volumeZCHF; // how much must be burned to make things even
350: 
351:         notifyRepaidInternal(repayment); // we assume the caller takes care of the actual repayment
352:         internalWithdrawCollateral(_bidder, _size); // transfer collateral to the bidder and emit update
353:         return (owner, _bid, volumeZCHF, repayment, reserveContribution);
354:     }
```

From the position owner's point of view, the position is on auction and has incurred loss already, only 10 zchf refund left. The owner can give up the tiny amount, and transfer the ownership to `addr(0)` to DoS the `end()` call.

When the position `owner` is `addr(0)`, the transfer in line 268 `MintingHub.sol` will revert, due to the requirement in zchf (inherited from `ERC20.sol`):
```solidity
File: contracts/ERC20.sol
151:     function _transfer(address sender, address recipient, uint256 amount) internal virtual {
152:         require(recipient != address(0));
```

Now the successful bidder can no longer call `end()`. The bid fund will be lost. Also the challenger will lose the collateral because the return call encounter DoS too.


## Tools Used
Manual analysis.

## Recommended Mitigation Steps

Disallow transferring position ownership to `addr(0)`

