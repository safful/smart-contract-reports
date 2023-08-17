## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- H-01

# [TRANSFERING KIBToken TO YOURSELF INCREASES YOUR BALANCE](https://github.com/code-423n4/2023-02-kuma-findings/issues/3) 

# Lines of code

https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KIBToken.sol#L290-L291


# Vulnerability details

## Impact
using temporary variables to update balances is a dangerous construction.
If transferred to yourself, it will cause your balance to increase, thus growing the token balance infinitely

## Proof of Concept

KIBToken overrides _transfer() to perform the transfer of the token, the code is as follows:

```solidity
    function _transfer(address from, address to, uint256 amount) internal override {
        if (from == address(0)) {
            revert Errors.ERC20_TRANSFER_FROM_THE_ZERO_ADDRESS();
        }
        if (to == address(0)) {
            revert Errors.ERC20_TRANSER_TO_THE_ZERO_ADDRESS();
        }
        _refreshCumulativeYield();
        _refreshYield();

        uint256 startingFromBalance = this.balanceOf(from);
        if (startingFromBalance < amount) {
            revert Errors.ERC20_TRANSFER_AMOUNT_EXCEEDS_BALANCE();
        }
        uint256 newFromBalance = startingFromBalance - amount;
        uint256 newToBalance = this.balanceOf(to) + amount;

        uint256 previousEpochCumulativeYield_ = _previousEpochCumulativeYield;
        uint256 newFromBaseBalance = WadRayMath.wadToRay(newFromBalance).rayDiv(previousEpochCumulativeYield_);
        uint256 newToBaseBalance = WadRayMath.wadToRay(newToBalance).rayDiv(previousEpochCumulativeYield_);

        if (amount > 0) {
            _totalBaseSupply -= (_baseBalances[from] - newFromBaseBalance);
            _totalBaseSupply += (newToBaseBalance - _baseBalances[to]);
            _baseBalances[from] = newFromBaseBalance;
            _baseBalances[to] = newToBaseBalance;//<--------if from==to,this place Will overwrite the reduction above
        }

        emit Transfer(from, to, amount);
    }
```
From the code above we can see that using temporary variables "newToBaseBalance" to update balances

Using temporary variables is a dangerous construction.

If the from and to are the same, the balance[to] update will overwrite the balance[from] update
simplify the example:

Suppose: balance[alice]=10 ,  and execute transferFrom(from=alice,to=alice,5)
Define the temporary variable:  temp_variable = balance[alice]=10
so update the steps as follows:

1.balance[to=alice] = temp_variable - 5 =5
2.balance[from=alice] = temp_variable + 5 =15

after alice transferred it to herself, the balance was increased by 5

The test code is as follows:

add to KIBToken.transfer.t.sol
```soldity
    //test from == to
    function test_transfer_same() public {
        _KIBToken.mint(_alice, 10 ether);
        assertEq(_KIBToken.balanceOf(_alice), 10 ether);
        vm.prank(_alice);
        _KIBToken.transfer(_alice, 5 ether);   //<-----alice transfer to alice 
        assertEq(_KIBToken.balanceOf(_alice), 15 ether); //<-----increases 5 eth
    }
```
```console
forge test --match test_transfer_same -vvv

Running 1 test for test/kuma-protocol/kib-token/KIBToken.transfer.t.sol:KIBTokenTransfer
[PASS] test_transfer_same() (gas: 184320)
Test result: ok. 1 passed; 0 failed; finished in 24.67ms
```


## Tools Used

## Recommended Mitigation Steps

a more general method is use:
```solidity
balance[to]-=amount
balance[from]+=amount
```

In view of the complexity of the amount calculation, if the code is to be easier to read, it is recommended：

```solidity
    function _transfer(address from, address to, uint256 amount) internal override {
        if (from == address(0)) {
            revert Errors.ERC20_TRANSFER_FROM_THE_ZERO_ADDRESS();
        }
        if (to == address(0)) {
            revert Errors.ERC20_TRANSER_TO_THE_ZERO_ADDRESS();
        }
        _refreshCumulativeYield();
        _refreshYield();

+       if (from != to) {
	        uint256 startingFromBalance = this.balanceOf(from);
	        if (startingFromBalance < amount) {
	            revert Errors.ERC20_TRANSFER_AMOUNT_EXCEEDS_BALANCE();
	        }
	        uint256 newFromBalance = startingFromBalance - amount;
	        uint256 newToBalance = this.balanceOf(to) + amount;

	        uint256 previousEpochCumulativeYield_ = _previousEpochCumulativeYield;
	        uint256 newFromBaseBalance = WadRayMath.wadToRay(newFromBalance).rayDiv(previousEpochCumulativeYield_);
	        uint256 newToBaseBalance = WadRayMath.wadToRay(newToBalance).rayDiv(previousEpochCumulativeYield_);

	        if (amount > 0) {
	            _totalBaseSupply -= (_baseBalances[from] - newFromBaseBalance);
	            _totalBaseSupply += (newToBaseBalance - _baseBalances[to]);
	            _baseBalances[from] = newFromBaseBalance;
	            _baseBalances[to] = newToBaseBalance;
	        }
+       }
        emit Transfer(from, to, amount);
    }
```