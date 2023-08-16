## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Gas optimizations - Remove isMarket from RCMarket](https://github.com/code-423n4/2021-06-realitycards-findings/issues/161) 

# Handle

a_delamo


# Vulnerability details

## Impact

`RCMarket` contains the constant variable `isMarket` to indicate it is a Market `bool public constant override isMarket = true;`.
This is after used in `RCFactory`
```
function changeMarketApproval(address _market) external onlyGovernors {
        require(_market != address(0));
        // check it's an RC contract
        IRCMarket _marketToApprove = IRCMarket(_market);
        assert(_marketToApprove.isMarket());
        isMarketApproved[_market] = !isMarketApproved[_market];
        emit LogMarketApproved(_market, isMarketApproved[_market]);
    }
```

Why not use `mappingOfMarkets` to verify the address is a Market? This would reduce the state space used.

