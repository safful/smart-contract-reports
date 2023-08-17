## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-01

# [PirexGmx.initiateMigration can be blocked](https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/61) 

# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/main/src/PirexGmx.sol#L921-L935


# Vulnerability details

## Impact
PirexGmx.initiateMigration can be blocked so contract will not be able to migrate his funds to another contract using gmx.

## Proof of Concept
PirexGmx was designed with the thought that the current contract can be changed with another during migration.
PirexGmx.initiateMigration is the first point in this long process. 
https://github.com/code-423n4/2022-11-redactedcartel/blob/main/src/PirexGmx.sol#L921-L935
```solidity
    function initiateMigration(address newContract)
        external
        whenPaused
        onlyOwner
    {
        if (newContract == address(0)) revert ZeroAddress();


        // Notify the reward router that the current/old contract is going to perform
        // full account transfer to the specified new contract
        gmxRewardRouterV2.signalTransfer(newContract);


        migratedTo = newContract;


        emit InitiateMigration(newContract);
    }
```
As you can see `gmxRewardRouterV2.signalTransfer(newContract);` is called to start migration.
This is the code of signalTransfer function
https://arbiscan.io/address/0xA906F338CB21815cBc4Bc87ace9e68c87eF8d8F1#code#F1#L282
```solidity
    function signalTransfer(address _receiver) external nonReentrant {
        require(IERC20(gmxVester).balanceOf(msg.sender) == 0, "RewardRouter: sender has vested tokens");
        require(IERC20(glpVester).balanceOf(msg.sender) == 0, "RewardRouter: sender has vested tokens");

        _validateReceiver(_receiver);
        pendingReceivers[msg.sender] = _receiver;
    }
```

As you can see the main condition to start migration is that PirexGmx doesn't control any gmxVester and glpVester tokens.
So attacker can [deposit](https://arbiscan.io/address/0xa75287d2f8b217273e7fcd7e86ef07d33972042e#code#F1#L117) and receive such tokens and then just transfer tokens directly to PirexGmx.
As a result migration will never be possible as there is no possibility for PirexGmx to burn those gmxVester and glpVester tokens.

Also in same way migration receiver can also be blocked.
## Tools Used
VsCode
## Recommended Mitigation Steps
Think about how to make contract to be sure that he doesn't control any gmxVester and glpVester tokens before migration.