## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-18

# [borrowRateMaxMantissa should be specific to the chain protocol is being deployed to](https://github.com/code-423n4/2023-07-moonwell-findings/issues/18) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MTokenInterfaces.sol#L23
https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/MToken.sol#L403


# Vulnerability details

## Impact
- The purpose of [borrowRateMaxMantissa](https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MTokenInterfaces.sol#L23) is to put the protocol in failure mode when absurd utilisation makes borrowRate [absurd](https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/MToken.sol#L403). It is defined as a constant for all the chains. It should really be changed according to average blocktime of the chain the protocol is being deployed to.
- borrowRateMaxMantissa = 0.0005e16 translates to maximum borrow rate of .0005% / block. 
- For Ethereum chain that has 12 seconds of average block time, this translates to maximum borrow rate of `0.0005% * (365 * 24 * 3600)/12 = 1314`. For BNB with 3 seconds of average block time it is `0.0005% * (365 * 24 * 3600)/3 = 5256%`. For fantom with 1 second of average block time it is  `0.0005% * (365 * 24 * 3600)/1 = 15768%`. 


## Proof of Concept
https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MTokenInterfaces.sol#L23
https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/MToken.sol#L403

## Tools Used
- Manual Review

## Recommended Mitigation Steps
- Decide on a maximum borrow rate protocol is ok with (my suggestion is ~1000%) and change `borrowRateMaxMantissa` according to what chain it is being deployed to. 


## Assessed type

Other