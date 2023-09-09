## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-34

# [Anyone can reset fees to 0 value when Vault is deployed ](https://github.com/code-423n4/2023-01-popcorn-findings/issues/78) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L540-L546


# Vulnerability details

## Impact
Anyone can reset fees to 0 value when Vault is deployed. As result protocol will stop collecting fees.

## Proof of Concept
Anyone can call `changeFees` function in order to change `fees` variable to `proposedFees`.
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L540-L546
```solidity
    function changeFees() external {
        if (block.timestamp < proposedFeeTime + quitPeriod)
            revert NotPassedQuitPeriod(quitPeriod);


        emit ChangedFees(fees, proposedFees);
        fees = proposedFees;
    }
```
There is a check that should not allow anyone to call function before `quitPeriod` has passed after fees changing was proposed.
However function doesn't check that `proposedFeeTime` is not 0, so that means that after Vault has deployed, anyone can call this function the check will pass.
That means that `fees` will be set to the `proposedFees`, which is 0.
As result protocol will stop collecting fees.

Use this test inside Vault.t.sol. Here you can see that noone proposed fee changing, but it was changed and set fees to 0.
```solidity
function test__changeFees() public {
    VaultFees memory newVaultFees = VaultFees({ deposit: 0, withdrawal: 0, management: 0, performance: 0 });
    //noone proposed
    //vault.proposeFees(newVaultFees);

    vm.warp(block.timestamp + 3 days);

    vm.expectEmit(false, false, false, true, address(vault));
    emit ChangedFees(VaultFees({ deposit: 0, withdrawal: 0, management: 0, performance: 0 }), newVaultFees);

    vault.changeFees();

    (uint256 deposit, uint256 withdrawal, uint256 management, uint256 performance) = vault.fees();
    assertEq(deposit, 0);
    assertEq(withdrawal, 0);
    assertEq(management, 0);
    assertEq(performance, 0);
  }
```
## Tools Used
VsCode
## Recommended Mitigation Steps
```solidity
    function changeFees() external {
        if (proposedFeeTime == 0 || block.timestamp < proposedFeeTime + quitPeriod)
            revert NotPassedQuitPeriod(quitPeriod);


        emit ChangedFees(fees, proposedFees);
        fees = proposedFees;
    }
```