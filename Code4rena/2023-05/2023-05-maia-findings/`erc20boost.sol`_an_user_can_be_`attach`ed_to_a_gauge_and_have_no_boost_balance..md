## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-13

# [`ERC20Boost.sol` An user can be `attach`ed to a gauge and have no boost balance.](https://github.com/code-423n4/2023-05-maia-findings/issues/656) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L150-L172
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L80-L83
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L336-L344


# Vulnerability details

When a user boosted gauge became deprecated, the user can transfer his boost tokens. When the same gauge is reintroduced to the active gauge list, the user will boost it again even if his boost token balance is zero.

## Impact

Same amount of boost tokens can be allocated to gauges by multiple addresses.

## Proof of Concept

Let's take an example:

1. Alice calls `attach()` from gaugeA to boost it;

-  `getUserBoost[alice]` is set to balanceOf(alice)

2. Owner removes gaugeA and it's added to `_deprecatedGauges`;

3. Alice calls `updateUserBoost()`; because gaugeA is now deprecated her allocated boost is set to `userBoost` which is initialized to zero (0) :
```solidity
    function updateUserBoost(address user) external {
        uint256 userBoost = 0;
        address[] memory gaugeList = _userGauges[user].values();
        uint256 length = gaugeList.length;
        for (uint256 i = 0; i < length;) {
            address gauge = gaugeList[i];
            if (!_deprecatedGauges.contains(gauge)) {
                uint256 gaugeBoost = getUserGaugeBoost[user][gauge].userGaugeBoost;
                if (userBoost < gaugeBoost) userBoost = gaugeBoost;
            }
            unchecked {
                i++;
            }
        }
        getUserBoost[user] = userBoost;
        emit UpdateUserBoost(user, userBoost);
    } 
```
4.  `freeGaugeBoost()` returns the amount of unallocated boost tokens:

```solidity

function freeGaugeBoost(address user) public view returns (uint256) {
	return balanceOf[user] - getUserBoost[user];
}

```

5. `transfer()` has `notAttached()` modifier that ensures the transferred amount is free (not allocated to any gauge);
```solidity
    /**
     * @notice Transfers `amount` of tokens from `msg.sender` to `to` address.
     * @dev User must have enough free boost.
     * @param to the address to transfer to.
     * @param amount the amount to transfer.
     */
    function transfer(address to, uint256 amount) public override notAttached(msg.sender, amount) returns (bool) {
        return super.transfer(to, amount);
    }
```
6. Alice transfer her tokens

7. When gaugeA is added back, `addGauge(gaugeA)` she will continue to boost gaugeA even if her balance is 0


https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L150-L172

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L81C1-L83C6

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L336-L344

## Tools Used

VS Code

## Recommended Mitigation Steps
One solution is `updateUserBoost()` to loop all gauges (active and deprecated) not only the active ones:

```solidity
    function updateUserBoost(address user) external {
        uint256 userBoost = 0;
        address[] memory gaugeList = _userGauges[user].values();


        uint256 length = gaugeList.length;
        for (uint256 i = 0; i < length;) {
            address gauge = gaugeList[i];
            uint256 gaugeBoost = getUserGaugeBoost[user][gauge].userGaugeBoost;
            if (userBoost < gaugeBoost) userBoost = gaugeBoost;
            unchecked {
                i++;
            }
        }
        getUserBoost[user] = userBoost;
        emit UpdateUserBoost(user, userBoost);
    }
```
Even the `updateUserBoost()` comments indicates all `_userGauges` should be iterated over.
```solidity
/**
* @notice Update geUserBoost for a user, loop through all _userGauges
* @param user the user to update the boost for.
*/
function  updateUserBoost(address user) external;
```


## Assessed type

Other