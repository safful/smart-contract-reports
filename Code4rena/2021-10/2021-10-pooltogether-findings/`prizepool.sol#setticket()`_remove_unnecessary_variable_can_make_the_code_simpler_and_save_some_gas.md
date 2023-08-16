## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [`PrizePool.sol#setTicket()` Remove unnecessary variable can make the code simpler and save some gas](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/36) 

# Handle

WatchPug


# Vulnerability details

https://github.com/pooltogether/v4-core/blob/055335bf9b09e3f4bbe11a788710dd04d827bf37/contracts/prize-pool/PrizePool.sol#L284-L297

```solidity
function setTicket(ITicket _ticket) external override onlyOwner returns (bool) {
    address _ticketAddress = address(_ticket);

    require(_ticketAddress != address(0), "PrizePool/ticket-not-zero-address");
...
```

`_ticketAddress` is unnecessary as it's being used only once.

### Recommendation

Change to:

```solidity
function setTicket(ITicket _ticket) external override onlyOwner returns (bool) {
    require(address(_ticket) != address(0), "PrizePool/ticket-not-zero-address");
...
```

