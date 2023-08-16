## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [caching multiple used variables](https://github.com/code-423n4/2021-10-union-findings/issues/106) 

# Handle

pants


# Vulnerability details

In Treasury.editSchedule

    function editSchedule(
        uint256 dripStart_,
        uint256 dripRate_,
        address target_,
        uint256 amount_
    ) public onlyAdmin {
        require(tokenSchedules[target_].target != address(0), "Target schedule doesn't exist");
        tokenSchedules[target_].dripStart = dripStart_;
        tokenSchedules[target_].dripRate = dripRate_;
        tokenSchedules[target_].amount = amount_;
    }


We suggest to cache  tokenSchedules[target_] at start and then use the cached value to save repeated access to a *storage* state variable.

