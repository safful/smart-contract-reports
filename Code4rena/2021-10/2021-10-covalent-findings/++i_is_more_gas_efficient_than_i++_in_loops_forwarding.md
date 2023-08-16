## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [++i is more gas efficient than i++ in loops forwarding](https://github.com/code-423n4/2021-10-covalent-findings/issues/37) 

# Handle

pants


# Vulnerability details

++i is more gas efficient than i++ in loops forwarding.
At line 440 for example.
We would also recommend to use unchecked{++i} and change i declaration to uint256

