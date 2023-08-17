## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-02

# [If DAO updates forkEscrow before forkThreshold is reached, the user's escrowed Nouns will be lost](https://github.com/code-423n4/2023-07-nounsdao-findings/issues/56) 

# Lines of code

https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/fork/NounsDAOV3Fork.sol#L95-L102


# Vulnerability details

## Impact
During the escrow period, users can escrow to or withdraw from forkEscrow their Nouns. 
During the escrow period, proposals can be executed.
```solidity
    function withdrawFromForkEscrow(NounsDAOStorageV3.StorageV3 storage ds, uint256[] calldata tokenIds) external {
        if (isForkPeriodActive(ds)) revert ForkPeriodActive();

        INounsDAOForkEscrow forkEscrow = ds.forkEscrow;
        forkEscrow.returnTokensToOwner(msg.sender, tokenIds);

        emit WithdrawFromForkEscrow(forkEscrow.forkId(), msg.sender, tokenIds);
    }
```
Since withdrawFromForkEscrow will only call the returnTokensToOwner function of ds.forkEscrow, and returnTokensToOwner is only allowed to be called by DAO.
If, during the escrow period, ds.forkEscrow is changed by the proposal's call to _setForkEscrow, then the user's escrowed Nouns will not be withdrawn by withdrawFromForkEscrow.
```solidity
    function returnTokensToOwner(address owner, uint256[] calldata tokenIds) external onlyDAO {
        for (uint256 i = 0; i < tokenIds.length; i++) {
            if (currentOwnerOf(tokenIds[i]) != owner) revert NotOwner();

            nounsToken.transferFrom(address(this), owner, tokenIds[i]);
            escrowedTokensByForkId[forkId][tokenIds[i]] = address(0);
        }

        numTokensInEscrow -= tokenIds.length;
    }
```
Consider that some Nouners is voting on a proposal that would change ds.forkEscrow.
There are some escrowed Nouns in forkEscrow (some Nouners may choose to always escrow their Nouns to avoid missing fork).
The proposal is executed, ds.forkEscrow is updated, and the escrowed Nouns cannot be withdrawn.
## Proof of Concept
https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/fork/NounsDAOV3Fork.sol#L95-L102
https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/fork/NounsDAOForkEscrow.sol#L116-L125
https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/NounsDAOV3Admin.sol#L527-L531
## Tools Used
None
## Recommended Mitigation Steps
Consider allowing the user to call forkEscrow.returnTokensToOwner directly to withdraw escrowed Nouns, and need to move isForkPeriodActive from withdrawFromForkEscrow to returnTokensToOwner.


## Assessed type

Context