## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-02

# [The recipient receives free collateral token if an ERC20 token that deducts a fee on transfer used as baseToken](https://github.com/code-423n4/2022-12-prepo-findings/issues/52) 

# Lines of code

https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L45-L61
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L64-L78
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/DepositHook.sol#L49-L50
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/WithdrawHook.sol#L76-L77


# Vulnerability details

## Impact
- There are some ERC20 tokens that deduct a fee on every transfer call. If these tokens used as baseToken then :
	1. When depositing into the **Collateral** contract, the recipient will receive collateral token more than what they should receive.
	2. The **DepositRecord** contract will track wrong user deposit amounts and wrong globalNetDepositAmount as the added amount to both will be always more than what was actually deposited.
	3. When withdrawing from the **Collateral** contract, the user will receive less baseToken amount than what they should receive.
	4. The treasury will receive less fee and the user will receive more `PPO` tokens that occur in **DepositHook**  and **WithdrawHook**.
	
	

## Proof of Concept
Given:
	- baseToken is an ERC20 token that deduct a fee on every transfer call.
	- **FoT** is the deducted fee on transfer.

1. The user deposits baseToken to the **Collateral** contract by calling `deposit` function passing **_amount** as 100e18.
2. `baseToken.transferFrom` is called to transfer the amount from the user to the contract.
	- https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L49
4. The contract receives the **_amount** - **FoT**. Let's assume the FoT percentage is 1%. therefore, the actual amount received is 99e18.
5. When the **DepositHook** is called. the **_amount** passed is 100e18 which is wrong as it should be the actual amount 99e18.
	- https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L53
7. Calculating **collateralMintAmount** is based on the **_amount**  (100e18- the fee for treasury) which will give the recipient additional collateral token that they shouldn't receive.
	- https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L57


## Tools Used
Manual analysis

## Recommended Mitigation Steps

1. Consider calculating the actual amount by recording the balance before and after.

 	- For example:
```sh
uint256 balanceBefore = baseToken.balanceOf(address(this));
baseToken.transferFrom(msg.sender, address(this), _amount);
uint256 balanceAfter = baseToken.balanceOf(address(this));
uint256 actualAmount = balanceAfter - balanceBefore;
```

2. Then use **actualAmount** instead of **_amount** to perform any further calculations or external calls. 


Note: apply the same logic for **DepositHook**  and **WithdrawHook** as well at:
- https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/DepositHook.sol#L49-L50

- https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/WithdrawHook.sol#L76-L77