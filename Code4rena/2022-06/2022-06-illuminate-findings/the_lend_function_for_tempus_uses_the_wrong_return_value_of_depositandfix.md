## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [The lend function for tempus uses the wrong return value of depositAndFix](https://github.com/code-423n4/2022-06-illuminate-findings/issues/37) 

# Lines of code

https://github.com/code-423n4/2022-06-illuminate/blob/92cbb0724e594ce025d6b6ed050d3548a38c264b/lender/Lender.sol#L452-L453


# Vulnerability details

## Impact
The depositAndFix function of the TempusController contract returns two uint256 data, the first is the number of shares exchanged for the underlying token, the second is the number of principalToken exchanged for the shares, the second return value should be used in the lend function for tempus.
This will cause the contract to mint an incorrect number of illuminateTokens to the user.
## Proof of Concept
https://github.com/code-423n4/2022-06-illuminate/blob/92cbb0724e594ce025d6b6ed050d3548a38c264b/lender/Lender.sol#L452-L453
https://github.com/tempus-finance/tempus-protocol/blob/master/contracts/TempusController.sol#L52-L76
## Tools Used
None
## Recommended Mitigation Steps
interfaces.sol
```
interface ITempus {
    function maturityTime() external view returns (uint256);

    function yieldBearingToken() external view returns (IERC20Metadata);

    function depositAndFix(
        Any,
        Any,
        uint256,
        bool,
        uint256,
        uint256
    ) external returns (uint256, uint256);
}
```
Lender.sol
```
        (,uint256 returned) = ITempus(tempusAddr).depositAndFix(Any(x), Any(t), a - fee, true, r, d);
        returned -= illuminateToken.balanceOf(address(this));
```

