## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas optimation proposal struct](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/238) 

# Handle

goatbug


# Vulnerability details

## Impact
Use less storage slots

## Proof of Concept
    struct Proposal {
        uint256 licenseFee;
        string tokenName;
        string tokenSymbol;
        address proposer;
        address[] tokens;
        uint256[] weights;
        address basket;
    }
License fee is a smaller number does not need to be uint256. 

Could use an 8 bit value and pack it comfortable with one of the addresses to save a full storage slot. 

## Tools Used

## Recommended Mitigation Steps

