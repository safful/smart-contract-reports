## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas optimization: Placement of require statements in `TimeswapPair:pay()`](https://github.com/code-423n4/2022-01-timeswap-findings/issues/80) 

# Handle

Dravee


# Vulnerability details

## Impact
Some of the require statements can be placed earlier to reduce gas usage on revert.
As a reminder from the [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf), Appendix G and Appendix H:
- TIMESTAMP costs 2 gas
- ADDRESS costs 2 gas
- MLOAD costs 3 gas

## Proof of Concept
The following can be reorder to save gas on revert: 
```
        require(block.timestamp < maturity, 'E202'); 
        require(ids.length == assetsIn.length && ids.length == collateralsOut.length, 'E205');
        require(to != address(0), 'E201');
        require(to != address(this), 'E204');
```
to
```
        require(block.timestamp < maturity, 'E202');
        require(to != address(0), 'E201');
        require(to != address(this), 'E204');
        require(ids.length == assetsIn.length && ids.length == collateralsOut.length, 'E205');
```

## Tools Used
VS Code

## Recommended Mitigation Steps
Relocate the said require statements

