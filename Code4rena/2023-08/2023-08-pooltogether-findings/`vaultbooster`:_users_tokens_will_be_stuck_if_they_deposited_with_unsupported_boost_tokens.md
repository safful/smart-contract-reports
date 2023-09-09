## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-11

# [`VaultBooster`: users tokens will be stuck if they deposited with unsupported boost tokens](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/22) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBooster.sol#L171-L176


# Vulnerability details

## Impact
- In `VaultBooster` contract : users can deposite their tokens to participate in boosting the chances of a vault winning.
- But in `deposit` function: users can deposit **any** ERC20 tokens without verifying if the token is supported or not (has a boost set to it; registered in `_boosts[_token]`).

- As there's no mechanism implemented in the contract for users to retreive their deposited tokens if these tokens are not supported; then they will lose them unless withdrawn by the `VaultBooster` owner; and then the owner transfers these tokens back to the users.
- If this behaviour is intended by design; then there must be a machanism to save unsupported deposited tokens with user address and amount; and another withdraw function accessible by the owner that enables transferring stuck tokens to their owners; or simply make `deposit` function reverts if the token is unsupported.

## Proof of Concept

- Code:

[Line 171-176](https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBooster.sol#L171-L176)

```solidity
File: pt-v5-vault-boost/src/VaultBooster.sol
Line 171-176:
  function deposit(IERC20 _token, uint256 _amount) external {
    _accrue(_token);
    _token.safeTransferFrom(msg.sender, address(this), _amount);

    emit Deposited(_token, msg.sender, _amount);
  }
```

- Foundry PoC:

1. A `MockERC20.t.sol` contract is added to the test folder to simulate the user experience (miting/approving..),

```solidity
pragma solidity 0.8.19;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20 {
  constructor() ERC20("MockToken", "MT") {}

  function mint(address account, uint256 amount) public returns (bool) {
    _mint(account, amount);
    return true;
  }
}
```

add remappings to the `foundry.toml`:  
 remappings = ["@openzeppelin/=lib/openzeppelin-contracts/"]

2.  This test is set in `VaultBooster.t.sol` file, where a user deposits unsuppotred token (basically no boosts were set),tries to liquidate but the call will revert, then the unsupported tokens can be withdrawn by the owner only (and not transferred to the original depositor):
    add this import line at the top of the test file:

```solidity
import "./MockERC20.t.sol";
```

add this test `testDepositWithUnsupportedToken()` to the `VaultBooster.t.sol` file:

```solidity
  function testDepositWithUnsupportedToken() public {
    //0- minting unsupportedToken to the user
    address user = address(0x2);
    uint256 userBalance = 1e18;
    MockERC20 unsupportedToken = new MockERC20();
    vm.startPrank(user);
    unsupportedToken.mint(user, userBalance);
    assertEq(unsupportedToken.balanceOf(user), userBalance);

    //1.the user deposits unsupported token (mainly there's no boosts set for any tokens):
    unsupportedToken.approve(address(booster), userBalance);

    vm.expectEmit(true, true, true, true);
    emit Deposited(unsupportedToken, user, userBalance);
    booster.deposit(unsupportedToken, userBalance);
    assertEq(unsupportedToken.balanceOf(user), 0);
    assertEq(unsupportedToken.balanceOf(address(booster)), userBalance);

    //2. assertions that the deposited unsupportedToken doesn't have a boost set for it:
    Boost memory boost = booster.getBoost(unsupportedToken);
    assertEq(boost.liquidationPair, address(0));
    assertEq(boost.multiplierOfTotalSupplyPerSecond.unwrap(), 0, "multiplier");
    assertEq(boost.tokensPerSecond, 0, "tokensPerSecond");
    assertEq(boost.lastAccruedAt, block.timestamp); //as the deposit function accrues rewards for the deposited token boost

    //3. the user tries to call liquidate to get back his tokens,but the call will revert as there's no boost set for this token (it's unsupported):
    vm.expectRevert(abi.encodeWithSelector(OnlyLiquidationPair.selector));
    booster.liquidate(user, address(prizeToken), 0, address(unsupportedToken), userBalance);
    vm.stopPrank();

    //4. unless the owner tries  withdraws the user stuck tokens,(these tokens will be transferred to the owner address not to the original depositor address) :
    assertEq(unsupportedToken.balanceOf(address(this)), 0);
    assertEq(unsupportedToken.balanceOf(address(booster)), userBalance);
    booster.withdraw(unsupportedToken, userBalance);
    assertEq(unsupportedToken.balanceOf(address(this)), userBalance);
    assertEq(unsupportedToken.balanceOf(address(booster)), 0);
  }
```

3. Test result:

```bash
$ forge test --match-test testDepositWithUnsupportedToken
Running 1 test for test/VaultBooster.t.sol:VaultBoosterTest
[PASS] testDepositWithUnsupportedToken() (gas: 636179)
Test result: ok. 1 passed; 0 failed; finished in 3.36ms
```

## Tools Used

Manual Testing & Foundry.

## Recommended Mitigation Steps

Update `deposit` function to revert if the user tries to deposit unsuppoerted tokens (that doesn't have a boost set):

```diff
  function deposit(IERC20 _token, uint256 _amount) external {
+    if(_boosts[_token].liquidationPair==address(0)) revert();
    _accrue(_token);
    _token.safeTransferFrom(msg.sender, address(this), _amount);

    emit Deposited(_token, msg.sender, _amount);
  }
```


## Assessed type

Token-Transfer