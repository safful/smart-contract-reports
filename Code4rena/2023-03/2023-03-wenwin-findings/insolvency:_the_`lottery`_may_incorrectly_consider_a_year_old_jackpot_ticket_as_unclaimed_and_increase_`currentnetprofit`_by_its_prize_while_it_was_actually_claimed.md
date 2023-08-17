## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-06

# [Insolvency: The `Lottery` may incorrectly consider a year old jackpot ticket as unclaimed and increase `currentNetProfit` by its prize while it was actually claimed](https://github.com/code-423n4/2023-03-wenwin-findings/issues/167) 

# Lines of code

https://github.com/code-423n4/2023-03-wenwin/blob/main/src/Lottery.sol#L135-L137
https://github.com/code-423n4/2023-03-wenwin/blob/main/src/Lottery.sol#L164-L166


# Vulnerability details

## Impact
According to the [documentation](https://docs.wenwin.com/wenwin-lottery/the-game):
> "Winning tickets have up to 1 year to claim their prize. If a prize is not claimed before this period, the unclaimed prize money will go back to the Prize Pot."

As part of this mechanism, the `Lottery.executeDraw()` function (which internally calls `Lottery.returnUnclaimedJackpotToThePot()`) increases `Lottery.currentNetProfit` by the prize of any 1 year old unclaimed jackpot ticket. At this point, the contract thinks it still hold that prize and it's going to include it in the prize pot of the next draw. This function can only be called when `block.timestamp >= drawScheduledAt(currentDraw)`.

On the other side, the `Lottery.claimWinningTickets()` function (which internally calls `Lottery.claimWinningTicket()` and `Lottery.claimable()`) sends the prize to the owner of a jackpot ticket only if it isn't 1 year old (if `block.timestamp <= ticketRegistrationDeadline(ticketInfo.drawId + LotteryMath.DRAWS_PER_YEAR)`).

However, if `Lottery.drawCoolDownPeriod` is zero, both of these condition can pass on the same time - when the `block.timestamp` is exactly the scheduled draw date of the draw that takes place exactly 1 year after the draw in which the jackpot ticket has won.

In this edge case, the owner of the jackpot ticket can call `Lottery.executeDraw()`, letting the `Lottery` think the prize wasn't claimed, followed by `Lottery.claimWinningTickets()`, claiming the prize, all in the same transaction.

## Proof of Concept
Let's say Eve buys a ticket to draw #1 in a `Lottery` contract where `Lottery.drawCoolDownPeriod` equals zero, and win the jackpot. Now, Eve can wait 1 year and then, when `block.timestamp` equals the scheduled draw date of draw #53 (which is also the registration deadline of draw #53), run a transaction that will do the following:
```
lottery.executeDraw()
lottery.claimWinningTickets([eveJackpotTicketId])
```
This will send Eve her prize, but will also leave the contract insolvent.

## Tools Used
Manual code review.

## Recommended Mitigation Steps
Fix `Lottery.claimable()` to set `claimableAmount` to `winAmount[ticketInfo.drawId][winTier]` only if `block.timestamp < ticketRegistrationDeadline(ticketInfo.drawId + LotteryMath.DRAWS_PER_YEAR)` (strictly less than).