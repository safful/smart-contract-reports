## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Public functions to external](https://github.com/code-423n4/2022-01-livepeer-findings/issues/21) 

# Handle

robee


# Vulnerability details

The following functions could be set external to save gas and improve code quality. 
External call cost is less expensive than of public functions. 

        L2LPTDataCache.sol, l1CirculatingSupply
        L2LPTGateway.sol, outboundTransfer
        DelegatorPool.sol, initialize
        IController.sol, getContract
        Manager.sol, constructor
        BridgeMinter.sol, constructor
        BridgeMinter.sol, getController


