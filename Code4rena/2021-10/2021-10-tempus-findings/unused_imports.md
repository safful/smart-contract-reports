## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [Unused imports](https://github.com/code-423n4/2021-10-tempus-findings/issues/33) 

# Handle

pauliax


# Vulnerability details

## Impact
There are unused imports. They will increase the size of deployment with no real benefit. Consider removing unused imports to save some gas. Examples of such imports are:

 In TempusAMMUserDataHelpers
  import "@balancer-labs/v2-solidity-utils/contracts/openzeppelin/IERC20.sol";

In TempusController
  import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

In TempusAMM
  import "@balancer-labs/v2-solidity-utils/contracts/helpers/WordCodec.sol";

## Recommended Mitigation Steps
Consider removing them to reduce deployment gas usage.

