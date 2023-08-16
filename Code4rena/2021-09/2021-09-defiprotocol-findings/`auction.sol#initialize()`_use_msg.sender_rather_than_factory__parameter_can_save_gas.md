## Tags

- bug
- duplicate
- G (Gas Optimization)
- sponsor confirmed

# [`Auction.sol#initialize()` Use msg.sender rather than factory_ parameter can save gas](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/137) 

# Handle

WatchPug


# Vulnerability details

https://github.com/code-423n4/2021-09-defiProtocol/blob/main/contracts/contracts/Auction.sol#L47-L52

`Auction.sol#initialize()` is using the factory_ parameter as the value of `factory`, while `Basket.sol#initialize()` uses `msg.sender`.

https://github.com/code-423n4/2021-09-defiProtocol/blob/main/contracts/contracts/Basket.sol#L39

Consider changing to `msg.sender` and remove the `factory_` parameter for the purpose of consistency and gas saving.

