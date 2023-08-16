## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Event params are of no practical value](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/163) 

# Handle

hack3r-0m


# Vulnerability details

https://github.com/code-423n4/2021-09-defiProtocol/blob/main/contracts/contracts/Factory.sol#L87

```
emit BasketLicenseProposed(msg.sender, tokenName);
```

same event can be emitted with excat same parameters multiple times causing confusion to actors relying on it.


Mitigation:

Add proposal id or some other parameter

