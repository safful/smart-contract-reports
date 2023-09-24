## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-09

# [`RootBridgeAgent->CheckParamsLib#checkParams` does not check that `_dParams.token` is underlying of `_dParams.hToken`](https://github.com/code-423n4/2023-05-maia-findings/issues/687) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L58


# Vulnerability details

## Impact
A malicious user would make a deposit specifying an hToken of a high value(say hEther), and a depositToken of relatively lower value(say USDC), and for that user, RootBridgeAgent would increment his hToken balance by the amount of depositTokens he sent

## Proof of Concept
Here is the `checkParams` function:

```solidity
function checkParams(address _localPortAddress, DepositParams memory _dParams, uint24 _fromChain)
        internal
        view
        returns (bool)
    {
        if (
            (_dParams.amount < _dParams.deposit) //Deposit can't be greater than amount.
                || (_dParams.amount > 0 && !IPort(_localPortAddress).isLocalToken(_dParams.hToken, _fromChain)) //Check local exists.
                || (_dParams.deposit > 0 && !IPort(_localPortAddress).isUnderlyingToken(_dParams.token, _fromChain)) //Check underlying exists.
        ) {
            return false;
        }
        return true;
    }
```

The function performs 3 checks:

- \_dParams.amount must be less than or equal to \_dParams.deposit
- If \_dParams.amount>0, \_dParams.hToken must be a valid localToken
- If \_dParams.deposit>0, \_dParams.token must be a valid underlying token.

The PROBLEM is that the check only requires that `getLocalTokenFromUnder[_dParams.token]`!=`address(0)`, but does not check that `getLocalTokenFromUnder[_dParams.token]`==`_dParams.hToken`:

```solidity
    function isUnderlyingToken(
        address _underlyingToken,
        uint24 _fromChain
    ) external view returns (bool) {
        return
            getLocalTokenFromUnder[_underlyingToken][_fromChain] != address(0);
    }
```

The checkParams function is used in the `RootBridgeAgent#bridgeIn` function.

This allows a user to call `BranchBridgeAgent#callOutAndBridge` with a `hToken` and `token` that are not related

## ATTACK SCENARIO

- Current price of Ether is 1800USDC
- RootBridgeAgent is deployed on Arbitrum
- BranchBridgeAgent for Ethereum mainnet has two local tokens recorded in RootBridgeAgent:
  - hEther(whose underlying is Ether)
  - hUSDC(whose underlying is USDC)
- Alice calls `BranchBridgeAgent#callOutAndBridge` on ethereum with the following as DepositInput(\_dParams):
  - hToken(address of local hEther)
  - token(address of USDC)
  - amount(0)
  - deposit(10)
  - toChain(42161)
- `BranchPort#bridgeOut` transfers 10 USDC from user to BranchPort, and anyCall call is made to RootBridgeAgent
- `RootBridgeAgent#bridgeIn` is called which calls `CheckParamsLib.checkParams`
  - `checkParams` verifies that \_dParams.amount(0) is less than or equal to \_dParams.deposit(10)✅
  - verifies that \_dParams.hToken(hEther) is a valid localToken✅
  - verifies that \_dParams.token(USDC) is a valid underlying token(i.e. its local token is non zero)✅
- `RootBridgeAgent#bridgeIn` calls `RootPort#bridgeToRoot` which mints 10 global hEther to user `if (_deposit > 0) mint(_recipient, _hToken, _deposit, _fromChainId);`
- With just 10 USDC, user has been able to get 10 ether(18000USDC) worth of funds on root chain.

Execution flow:
`BranchBridgeAgent#callOutAndBridge`->`BranchBridgeAgent#_callOutAndBridge`->`BranchBridgeAgent#_depositAndCall`->`BranchBridgeAgent#_performCall`->`RootBridgeAgent#anyExecute`->`RootBridgeAgentExecutor#executeWithDeposit`->`RootBridgeAgentExecutor#_bridgeIn`->`RootBridgeAgent#bridgeIn`

## Tools Used
Manual Review

## Recommended Mitigation Steps
Currently, the protocol only checks if the token is recognized by rootport as an underlying token by checking that the registered local token for `_dParams.token` is not zero address.

- Instead of that, it would be more effective to check that the registered local token for `_dParams.token` is equal to `_dParams.hToken`.
- Some sanity checks may also be done on DepositInput(\_dParams) in BranchBridgeAgent. Although this is not necessary.



## Assessed type

Invalid Validation