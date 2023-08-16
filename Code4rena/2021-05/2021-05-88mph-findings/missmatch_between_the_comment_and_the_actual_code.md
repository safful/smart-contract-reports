## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [Missmatch between the comment and the actual code](https://github.com/code-423n4/2021-05-88mph-findings/issues/5) 

# Handle

paulius.eth


# Vulnerability details

## Impact
Here the comment says that it should transfer from msg.sender but it actually transfers from the sender which is not always the msg.sender (e.g. sponsored txs):
  // Transfer `fundAmount` stablecoins from msg.sender
  stablecoin.safeTransferFrom(sender, address(this), fundAmount);

## Recommended Mitigation Steps
Update the comment to match the code.

