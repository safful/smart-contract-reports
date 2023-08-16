## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Unnecessary imports](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/91) 

# Handle

PranavG


# Vulnerability details

## Impact
MathUtils.sol has unused import at line #5:
import "@openzeppelin/contracts/math/SafeMath.sol";
The import is not used in any way.


## Recommended Mitigation Steps
Remove it to improve readability of the code.

