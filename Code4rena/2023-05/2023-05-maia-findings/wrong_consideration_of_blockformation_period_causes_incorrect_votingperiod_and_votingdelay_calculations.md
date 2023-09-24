## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-25

# [Wrong consideration of blockformation period causes incorrect votingPeriod and votingDelay calculations](https://github.com/code-423n4/2023-05-maia-findings/issues/417) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L18-L27
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L68-L72
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L397-L423


# Vulnerability details

## Impact
In GovernorBravoDelegateMaias.sol contract, There are wrong calculation in [MIN_VOTING_PERIOD](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L18),[MAX_VOTING_PERIOD](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L21C29-L21C46), [MIN_VOTING_DELAY](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L24), [MAX_VOTING_DELAY](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L27) because of incorrect consideration of block formation period. 

The contracts will be deployed on Ethereum mainnet Chain too and in a Ethereum mainnet chain, the blocks are made every 12 seconds but the votingPeriod and votingDelay variables has used 15 seconds while calculating their values.

**For example:**

MIN_VOTING_PERIOD is considered for 2 weeks.

```Solidity
   uint256 public constant MIN_VOTING_PERIOD = 80640; // About 2 weeks
```

2 weeks(in seconds) = 12,09,600
Considered Ethereum blockformation time in second = 15

Therefore, MAX_VOTING_PERIOD = 12,09,600 / 15 = 80,640 (blocks). 

This is how the calculations have arrived for other votingPeriod and votingDelay state variables.

However, Ethereum block formation happens on every 12 seconds and it is confirmed from below sources, 
[Reference-01](https://chainstack.com/protocols/ethereum/#Performance)
[Reference-02](https://ycharts.com/indicators/ethereum_average_block_time)

**The correct calculation should be with 12 seconds as block formation period.**

**For example:**
2 weeks(in seconds) = 12,09,600
Actual Ethereum blockformation time in second = 12

Therefore, MAX_VOTING_PERIOD = 12,09,600 / 12 = 100,800 (blocks). 

Total number of block difference for 2 week duration = 100,800 - 80,640 = 20,160 ~ 5.6 hours
This much time difference will affect the function validations which will cause unexpected design failure.

[MIN_VOTING_PERIOD](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L18),[MAX_VOTING_PERIOD](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L21C29-L21C46), [MIN_VOTING_DELAY](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L24), [MAX_VOTING_DELAY](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L27) are used in functions which are further explained as below,

```Solidity
File: src/governance/GovernorBravoDelegateMaia.sol

56    function initialize(
57        address timelock_,
58        address govToken_,
59        uint256 votingPeriod_,
60        uint256 votingDelay_,
61        uint256 proposalThreshold_
62    ) public virtual {
63        require(address(timelock) == address(0), "GovernorBravo::initialize: can only initialize once");
64        require(msg.sender == admin, "GovernorBravo::initialize: admin only");
65        require(timelock_ != address(0), "GovernorBravo::initialize: invalid timelock address");
66        require(govToken_ != address(0), "GovernorBravo::initialize: invalid govToken address");
67        require(
68            votingPeriod_ >= MIN_VOTING_PERIOD && votingPeriod_ <= MAX_VOTING_PERIOD,
69            "GovernorBravo::initialize: invalid voting period"
70        );
71        require(
72            votingDelay_ >= MIN_VOTING_DELAY && votingDelay_ <= MAX_VOTING_DELAY,
73            "GovernorBravo::initialize: invalid voting delay"
74        );

         // some code
```
At L-68 and L-72, these state variables are used to validate the conditions in initialize() function which can be called only once. These incorrect values makes the conditions at L-68 and L-72 obsolete and the conditions wont work as expected by design.

Further [MIN_VOTING_PERIOD](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L18),[MAX_VOTING_PERIOD](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L21C29-L21C46), [MIN_VOTING_DELAY](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L24), [MAX_VOTING_DELAY](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L27) these state variables are used in below setter functions which for sure wont work as per expected design.

```Solidity
397  function _setVotingDelay(uint256 newVotingDelay) external {


413   function _setVotingPeriod(uint256 newVotingPeriod) external {
```

#### Discussion with Sponsors
I had a discussion with sponsor(@0xbuzzlightyear) on this finding and the sponsor has confirmed the issue. Below is the discord discussion with sponsor for reference and finding confirmation only,


**Mohammed Rizwan — 06/30/2023 at 5:00 PM**

    uint256 public constant MIN_VOTING_PERIOD = 80640; // About 2 weeks

Here, it is considered 15 sec for block formation considering Ethereum chain.
On ethereum, the average block formation time is 12 sec.
Reference- https://ycharts.com/indicators/ethereum_average_block_time
Ethereum Average Block Time
In depth view into Ethereum Average Block Time including historical data from 2015 to 2023, charts and stats.

**0xbuzzlightyear — 06/30/2023 at 5:06 PM**
true, nice finding, we did that before the merge :pepepalm:

**Mohammed Rizwan — 06/30/2023 at 5:07 PM**
Thank you.
If you allow me, can i add this discussion in report?

**0xbuzzlightyear — 06/30/2023 at 5:09 PM**
yes, of course. you are free to do what you wish fren!

**Mohammed Rizwan — 06/30/2023 at 5:09 PM**
Thanks again!!!

**0xbuzzlightyear — 06/30/2023 at 5:11 PM**
you're welcome! :pepoloveleaf:


## Proof of Concept
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L18-L27

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L68-L72

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L397-L423

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider 12 seconds block formation period and correct the calculations.


## Assessed type

Other