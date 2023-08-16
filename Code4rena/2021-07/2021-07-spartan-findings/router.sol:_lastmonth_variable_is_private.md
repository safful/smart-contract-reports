## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Router.sol: lastMonth variable is private](https://github.com/code-423n4/2021-07-spartan-findings/issues/50) 

# Handle

hickuphh3


# Vulnerability details

### Impact

`uint private lastMonth; // Timestamp of the start of current metric period (For UI)` 

There is no getter method for `lastMonth`, which makes the (For UI) comment is erroneous.

### Recommended Mitigation Steps

Make it `public` or edit the comment

