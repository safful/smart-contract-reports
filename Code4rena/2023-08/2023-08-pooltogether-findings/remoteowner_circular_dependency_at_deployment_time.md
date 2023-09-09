## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-01

# [RemoteOwner circular dependency at deployment time](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/147) 

# Lines of code

https://github.com/GenerationSoftware/remote-owner/blob/9c093dbd36c1f18ab7083549d10ac601d91630df/src/RemoteOwner.sol#L58
https://github.com/GenerationSoftware/remote-owner/blob/9c093dbd36c1f18ab7083549d10ac601d91630df/src/RemoteOwner.sol#L120
https://github.com/GenerationSoftware/remote-owner/blob/9c093dbd36c1f18ab7083549d10ac601d91630df/src/RemoteOwner.sol#L96-L99
https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngAuctionRelayerRemoteOwner.sol#L47
https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngAuctionRelayerRemoteOwner.sol#L64


# Vulnerability details

## Impact
The `RemoteOwner.sol` contract has a security measure that ensures the sender from the remote/origin chain was the origin chain owner (i.e. a `RngAuctionRelayerRemoteOwner.sol` deployment), and this address is set at deployment time in the constructor. The `RngAuctionRelayerRemoteOwner` contract also has a security measure to ensure that messages are only dispatched across chain to the `RemoteOwner` contract deployed in the destination chain, and this address is set at deployment time in the constructor.

Clearly there is a circular dependency here that means the deployment phase will fail. There is a `setOriginChainOwner` method on the `RemoteOwner` contract, however this can only be called by the address on the origin chain specified in the constructor. This method is never called from the origin chain either. In summary, the circular dependency prevents the contracts from being deployed and ever initialised properly.

It is possible that there is an intermediary `__originChainOwner` used in the constructor when deploying `RemoteOwner`, but since I couldn't find any deployment scripts to verify this I have assumed that this is an unintended bug. The severity of this report depends on whether or not this was intended.

## Proof of Concept
In the `RemoteOwner.sol` contract, the origin chain owner is set in the constructor:

```
  constructor(
    uint256 originChainId_,
    address executor_,
    address __originChainOwner
  ) ExecutorAware(executor_) {
    if (originChainId_ == 0) revert OriginChainIdZero();
    _originChainId = originChainId_;
    _setOriginChainOwner(__originChainOwner);
  }
```

Any calls to the `RemoteOwner` contract are protected by the `_checkSender` view:

```
  function _checkSender() internal view {
    if (!isTrustedExecutor(msg.sender)) revert LocalSenderNotExecutor(msg.sender);
    if (_fromChainId() != _originChainId) revert OriginChainIdUnsupported(_fromChainId());
    if (_msgSender() != address(_originChainOwner)) revert OriginSenderNotOwner(_msgSender());
  }
```

Now, if we have a look at the `RngAuctionRelayerRemoteOwner.sol` contract, we can see that the remote owner address is also specified in the constructor:

```
    constructor(
        RngAuction _rngAuction,
        ISingleMessageDispatcher _messageDispatcher,
        RemoteOwner _remoteOwner,
        uint256 _toChainId
    ) RngAuctionRelayer(_rngAuction) {
        messageDispatcher = _messageDispatcher;
        account = _remoteOwner;
        toChainId = _toChainId;
    }
```

This `account` address is now hard-coded and used with any calls to `relay`:

```
function relay(
        IRngAuctionRelayListener _remoteRngAuctionRelayListener,
        address rewardRecipient
    ) external returns (bytes32) {
        bytes memory listenerCalldata = encodeCalldata(rewardRecipient);
        bytes32 messageId = messageDispatcher.dispatchMessage(
            toChainId,
            address(account),
            RemoteOwnerCallEncoder.encodeCalldata(address(_remoteRngAuctionRelayListener), 0, listenerCalldata)
        );
        emit RelayedToDispatcher(rewardRecipient, messageId);
        return messageId;
    }
```

There is a circular dependency here due to the reliance on specifying the relevant addresses in the constructor.

## Tools Used
Manual review

## Recommended Mitigation Steps
To remove the circular dependency and reliance on a very specific deployment pipeline that requires a specific call from a remote chain address, I would make the following change to the `RemoteOwner` contract:

```
diff --git a/src/RemoteOwner.sol b/src/RemoteOwner.sol
index 7c1de6d..a6cb8f1 100644
--- a/src/RemoteOwner.sol
+++ b/src/RemoteOwner.sol
@@ -55,7 +55,6 @@ contract RemoteOwner is ExecutorAware {
   ) ExecutorAware(executor_) {
     if (originChainId_ == 0) revert OriginChainIdZero();
     _originChainId = originChainId_;
-    _setOriginChainOwner(__originChainOwner);
   }
 
   /* ============ External Functions ============ */
@@ -94,7 +93,7 @@ contract RemoteOwner is ExecutorAware {
    *      If the transaction get front-run at deployment, we can always re-deploy the contract.
    */
   function setOriginChainOwner(address _newOriginChainOwner) external {
-    _checkSender();
+    require(_originChainOwner == address(0), "Already initialized");
     _setOriginChainOwner(_newOriginChainOwner);
   }
 
```

However I can understand how the current deployment pipeline functionality would make it harder to frontrun `setOriginChainOwner` if this was done deliberately, so alternatively you could keep the functionality the same but just provide better comments.


## Assessed type

Other