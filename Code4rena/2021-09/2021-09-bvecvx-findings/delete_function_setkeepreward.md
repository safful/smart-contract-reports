## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Delete function setKeepReward](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/31) 

# Handle

pauliax


# Vulnerability details

## Impact
even the comment says it, delete to save some gas:
    /// @notice Delete if you don't need!
    function setKeepReward(uint256 _setKeepReward) external {
        _onlyGovernance();
    }


