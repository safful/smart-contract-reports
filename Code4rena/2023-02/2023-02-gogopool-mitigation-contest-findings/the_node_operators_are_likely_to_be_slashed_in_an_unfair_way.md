## Tags

- 2 (Med Risk)
- downgraded by judge
- selected for report
- MR-H-04

# [The node operators are likely to be slashed in an unfair way](https://github.com/code-423n4/2023-02-gogopool-mitigation-contest-findings/issues/23) 

# Lines of code

https://github.com/multisig-labs/gogopool/blob/4bcef8b1d4e595c9ba41a091b2ebf1b45858f022/contracts/contract/MinipoolManager.sol#L464


# Vulnerability details

# C4 issue

H-04: [Hijacking of node operators minipool causes loss of staked funds](https://github.com/code-423n4/2022-12-gogopool-findings/issues/213)

# Comments
In the original implementation, the protocol had some unnecessary state transitions and it was possible for node operators to interfere the recreation process.
The main problem was the `recordStakingEnd()` and `recreateMiniPool()` were separate external functions and the operator could frontrun the `recreateMiniPool()` and call `withdrawMinipoolFunds()`.

# Mitigation
[PR #23](https://github.com/multisig-labs/gogopool/pull/23)
The mitigation added a new function `recordStakingEndThenMaybeCycle()` and handled `recordStakingEnd()` and `recreateMiniPool()` in an atomic way.
With this mitigation, the state flow is now as below and it is impossible for a node operator to interfere the recreation process.
![Imgur](https://imgur.com/JCoiCvl.jpg)
But this mitigation created another minor issue that the node operators have risks to be slashed in an unfair way.

# New issue
The node operators are likely to be slashed in an unfair way

# Code snippet
https://github.com/multisig-labs/gogopool/blob/4bcef8b1d4e595c9ba41a091b2ebf1b45858f022/contracts/contract/MinipoolManager.sol#L464

# Proof of concept
In the previous implementation, I assumed rialtos are smart enough to recreate minipools only when it's necessary.
But now, the recreation process is included as an optional way in the `recordStakingEndThenMaybeCycle()`, so as long as the check `initialStartTime + duration > block.timestamp` at L#464 passes, recreation will be processed.

Now let us consider the timeline. One validation cycle in the whole sense contains several steps as below.
![Imgur](https://imgur.com/p6xWqgC.jpg)

1) Let us assume it is somehow possible that `startTime[1] > endTime[0]`, i.e., the multisig failed to start the next cycle at the exact the same timestamp to the previous end time. This is quite possible due to various reasons because there are external processes included.
In this case the timeline will look as below.
![Imgur](https://imgur.com/e292GIO.jpg)
As an extreme example, let us say the node operator created a minipool with duration of 42 days (with 3 cycles in mind) and it took 12 days to start the second cycle. When the `recordStakingEndThenMaybeCycle()` (finishing the second cycle) was called, two cases are possible.
- It is possible that the `initialStartTime + duration <= block.timestamp`. In this case, the protocol will not start the next cycle. And the node validation was done for two cycles different to the initial plan.
- If `initialStartTime + duration > block.timestamp`, the protocol will start the third cycle. But on the end of that cycle, it is likely that the node is not eligible for reward by the Avalanche validators voting. (Imagine the node op lent a server for 42 days, then 42-14*2-12=2 days from the third cycle start the node might have stopped working and does not meet the 80% uptime condition) Then the node operator will be punished and GGP stake will be slashed. This is unfair.

2) Assume it is 100% guaranteed that `startTime[n+1]=endTime[n]` for all cycles.
The timeline will look as below and we can say the second case of the above scenario still exists if the node operator didn't specify the duration to be a complete multiple of 14 days. (365 days is not!)
![Imgur](https://imgur.com/tGHnMTL.jpg)
Then the last cycle end will be later than `initialStartTime + duration` and the node op can be slashed in an unfair way again.
So even assuming the perfect condition, the protocol works in kind of unfair way for node operators.

The main reason of this problem is that technically there exists two timelines. And the protocol does not track the actual validation duration that the node was used accurately.
At least, the protocol should not start a new cycle if `initialStartTime + duration < block.timestamp + 14 days` because it is likely that the node operator get punished at the end of that cycle.

# Tools used
Manual review

# Recommended additional mitigation
- If it is 100% guaranteed that `startTime[n+1]=endTime[n]` for all cycles, I recommend starting a new cycle only if `initialStartTime + duration < block.timestamp + 14 days`.
- If not, I suggest adding a new state variable that will track the actual validation period (actual utilized period).

# Conclusion
Mitigation error - created another issue for the same edge case.

