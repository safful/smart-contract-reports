## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [functions safeTransferFrom and transferFrom are too similar](https://github.com/code-423n4/2021-06-realitycards-findings/issues/34) 

# Handle

pauliax


# Vulnerability details

## Impact
function safeTransferFrom is almost identical to function transferFrom. It would be better to reduce code duplication by re-using the code.

## Recommended Mitigation Steps
   function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory _data
    ) public override {
        transferFrom(from, to, tokenId);
        _data;
    }

