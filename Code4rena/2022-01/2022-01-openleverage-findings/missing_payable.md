## Tags

- bug
- 2 (Med Risk)
- resolved
- sponsor confirmed

# [Missing payable](https://github.com/code-423n4/2022-01-openleverage-findings/issues/61) 

# Handle

robee


# Vulnerability details

The following functions are not payable but uses msg.value - therefore the function must be payable.
This can lead to undesired behavior.

        LPool.sol, addReserves should be payable since using msg.value


