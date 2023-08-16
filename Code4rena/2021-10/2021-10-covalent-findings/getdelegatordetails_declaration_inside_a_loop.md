## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [getDelegatorDetails declaration inside a loop](https://github.com/code-423n4/2021-10-covalent-findings/issues/39) 

# Handle

pants


# Vulnerability details

Declaration inside a loop is less gas efficient than before it. See line 462 for example.

