## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Using 10**X for constants isn't gas efficient](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/148) 

# Handle

tensors


# Vulnerability details

Constants of form 10**X can be rewritten as 1eX and this will save gas.

