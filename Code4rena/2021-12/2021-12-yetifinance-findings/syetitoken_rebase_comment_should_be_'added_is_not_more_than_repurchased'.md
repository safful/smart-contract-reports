## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [sYETIToken rebase comment should be 'added is not more than repurchased'](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/100) 

# Handle

hyh


# Vulnerability details

# Impact

Comment is misleading, now stating the opposite of the implemented logic.

## Proof of Concept

Comment states that tokens added are `not less than amount repurchased`, while it should be `not more`:
https://github.com/code-423n4/2021-12-yetifinance/blob/main/packages/contracts/contracts/YETI/sYETIToken.sol#L246 

## Recommended Mitigation Steps

Now:
```
// Ensure that the amount of YETI tokens effectively added is >= the amount we have repurchased.
if (yetiTokenBalance < additionalYetiTokenBalance) {
		additionalYetiTokenBalance = yetiTokenBalance;
}
```

To be:
```
// Ensure that the amount of YETI tokens effectively added is <= the amount we have repurchased.
if (yetiTokenBalance < additionalYetiTokenBalance) {
		additionalYetiTokenBalance = yetiTokenBalance;
}
```

