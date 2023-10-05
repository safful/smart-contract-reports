## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-07

# [`fastTrackProposalExecution` doesn't check `intendedRecipient`](https://github.com/code-423n4/2023-07-moonwell-findings/issues/308) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Governance/TemporalGovernor.sol#L261-L268


# Vulnerability details

## Description
Wormhole cross chain communication [is multicasted](https://docs.wormhole.com/wormhole/explore-wormhole/core-contracts#multicast) to all their supported chains:

> Please notice that there is no destination address or chain in these functions.
> VAAs simply attest that "this contract on this chain said this thing." Therefore, VAAs are multicast by default and will be verified as authentic on any chain they are brought to.

The `TermporalGovernor` contract handles this by checking `queueProposal` that the `intendedRecipient` is itself:

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Governance/TemporalGovernor.sol#L320-L322
```solidity
File: core/Governance/TemporalGovernor.sol

320:        // Very important to check to make sure that the VAA we're processing is specifically designed
321:        // to be sent to this contract
322:        require(intendedRecipient == address(this), "TemporalGovernor: Incorrect destination");
```

There is also a `fastTrackProposalExecution` that the guardian can perform (`owner` is referred ot as `guardian`):

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Governance/TemporalGovernor.sol#L261-L268
```solidity
File: core/Governance/TemporalGovernor.sol

261:    /// @notice Allow the guardian to process a VAA when the
262:    /// Temporal Governor is paused this is only for use during
263:    /// periods of emergency when the governance on moonbeam is
264:    /// compromised and we need to stop additional proposals from going through.
265:    /// @param VAA The signed Verified Action Approval to process
266:    function fastTrackProposalExecution(bytes memory VAA) external onlyOwner {
267:        _executeProposal(VAA, true); /// override timestamp checks and execute
268:    }
```

This will bypass the `queueProposal` flow and execute commands immediately.

The issue here is that there is no check in `_executeProposal` that the `intendedRecipient` is this `TemporalGovernor`.

## Impact
`governor` can execute any commands communicated across `Wormhole` as long as they are from a trusted source.

This requires that the `targets` line up between chains for a `governor` to abuse this. However only one `target` must line up as the `.call` made [does not check for contract existence](https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Governance/TemporalGovernor.sol#L395-L402). Since "unaligned" addresses between chains will most likely not exist on other chains these calls will succeeed.

## Proof of Concept
Test in `TemporalGovernorExec.t.sol`, all of the test is copied from `testProposeFailsIncorrectDestination` with just the change at the end where instead of calling `queueProposal`, `fastTrackProposalExecution` is called instead, which doesn't revert, thus highligting the issue.

```solidity
    function testFasttrackExecuteSucceedsIncorrectDestination() public {
        address[] memory targets = new address[](1);
        targets[0] = address(governor);

        uint256[] memory values = new uint256[](1);
        values[0] = 0;

        TemporalGovernor.TrustedSender[]
            memory trustedSenders = new TemporalGovernor.TrustedSender[](1);

        trustedSenders[0] = ITemporalGovernor.TrustedSender({
            chainId: trustedChainid,
            addr: newAdmin
        });

        bytes[] memory payloads = new bytes[](1);

        payloads[0] = abi.encodeWithSignature( /// if issues use encode with selector
            "setTrustedSenders((uint16,address)[])",
            trustedSenders
        );

        /// wrong intendedRecipient
        bytes memory payload = abi.encode(newAdmin, targets, values, payloads);

        mockCore.setStorage(
            true,
            trustedChainid,
            governor.addressToBytes(admin),
            "reeeeeee",
            payload
        );

        // guardian can fast track execute calls intended for other contracts
        governor.fastTrackProposalExecution("");
    }
```

## Tools Used
Manual audit

## Recommended Mitigation Steps
Consider adding a check in `_executeProposal` for `intendedRecipient` same as is done in `_queueProposal`.


## Assessed type

Invalid Validation