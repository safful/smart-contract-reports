## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [double reading from memory inside a for loop.](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/11) 

# Handle

pants


# Vulnerability details

At lines 197,199 PrizeDistributionHistory for example, the value _prizeDistribution.distributions[index] is read twice from memory instead of caching it as a local variable distributions_index = _prizeDistribution.distributions[index]; and using it instead.

Same at DrawCalculator.sol at lines 312, 315, 319 (here _picks[index] is read 3 times)


