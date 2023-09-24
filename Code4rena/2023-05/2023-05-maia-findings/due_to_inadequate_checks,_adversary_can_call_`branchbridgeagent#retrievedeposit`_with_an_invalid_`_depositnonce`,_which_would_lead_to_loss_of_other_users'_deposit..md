## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-08

# [Due to inadequate checks, Adversary can call `BranchBridgeAgent#retrieveDeposit` with an invalid `_depositNonce`, which would lead to loss of other users' deposit.](https://github.com/code-423n4/2023-05-maia-findings/issues/688) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L433


# Vulnerability details

## Impact
Attacker will cause user's funds to be collected and locked on Branch chain without it being recorded on root chain.


## Proof of Concept
Anyone can call `BranchBridgeAgent#retrieveDeposit`, with an invalid `_depositNonce`:

```solidity
function retrieveDeposit(
        uint32 _depositNonce
    ) external payable lock requiresFallbackGas {
        //Encode Data for cross-chain call.
        bytes memory packedData = abi.encodePacked(
            bytes1(0x08),
            _depositNonce,
            msg.value.toUint128(),
            uint128(0)
        );

        //Update State and Perform Call
        _sendRetrieveOrRetry(packedData);
    }
```

If for example, global depositNonce is x, attacker can call `retrieveDeposit(x+y)`.
`RootBridgeAgent#anyExecute` will be called, and the executionHistory for the depositNonce that the attacker specified would be updated to true:

```solidity
    function anyExecute(bytes calldata data){
        ...
    /// DEPOSIT FLAG: 8 (retrieveDeposit)
    else if (flag == 0x08) {
    //Get nonce
    uint32 nonce = uint32(bytes4(data[1:5]));

    //Check if tx has already been executed
    if (!executionHistory[fromChainId][uint32(bytes4(data[1:5]))]) {
        //Toggle Nonce as executed
        executionHistory[fromChainId][nonce] = true;

        //Retry failed fallback
        (success, result) = (false, "");
    } else {
        _forceRevert();
        //Return true to avoid triggering anyFallback in case of `_forceRevert()` failure
        return (true, "already executed tx");
    }
    }
    ...
    }
```

This means that when a user makes a deposit on that BranchBridgeAgent and his Deposit gets assigned a depositNonce which attacker previously called retrieveDeposit for, his tokens would be collected on that BranchBridgeAgent, but would not succeed on RootBridgeAgent because executionHistory for that depositNonce has already been maliciously set to true.

### Attack Scenario

- current global deposit nonce is 50
- attacker calls retrieveDeposit(60), which would update executionHistory of depositNonce 60 to true on Root chain
- when a user tries to call any of the functions(say callOutAndBridge), and gets assigned depositNonce of 60, it won't be executed on root chain because executionHistory for depositNonce 60 is already set to true
- User won't also be able to claim his tokens because anyFallback was not triggered. So he has lost his deposit.

## Tools Used
Manual Review

## Recommended Mitigation Steps
A very simple and effective solution is to ensure that in the `BranchBridgeAgent#retrieveDepoit` function, `msg.sender==getDeposit[_depositNonce].owner` just like it was done in [`BranchBridgeAgent#retryDeposit`](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L327)



## Assessed type

Invalid Validation