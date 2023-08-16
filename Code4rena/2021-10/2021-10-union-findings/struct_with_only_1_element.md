## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Struct with only 1 element](https://github.com/code-423n4/2021-10-union-findings/issues/88) 

# Handle

pauliax


# Vulnerability details

## Impact
It is not efficient to have a struct with only 1 field as structs are meant for grouping related information together. A market struct can be replaced by directly pointing to a bool value:
    //before
    mapping(address => Market) public supportedMarkets;
    struct Market {
        bool isSupported;
    }

   //after
   mapping(address => bool) public supportedMarkets;


