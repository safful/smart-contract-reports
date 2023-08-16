## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: Avoid double assignment on variable](https://github.com/code-423n4/2022-01-insure-findings/issues/80) 

# Handle

Dravee


# Vulnerability details

## Impact
Increased gas cost

## Proof of Concept
The variable `T_0` can go through 2 assignments in a row:
Here:
```
75:         uint256 T_0 = _totalLiquidity;
76:         if (T_0 > T_1) {
77:             T_0 = T_1;
78:         }
```
And here:
```
134:         uint256 T_0 = _totalLiquidity;
135:         if (T_0 > T_1) {
136:             T_0 = T_1;
137:         }
```

The code can be optimized as such to save some gas:
```
        uint256 T_0 = _totalLiquidity > T_1 ? _totalLiquidity : T_1;
```

## Tools Used
VS Code

## Recommended Mitigation Steps
Apply the refacto 

