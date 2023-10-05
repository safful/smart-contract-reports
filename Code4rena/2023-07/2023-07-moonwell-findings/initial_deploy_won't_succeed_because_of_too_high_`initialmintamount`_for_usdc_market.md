## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor confirmed
- M-11

# [Initial deploy won't succeed because of too high `initialMintAmount` for USDC market](https://github.com/code-423n4/2023-07-moonwell-findings/issues/143) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/test/proposals/Configs.sol#L55


# Vulnerability details

## Impact
At the time of deploy, deployer initializes token markets with initial amount to prevent exploit. At least USDC and WETH markets will be initialized during deploy.
But `initialMintAmount` is hardcoded to `1e18` which is of for WETH (1,876 USD), but unrealistic for USDC (1,000,000,000,000 USD). Therefore deploy will fail.

## Proof of Concept
Here `initialMintAmount = 1 ether`:
https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/test/proposals/Configs.sol#L55
```solidity
    /// @notice initial mToken mint amount
    uint256 public constant initialMintAmount = 1 ether;
```

Here this amount is approved to MToken contract and supplied to mint MToken:
https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/test/proposals/mips/mip00.sol#L334-L379
```solidity
            for (uint256 i = 0; i < cTokenConfigs.length; i++) {
                Configs.CTokenConfiguration memory config = cTokenConfigs[i];

                address cTokenAddress = addresses.getAddress(
                    config.addressesString
                );

               ...

                /// Approvals
                _pushCrossChainAction(
                    config.tokenAddress,
                    abi.encodeWithSignature(
                        "approve(address,uint256)",
                        cTokenAddress,
                        initialMintAmount
                    ),
                    "Approve underlying token to be spent by market"
                );

                /// Initialize markets
                _pushCrossChainAction(
                    cTokenAddress,
                    abi.encodeWithSignature("mint(uint256)", initialMintAmount),
                    "Initialize token market to prevent exploit"
                );

                ...
            }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Specify `initialMintAmount` for every token separately in config


## Assessed type

Other