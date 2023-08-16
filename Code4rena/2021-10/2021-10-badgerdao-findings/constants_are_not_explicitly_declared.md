## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Constants are not explicitly declared](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/57) 

# Handle

WatchPug


# Vulnerability details

It's a best practice to use constant variables rather than literal values to make the code easier to understand and maintain.

The literal `1e18` is used throughout the contracts multiple times.

Consider defining a constant variable for the literal value used and giving it a clear and self-explanatory name.

