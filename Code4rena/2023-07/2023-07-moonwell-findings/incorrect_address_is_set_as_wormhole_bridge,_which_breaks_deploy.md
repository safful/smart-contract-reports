## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor disputed
- M-12

# [Incorrect address is set as Wormhole Bridge, which breaks deploy](https://github.com/code-423n4/2023-07-moonwell-findings/issues/118) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/test/proposals/Addresses.sol#L50


# Vulnerability details

## Impact
Wormhole Bridge will have incorrect address, which disables whole TemporalGovernance contract and therefore administration of Moonbeam. But during deployment markets are initialized with initial token amounts to prevent exploits, therefore TemporalGovernance must own this initial tokens. But because of incorrect bridge address TemporalGovernance can't perform any action and these tokens are brick.
Protocol needs redeployment and loses initial tokens.

## Proof of Concept
To grab this report easily I divided it into 3 parts
1) Why `WORMHOLE_CORE` set incorrectly
There is no config for Base Mainnet, however this comment states for used parameters:
https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/test/proposals/mips/mip00.sol#L55-L61
```solidity
    /// --------------------------------------------------------------------------------------------------///
    /// Chain Name	       Wormhole Chain ID   Network ID	Address                                      |///
    ///  Ethereum (Goerli)   	  2	                5	    0x706abc4E45D419950511e474C7B9Ed348A4a716c   |///
    ///  Ethereum (Sepolia)	  10002          11155111	    0x4a8bc80Ed5a4067f1CCf107057b8270E0cC11A78   |///
    ///  Base	                 30    	        84531	    0xA31aa3FDb7aF7Db93d18DDA4e19F811342EDF780   |///
    ///  Moonbeam	             16	             1284 	    0xC8e2b0cD52Cf01b0Ce87d389Daa3d414d4cE29f3   |///
    /// --------------------------------------------------------------------------------------------------///
```
Why to treat this comment? Other parameters are correct except Base Network ID. But Base Network ID has reflection in code, described in #114.
0xA31aa3FDb7aF7Db93d18DDA4e19F811342EDF780 is address of Wormhole Token Bridge on Base Testnet [according to docs](https://docs.wormhole.com/wormhole/supported-environments/evm#base) and this address has no code [in Base Mainnet](https://basescan.org/address/0xA31aa3FDb7aF7Db93d18DDA4e19F811342EDF780)

2) Why governor can't execute proposals with incorrect address of Wormhole Bridge
Proposal execution will revert here because address `wormholeBridge` doesn't contain code:
https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Governance/TemporalGovernor.sol#L344-L350
```solidity
    function _executeProposal(bytes memory VAA, bool overrideDelay) private {
        // This call accepts single VAAs and headless VAAs
        (
            IWormhole.VM memory vm,
            bool valid,
            string memory reason
        ) = wormholeBridge.parseAndVerifyVM(VAA);
        ...
    }
```

3) Impact of inability to execute proposals in TemporalGovernor.sol
Here you can see that deployer should provide initial amounts of tokens to initialize markets:
```solidity
    /// @notice the deployer should have both USDC, WETH and any other assets that will be started as
    /// listed to be able to deploy on base. This allows the deployer to be able to initialize the
    /// markets with a balance to avoid exploits
    function deploy(Addresses addresses, address) public {
    ...
    }
```
And here governor approves underlying token to MToken and mints initial mTokens. It means that governor has tokens on balance.
```solidity
    function build(Addresses addresses) public {
        /// Unitroller configuration
        _pushCrossChainAction(
            addresses.getAddress("UNITROLLER"),
            abi.encodeWithSignature("_acceptAdmin()"),
            "Temporal governor accepts admin on Unitroller"
        );

        ...

        /// set mint unpaused for all of the deployed MTokens
        unchecked {
            for (uint256 i = 0; i < cTokenConfigs.length; i++) {
                Configs.CTokenConfiguration memory config = cTokenConfigs[i];

                address cTokenAddress = addresses.getAddress(
                    config.addressesString
                );

                _pushCrossChainAction(
                    unitrollerAddress,
                    abi.encodeWithSignature(
                        "_setMintPaused(address,bool)",
                        cTokenAddress,
                        false
                    ),
                    "Unpause MToken market"
                );

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
        }

        ...
    }
```

To conclude, there is no way to return the tokens from the TemporalGovernor intended for market initialization

## Tools Used
Manual Review

## Recommended Mitigation Steps
Change WORMHOLE_CORE to 0xbebdb6C8ddC678FfA9f8748f85C815C556Dd8ac6 [according to docs](https://docs.wormhole.com/wormhole/supported-environments/evm#base)


## Assessed type

Other