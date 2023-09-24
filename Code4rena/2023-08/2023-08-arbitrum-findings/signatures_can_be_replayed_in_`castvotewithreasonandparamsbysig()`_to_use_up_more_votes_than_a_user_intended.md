## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- H-01

# [Signatures can be replayed in `castVoteWithReasonAndParamsBySig()` to use up more votes than a user intended](https://github.com/code-423n4/2023-08-arbitrum-findings/issues/252) 

# Lines of code

https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L480-L495


# Vulnerability details

## Bug Description

In the `SecurityCouncilNomineeElectionGovernor` and `SecurityCouncilMemberElectionGovernor` contracts, users can provide a signature to allow someone else to vote on their behalf using the `castVoteWithReasonAndParamsBySig()` function, which is in Openzeppelin's [`GovernorUpgradeable`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol):

[GovernorUpgradeable.sol#L480-L495](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L480-L495)

```solidity
        address voter = ECDSAUpgradeable.recover(
            _hashTypedDataV4(
                keccak256(
                    abi.encode(
                        EXTENDED_BALLOT_TYPEHASH,
                        proposalId,
                        support,
                        keccak256(bytes(reason)),
                        keccak256(params)
                    )
                )
            ),
            v,
            r,
            s
        );
```

As seen from above, the signature provided does not include a nonce. This becomes an issue in nominee and member elections, as users can choose not to use all of their votes in a single call, allowing them split their voting power amongst contenders/nominees:

[Nominee Election Specification](https://forum.arbitrum.foundation/t/proposal-security-council-elections-proposed-implementation-spec/15425#h-1-nominee-selection-7-days-10)

>  A single delegate can split their vote across multiple candidates.

[Member Election Specification](https://forum.arbitrum.foundation/t/proposal-security-council-elections-proposed-implementation-spec/15425#h-3-member-election-21-days-14)

> Additionally, delegates can cast votes for more than one nominee:
> * Split voting. delegates can split their tokens across multiple nominees, with 1 token representing 1 vote.

Due to the lack of a nonce, `castVoteWithReasonAndParamsBySig()` can be called multiple times with the same signature. 

Therefore, if a user provides a signature to use a portion of his votes, an attacker can repeatedly call `castVoteWithReasonAndParamsBySig()` with the same signature to use up more votes than the user originally intended.

## Impact

Due to the lack of signature replay protection in `castVoteWithReasonAndParamsBySig()`, during nominee or member elections, an attacker can force a voter to use more votes on a contender/nominee than intended by replaying his signature multiple times.

## Proof of Concept

Assume that a nominee election is currently ongoing:
* Bob has 1000 votes, he wants to split his votes between contender A and B:
  * He signs one signature to give 500 votes to contender A.
  * He signs a second signature to allocate 500 votes to contender B.
* `castVoteWithReasonAndParamsBySig()` is called to submit Bob's first signature:
  * This gives contender A 500 votes.
* After the transaction is executed, Alice sees Bob's signature in the transaction.
* As Alice wants contender A to be elected, she calls `castVoteWithReasonAndParamsBySig()` with Bob's first signature again:
  * Due to a lack of a nonce, the transaction is executed successfully, giving contender A another 500 votes.
* Now, when `castVoteWithReasonAndParamsBySig()` is called with Bob's second signature, it reverts as all his 1000 votes are already allocated to contender A.

In the scenario above, Alice has managed to allocate all of Bob's votes to contender A against his will. Note that this can also occur in member elections, where split voting is also allowed.

## Recommended Mitigation

Consider adding some form of signature replay protection in the `SecurityCouncilNomineeElectionGovernor` and `SecurityCouncilMemberElectionGovernor` contracts.

One way of achieving this is to override the `castVoteWithReasonAndParamsBySig()` function to include a nonce in the signature, which would protect against signature replay.





## Assessed type

Other