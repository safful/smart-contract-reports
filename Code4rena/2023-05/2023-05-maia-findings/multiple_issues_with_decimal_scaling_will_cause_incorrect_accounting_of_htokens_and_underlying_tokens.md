## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-05

# [Multiple issues with decimal scaling will cause incorrect accounting of hTokens and underlying tokens](https://github.com/code-423n4/2023-05-maia-findings/issues/758) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchPort.sol#L388-L390
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1340-L1342
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/ArbitrumBranchPort.sol#L52-L54
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/ArbitrumBranchBridgeAgent.sol#L104
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L269
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L313
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L696
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L745


# Vulnerability details


`_normalizeDecimals()` and `_denormalizeDecimals()` are used to handle non-18 decimals tokens when bridging deposit, by scaling them to a normalized 18 decimals form for hToken accounting, and then denormalizing to the token's decimals when interacting with the underlying token.

However, there are 3 issues as follows,
1. implementations of `_normalizeDecimals()` and `_denormalizeDecimals()` are reversed. 
2. `_denormalizeDecimals()` is missing in `ArbitrumBranchPort.depositToPort()`.
3. `_normalizeDecimals()` is missing in functions within `BranchBridgeAgent`.

These issues will cause an incorrect accounting of hTokens and underlying tokens in the system. 


## Impact
Incorrect decimal scaling will lead to loss of fund as the amount deposited and withdrawn for bridging will be inaccurate, which can be abused by an attacker or result in users inccuring losses.

For example, an attacker can abuse the `ArbitrumBranchPort.depositToPort()` issue and steal from the system by first depositing a token that has more than 18 decimals, as the attacker will receive more hTokens than the deposited underlying token amount. The attacker can then make a profit by withdrawing from the port with the excess hTokens.

On the other hand, if the underlying token is less than 18 decimals, the depositor can inccur losses as the amount of underlying tokens deposited will be more than the amount of hTokens received.

## Detailed Explanation

### Issue 1 
`BranchBridgeAgent._normalizeDecimals()` and `BranchPort._denormalizeDecimals()` (shown vbelow) are incorrect as they are implemented in a reversed manner, such that `_denormalizeDecimals()` is normalizing to 18 decimals while `_normalizeDecimals()` is denormalizing to the underlying token decimals.

The result is that for tokens with > 18 decimals, `_normalizeDecimals()` will overscale the decimals, while for tokens with < 18 decimals, `_normalizeDecimals()` will underscale the decimals.

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1340-L1342
```Solidity
    function _normalizeDecimals(uint256 _amount, uint8 _decimals) internal pure returns (uint256) {
        return _decimals == 18 ? _amount : _amount * (10 ** _decimals) / 1 ether;
    }
```

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchPort.sol#L388-L390
```Solidity
    function _denormalizeDecimals(uint256 _amount, uint8 _decimals) internal pure returns (uint256) {
        return _decimals == 18 ? _amount : _amount * 1 ether / (10 ** _decimals);
    }
```


### Issue 2
`ArbitrumBranchPort.depositToPort()` is mssing `_denormalizeDecimals()` to scale back the decimals of the underlying token amount before transfering. This will cause a wrong amount of the underlying token to be transfered.

As shown below, `ArbitrumBranchBridgeAgent.depositToPort()` has normalized the `amount` to 18 decimals before passing into `ArbitrumBranchPort.depositToPort()`.

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/ArbitrumBranchBridgeAgent.sol#L104
```Solidity
    function depositToPort(address underlyingAddress, uint256 amount) external payable lock {

        //@audit - amount is normalized to 18 decimals here
        IArbPort(localPortAddress).depositToPort(
            msg.sender, msg.sender, underlyingAddress, _normalizeDecimals(amount, ERC20(underlyingAddress).decimals())
        );
    }
```

That means the `_deposit` amount for `ArbitrumBranchPort.depositToPort()` (see below) will be incorrect as it is not denormalized back to the underlying token's decimal, causing the wrong value to be transfered from the depositor.

If the underlying token is more than 18 decimals, the depositor will transfer less underlying tokens than the hToken received resulting in excess hTokens. The depositor can then call `withdrawFromPort()` to receive more underlying tokens than deposited.

If the underlying token is less than 18 decimals, that will inflate the amount to be transferred from the depositor, causing the depositor to deposit more underlying token than the amount of hToken received. The depositor will incurr a loss when withdrawing from the port.

Instead, the `_deposit` should be denormalized in `ArbitrumBranchPort.depositToPort()` when passing to `_underlyingAddress.safeTransferFrom()`, so that it is scaled back to the underlying token's decimals when transfering. 

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/ArbitrumBranchPort.sol#L52-L54
```Solidity
    function depositToPort(address _depositor, address _recipient, address _underlyingAddress, uint256 _deposit)
        external
        requiresBridgeAgent
    {
        address globalToken = IRootPort(rootPortAddress).getLocalTokenFromUnder(_underlyingAddress, localChainId);
        if (globalToken == address(0)) revert UnknownUnderlyingToken();

        //@audit - the amount of underlying token should be denormalized first before transfering
        _underlyingAddress.safeTransferFrom(_depositor, address(this), _deposit);

        IRootPort(rootPortAddress).mintToLocalBranch(_recipient, globalToken, _deposit);
    }
```

### Issue 3

In `BranchBridgeAgent`, the deposit amount passed into `_depositAndCall()` and `_depositAndCallMultiple()` are missing `_normalizeDecimals()`.
Example below shows `callOutSignedAndBridge()`, but the issue is also present in `callOutAndBridge()`, `callOutSignedAndBridgeMultiple()`, `callOutAndBridgeMultiple()`.

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L269
```Solidity
    function callOutSignedAndBridge(bytes calldata _params, DepositInput memory _dParams, uint128 _remoteExecutionGas)
        external
        payable
        lock
        requiresFallbackGas
    {
        //Encode Data for cross-chain call.
        bytes memory packedData = abi.encodePacked(
            bytes1(0x05),
            msg.sender,
            depositNonce,
            _dParams.hToken,
            _dParams.token,
            _dParams.amount,
            _normalizeDecimals(_dParams.deposit, ERC20(_dParams.token).decimals()),
            _dParams.toChain,
            _params,
            msg.value.toUint128(),
            _remoteExecutionGas
        );

        //Wrap the gas allocated for omnichain execution.
        wrappedNativeToken.deposit{value: msg.value}();

        //Create Deposit and Send Cross-Chain request
        _depositAndCall(
            msg.sender,
            packedData,
            _dParams.hToken,
            _dParams.token,
            _dParams.amount,
            //@audit - the deposit amount of underlying token should be noramlized first
            _dParams.deposit,
            msg.value.toUint128()
        );
    }
```
And that will affect `_createDepositSingle()` and `_createDepositMultiple()`, leading to incorrect decimals for `IPort(localPortAddress).bridgeOut()`. That will affect hToken burning and deposit of underlying tokens.

At the same time, the deposits to be stored in `getDeposit[]` is also not normalized, causing a mis-match of decimals when `clearToken()` is called via `redeemDeposit()`.

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L857-L891
```Solidity
    function _createDepositSingle(
        address _user,
        address _hToken,
        address _token,
        uint256 _amount,
        uint256 _deposit,
        uint128 _gasToBridgeOut
    ) internal {
        //Deposit / Lock Tokens into Port
        IPort(localPortAddress).bridgeOut(_user, _hToken, _token, _amount, _deposit);

        //Deposit Gas to Port
        _depositGas(_gasToBridgeOut);

        // Cast to dynamic memory array
        address[] memory hTokens = new address[](1);
        hTokens[0] = _hToken;
        address[] memory tokens = new address[](1);
        tokens[0] = _token;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = _amount;
        uint256[] memory deposits = new uint256[](1);
        deposits[0] = _deposit;

        // Update State
        getDeposit[_getAndIncrementDepositNonce()] = Deposit({
            owner: _user,
            hTokens: hTokens,
            tokens: tokens,
            amounts: amounts,
            //@audit the deposits stored is not normalized, causing a mis-match of decimals when `clearToken()` is called via `redeemDeposit()`
            deposits: deposits,
            status: DepositStatus.Success,
            depositedGas: _gasToBridgeOut
        });
    }
```


## Recommended Mitigation Steps

1. Switch the implementation of `_normalizeDecimals()` to `_denormalizeDecimals()` and vice versa.

2. Add `_denormalizeDecimals()` to `ArbitrumBranchPort.depositToPort()` when calling `IRootPort(rootPortAddress).mintToLocalBranch()`.

3. Utilize `_normalizeDecimals()` for when passing deposit amount to  `_depositAndCall()` and `_depositAndCallMultiple()` within `BranchBridgeAgent`.








## Assessed type

Decimal