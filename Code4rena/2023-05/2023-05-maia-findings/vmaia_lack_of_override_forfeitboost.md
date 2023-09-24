## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-07

# [vMaia Lack of override forfeitBoost](https://github.com/code-423n4/2023-05-maia-findings/issues/738) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/vMaia.sol#L91


# Vulnerability details

## Impact
Lack of override forfeitBoost ,  when need `forfeit` will underflow

## Proof of Concept
In `vMaia`, override the `claimBoost()` code to be empty to avoid fail

The code and comments are as follows.

```solidity
    /// @dev Boost can't be claimed; does not fail. It is all used by the partner vault.
    function claimBoost(uint256 amount) public override {}
```

But it does not override the corresponding `forfeitBoost()`
This will still reduce `userClaimedBoost` when `forfeit()` is needed, resulting in `underflow`

`UtilityManager.forfeitBoost()`
```solidity
    function forfeitBoost(uint256 amount) public virtual {
        if (amount == 0) return;
@>      userClaimedBoost[msg.sender] -= amount;
        address(gaugeBoost).safeTransferFrom(msg.sender, address(this), amount);

        emit ForfeitBoost(msg.sender, amount);
    }
```    


So you should also override `forfeitBoost()` and turn it into an empty code to avoid failure when you need to use `forfeit`


## Tools Used

## Recommended Mitigation Steps

```solidity
contract vMaia is ERC4626PartnerManager {

+   /// @dev Boost can't be forfeit; does not fail.
+   function forfeitBoost(uint256 amount) public override {}

...
```


## Assessed type

Context