## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- H-11

# [Malicious strategist could deny borrowers from repaying loan and force liquidation by setting a extremely high vault fee](https://github.com/code-423n4/2023-01-astaria-findings/issues/295) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L605
https://github.com/code-423n4/2023-01-astaria/blob/main/src/test/TestHelpers.t.sol#L471


# Vulnerability details

## Impact
Issue: A malicious strategist can deny the repayment of loans by setting a extremely high vault fee during creation of a public vault. The high vault fee will cause a revert due to a failed integer conversion using SafeCastTo88(). This will lead to forced liquidation of the borrowers when the loans expire outstanding, making them lose their NFT collaterals. (Vault fee is an incentive awarded to strategist on each loan repayment,  where a percentage of the interest payment is allocated to the strategist, in terms of vault shares.)

High Likelihood: The strategist could target borrowers by refinancing outstanding loans and transfering the loans to his high fee vault, which does not require the borrowers' consents. This can be achieved as refinancing can be done by anyone and the logic only checks for better interest rate and duration, but not the vault fee. The borrowers will not be aware of the issue, until they attempt to repay the loan. Furthermore, the strategist could specifically target loans that are about to expire, giving little reaction time for borrowers to report the issue.

Financial gain: With the ability to force a liquidation, the strategist can possibly stand to gain financially (e.g. by refinancing the loans with a lower liquidationInitialAsk and then bid for the NFT collateral (with high gas fee) during liquidation.

Note: Even if there are no lenders willing to lend to the vault due to the high vault fee, the strategist still can lend to its own vault to faciliate the refinance.

## Proof of Concept
The bug can be replicated by changing the test case. Set vaultFee parameter to a high value (e.g. 1e13) as shown below in the file /src/test/TestHelpers.t.sol.  Then run testBasicPublicVaultLoan() in AsteriaTest.t.sol. In this test case, we will see that the strategist could create a public vault with a high vaultFee as there is no validation check for it. And any borrower could still proceed to deposit their collateral and take loan without any issues as the vaultFee is only accessed upon loan repayment. 

	function _createPublicVault(
		address strategist,
		address delegate,
		uint256 epochLength
	) internal returns (address publicVault) {
		vm.startPrank(strategist);
		//bps
		publicVault = ASTARIA_ROUTER.newPublicVault(
			epochLength,
			delegate,
			address(WETH9),
				//uint256(0)
				uint256(1e13),              //to replicate the bug, change vaultFee parameter from 0 to a high value like 1e13   
			false,
			new address[](0),
			uint256(0)
		);
		vm.stopPrank();
	}

https://github.com/code-423n4/2023-01-astaria/blob/main/src/test/TestHelpers.t.sol#L471
https://github.com/code-423n4/2023-01-astaria/blob/main/src/test/AstariaTest.t.sol#L90

However, when the borrower attempts to repay the loan, it will revert due to a failed integer conversion. As the fee is too high, convertToShare() will return a value that exceeds 88-bit, causing the safeCastTo88() in _handleStrategistInterestReward() to fail.

      uint88 feeInShares = convertToShares(fee).safeCastTo88();

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L605

## Tools Used
## Recommended Mitigation Steps
Check that the vaultFee is within a reasonable range during vault creation.