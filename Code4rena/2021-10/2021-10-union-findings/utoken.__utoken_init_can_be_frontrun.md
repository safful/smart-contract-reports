## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [UToken.__UToken_init can be frontrun](https://github.com/code-423n4/2021-10-union-findings/issues/97) 

# Handle

pants


# Vulnerability details

The function __UToken_init can be frontrun. We recommend adding an initializer owner which only it allowed to call such functions, instead of the current _admin there.

Not sure whether frontrunning is Low / Medium risk.

