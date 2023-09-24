## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [Although `ERC20Boost.decrementGaugesBoostIndexed` function would require user to remove all of her or his boost from a deprecated gauge at once, such user can instead call `ERC20Boost.decrementGaugeBoost` function for multiple times to utilize such deprecated gauge and decrement its `userGaugeBoost` for multiple times](https://github.com/code-423n4/2023-05-maia-findings/issues/904) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/62f4f01a522dcbb4b9abfe2f6783bbb67c0da022/src/erc-20/ERC20Boost.sol#L175-L187
https://github.com/code-423n4/2023-05-maia/blob/62f4f01a522dcbb4b9abfe2f6783bbb67c0da022/src/erc-20/ERC20Boost.sol#L198-L200
https://github.com/code-423n4/2023-05-maia/blob/62f4f01a522dcbb4b9abfe2f6783bbb67c0da022/src/erc-20/ERC20Boost.sol#L203-L227


# Vulnerability details

## Impact
When the `gauge` input corresponds to a deprecated gauge, calling the following `ERC20Boost.decrementGaugeBoost` function can still execute `gaugeState.userGaugeBoost -= boost.toUint128()` if `boost >= gaugeState.userGaugeBoost` is false.

https://github.com/code-423n4/2023-05-maia/blob/62f4f01a522dcbb4b9abfe2f6783bbb67c0da022/src/erc-20/ERC20Boost.sol#L175-L187
```solidity
    function decrementGaugeBoost(address gauge, uint256 boost) public {
        GaugeState storage gaugeState = getUserGaugeBoost[msg.sender][gauge];
        if (boost >= gaugeState.userGaugeBoost) {
            _userGauges[msg.sender].remove(gauge);
            delete getUserGaugeBoost[msg.sender][gauge];

            emit Detach(msg.sender, gauge);
        } else {
            gaugeState.userGaugeBoost -= boost.toUint128();

            emit DecrementUserGaugeBoost(msg.sender, gauge, gaugeState.userGaugeBoost);
        }
    }
```

However, for the same deprecated gauge, calling the following `ERC20Boost.decrementAllGaugesBoost` and `ERC20Boost.decrementGaugesBoostIndexed` functions below would execute `_userGauges[msg.sender].remove(gauge)` and `delete getUserGaugeBoost[msg.sender][gauge]` without executing `gaugeState.userGaugeBoost -= boost.toUint128()` because `_deprecatedGauges.contains(gauge)` is true. Hence, an inconsistency exists between the `ERC20Boost.decrementGaugeBoost` and `ERC20Boost.decrementGaugesBoostIndexed` functions when the corresponding gauge is deprecated. As a result, although the `ERC20Boost.decrementGaugesBoostIndexed` function would require the user to remove all of her or his boost from a deprecated gauge at once, such user can instead call the `ERC20Boost.decrementGaugeBoost` function for multiple times to utilize such deprecated gauge and decrement its `userGaugeBoost` for multiple times if `boost >= gaugeState.userGaugeBoost` is false each time.

https://github.com/code-423n4/2023-05-maia/blob/62f4f01a522dcbb4b9abfe2f6783bbb67c0da022/src/erc-20/ERC20Boost.sol#L198-L200
```solidity
    function decrementAllGaugesBoost(uint256 boost) external {
        decrementGaugesBoostIndexed(boost, 0, _userGauges[msg.sender].length());
    }
```

https://github.com/code-423n4/2023-05-maia/blob/62f4f01a522dcbb4b9abfe2f6783bbb67c0da022/src/erc-20/ERC20Boost.sol#L203-L227
```solidity
    function decrementGaugesBoostIndexed(uint256 boost, uint256 offset, uint256 num) public {
        address[] memory gaugeList = _userGauges[msg.sender].values();

        uint256 length = gaugeList.length;
        for (uint256 i = 0; i < num && i < length;) {
            address gauge = gaugeList[offset + i];

            GaugeState storage gaugeState = getUserGaugeBoost[msg.sender][gauge];

            if (_deprecatedGauges.contains(gauge) || boost >= gaugeState.userGaugeBoost) {
                require(_userGauges[msg.sender].remove(gauge)); // Remove from set. Should never fail.
                delete getUserGaugeBoost[msg.sender][gauge];

                emit Detach(msg.sender, gauge);
            } else {
                gaugeState.userGaugeBoost -= boost.toUint128();

                emit DecrementUserGaugeBoost(msg.sender, gauge, gaugeState.userGaugeBoost);
            }

            unchecked {
                i++;
            }
        }
    }
```

## Proof of Concept
The following steps can occur for the described scenario.
1. Alice's 1e18 boost are attached to a gauge.
2. Such gauge becomes deprecated.
3. Alice calls the `ERC20Boost.decrementGaugeBoost` function to decrement 0.5e18 boost from such deprecated gauge.
4. Alice calls the `ERC20Boost.decrementGaugeBoost` function to decrement 0.2e18 boost from such deprecated gauge.
5. Alice still has 0.3e18 boost from such deprecated gauge so she can continue utilize and decrement boost from such deprecated gauge in the future. 

## Tools Used
VSCode

## Recommended Mitigation Steps
The `ERC20Boost.decrementGaugeBoost` function can be updated to execute `require(_userGauges[msg.sender].remove(gauge))` and `delete getUserGaugeBoost[msg.sender][gauge]` if `_deprecatedGauges.contains(gauge) || boost >= gaugeState.userGaugeBoost` is true, which is similar to the `ERC20Boost.decrementGaugesBoostIndexed` function.


## Assessed type

Other