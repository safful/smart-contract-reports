## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-02

# [SecurityCouncilNomineeElectionGovernor might have to wait for more than 6 months to create election again](https://github.com/code-423n4/2023-08-arbitrum-findings/issues/182) 

# Lines of code

https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilNomineeElectionGovernorTiming.sol#L75-#L94


# Vulnerability details

## Impact
- SecurityCouncilNomineeElectionGovernor might have to wait for more than 6 months to create election again

## Proof of Concept
According to the document https://forum.arbitrum.foundation/t/proposal-security-council-elections-proposed-implementation-spec/15425/1#h-1-nominee-selection-7-days-10, security council election can be create every 6 months. Contract `SecurityCouncilNomineeElectionGovernor` implements this by these codes:
```solidity
 function createElection() external returns (uint256 proposalId) {
        // require that the last member election has executed
        _requireLastMemberElectionHasExecuted();

        // each election has a deterministic start time
        uint256 thisElectionStartTs = electionToTimestamp(electionCount);
        if (block.timestamp < thisElectionStartTs) {
            revert CreateTooEarly(block.timestamp, thisElectionStartTs);
        }
        ...
}

    function electionToTimestamp(uint256 electionIndex) public view returns (uint256) {
        // subtract one to make month 0 indexed
        uint256 month = firstNominationStartDate.month - 1;

        month += 6 * electionIndex;
        uint256 year = firstNominationStartDate.year + month / 12;
        month = month % 12;

        // add one to make month 1 indexed
        month += 1;

        return DateTimeLib.dateTimeToTimestamp({
            year: year,
            month: month,
            day: firstNominationStartDate.day,
            hour: firstNominationStartDate.hour,
            minute: 0,
            second: 0
        });
    }

```
If `electionIndex` = 1, function `createElection` will call `electionToTimestamp` to calculate for timestamp 6 months from `firstNominationStartDate`. However, the code uses `firstNominationStartDate.day` to form the result day:
```solidity
        return DateTimeLib.dateTimeToTimestamp({
            year: year,
            month: month,
            day: firstNominationStartDate.day,
            hour: firstNominationStartDate.hour,
            minute: 0,
            second: 0
        });
```
This could result in wrong calculation because the day in months can varies from 28-31. Therefore, the worst case is that the user has to wait for 6 months + 4 more days to create new election. 

For example, if `firstNominationStartDate` = 2024-08-31-01:00:00 (which is the last day of August). The user might expect that they can create election again 6 months from that, which mean 1:00 AM of the last day of February 2025 (which is 2025-02-28-01:00:00), but in fact the  result of `electionToTimestamp` would be 2025-03-03-01:00:00, 4 days from that.

Below is POC for the above example, for easy of testing, place this test case in file `test/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.t.sol` under contract `SecurityCouncilNomineeElectionGovernorTest` and run it using command:
`forge test --match-path test/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.t.sol --match-test testDateTime -vvvv`

```solidity
    function testDateTime() public {
        // Deploy a new governor
        // with first nomination start date = 2024-08-30T01:00
        SecurityCouncilNomineeElectionGovernor  newGovernor = _deployGovernor();

        SecurityCouncilNomineeElectionGovernor.InitParams memory newInitParams
        =	SecurityCouncilNomineeElectionGovernor.InitParams({
            firstNominationStartDate: Date({year: 2024, month: 8, day:31, hour:1}),
            nomineeVettingDuration: 1 days,
            nomineeVetter: address(0x11),
            securityCouncilManager: ISecurityCouncilManager(address(0x22)),
            securityCouncilMemberElectionGovernor: ISecurityCouncilMemberElectionGovernor(
                payable(address(0x33))
            ),
            token: IVotesUpgradeable(address(0x44)),
            owner: address(0x55),
            quorumNumeratorValue: 20,
            votingPeriod: 1 days
        });

        // The next selection is not available until timestamp 1740963600,
        // which is 2025-03-03T1:00:00 AM GMT
        newGovernor.initialize(newInitParams);
        assertEq(newGovernor.electionToTimestamp(1), 1740963600);

    }
```
You can use an online tool like https://www.epochconverter.com/ to check that `1740963600` is ` Monday, March 3, 2025 1:00:00 AM GMT`


## Tools Used
Manual review

## Recommended Mitigation Steps
I recommend fixing the math so that the duration between elections are exactly 6 months like documented.





## Assessed type

Math