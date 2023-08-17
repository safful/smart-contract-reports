## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-02

# [Owner can transfer all ERC20 reward token out using function recoverERC20](https://github.com/code-423n4/2022-10-paladin-findings/issues/68) 

# Lines of code

https://github.com/code-423n4/2022-10-paladin/blob/d6d0c0e57ad80f15e9691086c9c7270d4ccfe0e6/contracts/WardenPledge.sol#L653


# Vulnerability details

## Impact

the function recoverERC20 is very privileged. It means to recover any token that is accidently sent to the contract.

```solidity
function recoverERC20(address token) external onlyOwner returns(bool) {
	if(minAmountRewardToken[token] != 0) revert Errors.CannotRecoverToken();

	uint256 amount = IERC20(token).balanceOf(address(this));
	if(amount == 0) revert Errors.NullValue();
	IERC20(token).safeTransfer(owner(), amount);

	return true;
}
```

However, admin / owner can use this function to transfer all the reserved reward token, which result in fund loss of the pledge creator and the lose of reward for users that want to delegate the veToken.

Also, the recovered token is sent to owner directly instead of sending to a recipient address.

The safeguard 

```solidity
if(minAmountRewardToken[token] != 0)
```

cannot stop owner transferring fund because if the owner is compromised or misbehave, he can adjust the whitelist easily.

## Proof of Concept

The admin can set minAmountRewardToken[token] to 0 first by calling updateRewardToken

```solidity
function updateRewardToken(address token, uint256 minRewardPerSecond) external onlyOwner {
```

By doing this the admin remove the token from the whitelist,

then the token can call recoverERC20 to transfer all the token into the owner wallet.

```solidity
function recoverERC20(address token) external onlyOwner returns(bool) {
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the project use multisig wallet to safeguard the owner wallet.

We can also keep track of the reserved amount for rewarding token and only transfer the remaining amount of token out.

```solidity
 pledgeAvailableRewardAmounts[pledgeId] += totalRewardAmount;
 reservedReward[token] += totalRewardAmount;
```

Then we can change the implementation to:

```solidity
function recoverERC20(address token, address recipient) external onlyOwner returns(bool) {

	uint256 amount = IERC20(token).balanceOf(address(this));
	if(amount == 0) revert Errors.NullValue();

	if(minAmountRewardToken[token] == 0) {
	 // if it is not whitelisted, we assume it is mistakenly sent, 
	   // we transfer the token to recipient
	 IERC20(token).safeTransfer(recipient, amount);
	} else {
	// revert if the owner over transfer
	if(amount >  reservedReward[token]) revert rewardReserved();
	  IERC20(token).safeTransfer(recipient, amount - reservedReward[token]);
	}

	return true;

}
```