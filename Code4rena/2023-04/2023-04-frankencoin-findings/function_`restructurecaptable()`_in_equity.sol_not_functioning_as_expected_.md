## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [function `restructureCapTable()` in Equity.sol not functioning as expected ](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/941) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L313


# Vulnerability details

## Impact
Incorrect typo in function `restructureCapTable()` leading to only burning tokens of first address of `addressToWipe` array arguement. 

## Proof of Concept
Here, in L313, addressToWipe[0] only takes first address of the array. While ignoring the rest and also since first address's tokens are burned it will fail `addressesToWipe` array has more than one addresses.
```
    function restructureCapTable(address[] calldata helpers, address[] calldata addressesToWipe) public {
        require(zchf.equity() < MINIMUM_EQUITY);
        checkQualified(msg.sender, helpers);
        for (uint256 i = 0; i<addressesToWipe.length; i++){
            address current = addressesToWipe[0];
            _burn(current, balanceOf(current));
        }
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Change `address current = addressesToWipe[0];` ==> ` address current = addressesToWipe[i];`