## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Gas optimizations - Duplicated state variable ](https://github.com/code-423n4/2021-06-realitycards-findings/issues/139) 

# Handle

a_delamo


# Vulnerability details

## Impact
On `RCOrderbook`, there are duplicated the state variable `treasuryAddress` and `treasury`

```
    address public treasuryAddress;
    IRCTreasury public treasury;
```
```
    constructor(address _factoryAddress, address _treasuryAddress) {
        factoryAddress = _factoryAddress;
        treasuryAddress = _treasuryAddress;
        treasury = IRCTreasury(treasuryAddress);
        uberOwner = msgSender();
    }

```

