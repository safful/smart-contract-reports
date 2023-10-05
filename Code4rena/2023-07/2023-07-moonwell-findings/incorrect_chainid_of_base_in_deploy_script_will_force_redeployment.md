## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-13

# [Incorrect chainId of Base in deploy script will force redeployment](https://github.com/code-423n4/2023-07-moonwell-findings/issues/114) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/test/utils/ChainIds.sol#L7


# Vulnerability details

## Impact
Incorrect chainId of Base in deploy parameters results in incorrect deploy and subsequent redeployment

## Proof of Concept
Contract ChainIds.sol is responsible for mapping `chainId -> wormholeChainId` which is used in contract `Addresses` to associate contract name with its address on specific chain. `Addresses` is the main contract which keeps track of all dependency addresses and passed into main `deploy()` and here addresses accessed via block.chainId:
https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/test/proposals/mips/mip00.sol#L77
```solidity
    function deploy(Addresses addresses, address) public {
        ...
            trustedSenders[0].chainId = chainIdToWormHoleId[block.chainid];
            
        ...
            memory cTokenConfigs = getCTokenConfigurations(block.chainid);
    }
```
Here you can see that Network ID of Base set to 84531. But actual network id is 8453 from [Base docs](https://docs.base.org/network-information/)
```solidity
contract ChainIds {
    uint256 public constant baseChainId = 84531;
    uint16 public constant baseWormholeChainId = 30; /// TODO update when actual base chain id is known
    
    uint256 public constant baseGoerliChainId = 84531;
    uint16 public constant baseGoerliWormholeChainId = 30;
    
    ...

    constructor() {
        ...
        chainIdToWormHoleId[baseChainId] = moonBeamWormholeChainId; /// base deployment is owned by moonbeam governance
        ...
    }
}
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Change Base Network ID to 8453


## Assessed type

Other