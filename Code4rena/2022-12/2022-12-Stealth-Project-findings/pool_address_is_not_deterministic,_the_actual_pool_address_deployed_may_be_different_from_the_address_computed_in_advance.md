## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-02

# [Pool address is not deterministic, the actual Pool address deployed may be different from the address computed in advance](https://github.com/code-423n4/2022-12-Stealth-Project-findings/issues/27) 

# Lines of code

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/libraries/Deployer.sol#L12-L36


# Vulnerability details

## Impact
Users and contracts associated with Maverick may result in errors or token losses due to use a wrong Pool address.

## Proof of Concept
Pools ared deployed in a `create2` manner just like uniswap. This is used to ensure that the pool contract address is deterministic and guaranteed, because in some scenarios it is necessary to know the address of the pool in advance and perform some operations.

However, the address of the Pool contract in the current implementation is not fully deterministic.

Let's see the code of [Deployer#deploy()](https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/libraries/Deployer.sol#L24):

```solidity
function deploy(
    uint256 _fee,
    uint256 _tickSpacing,
    int32 _activeTick,
    uint16 _lookback,
    uint16 _protocolFeeRatio,
    IERC20 _tokenA,
    IERC20 _tokenB,
    uint256 _tokenAScale,
    uint256 _tokenBScale,
    IPosition _position
) external returns (IPool pool) {
    pool = new Pool{salt: keccak256(abi.encode(_fee, _tickSpacing, _lookback, _tokenA, _tokenB))}(
        _fee,
        _tickSpacing,
        _activeTick,
        _lookback,
        _protocolFeeRatio,
        _tokenA,
        _tokenB,
        _tokenAScale,
        _tokenBScale,
        _position
    );
```

Any change in `_fee, _tickSpacing, _activeTick, _lookback, _protocolFeeRatio, _tokenA, _tokenB, _tokenAScale, _tokenBScale, _position` will result in a different contract address.
This is because they are all used as the Pool's constructor arguments which will be used to calculate the contract address.
> see [Salted contract creations / create2](https://docs.soliditylang.org/en/latest/control-structures.html#salted-contract-creations-create2)

Of these parameters, only `_fee, _tickSpacing, _lookback, _tokenA, _tokenB` are used to determine a pool:

* they are used to [lookup](https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Factory.sol#L46) a pool
  ```solidity
  function lookup(uint256 _fee, uint256 _tickSpacing, uint16 _lookback, IERC20 _tokenA, IERC20 _tokenB) external view override returns (IPool pool) 
  ```
* they are the keys of the [pools mapping](https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Factory.sol#L16)
  ```
  mapping(uint256 => mapping(uint256 => mapping(uint32 => mapping(IERC20 => mapping(IERC20 => IPool))))) pools;

  pools[_fee][_tickSpacing][_lookback][_tokenA][_tokenB] = pool;
  ```

Other parameters are the properties or state of the pool, they do not determine a pool.

Some of them may be set to or modified to any value, which leads to the loss of determinism and guarantee of the poll address:
* `_activeTick`: may be set to any value by creator when [`create()`](https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Factory.sol#L58), :
* `_protocolFeeRatio`: may be set to any value at any time by factory owner. see [setProtocolFeeRatio](https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Factory.sol#L33)
* `_tokenAScale, _tokenBScale`: may be set to any value at any time by token owner (if the token is  settable or upgradable)

## Tools Used
VS Code

## Recommended Mitigation Steps

I recommend using a similar approach to uniswap, which is to remove all constructor parameters of the Pool, save them in factory's storage, and retrieve them by calling the factory in constructor.

Related code in uniswap:
[UniswapV3PoolDeployer.deploy](https://github.com/Uniswap/v3-core/blob/05c10bf6d547d6121622ac51c457f93775e1df09/contracts/UniswapV3PoolDeployer.sol#L27)
[UniswapV3Pool constructor](https://github.com/Uniswap/v3-core/blob/05c10bf6d547d6121622ac51c457f93775e1df09/contracts/UniswapV3Pool.sol#L117-L123)

Recommended implementation:
```solidity
diff --git a/maverick-v1/contracts/libraries/Deployer.sol b/maverick-v1/contracts/libraries/Deployer.sol
deleted file mode 100644
index 674602e..0000000
--- a/maverick-v1/contracts/libraries/Deployer.sol
+++ /dev/null
@@ -1,37 +0,0 @@
-// SPDX-License-Identifier: UNLICENSED
-pragma solidity ^0.8.0;
-
-import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
-
-import "../models/Pool.sol";
-import "../interfaces/IFactory.sol";
-import "../interfaces/IPool.sol";
-import "../interfaces/IPosition.sol";
-
-library Deployer {
-    function deploy(
-        uint256 _fee,
-        uint256 _tickSpacing,
-        int32 _activeTick,
-        uint16 _lookback,
-        uint16 _protocolFeeRatio,
-        IERC20 _tokenA,
-        IERC20 _tokenB,
-        uint256 _tokenAScale,
-        uint256 _tokenBScale,
-        IPosition _position
-    ) external returns (IPool pool) {
-        pool = new Pool{salt: keccak256(abi.encode(_fee, _tickSpacing, _lookback, _tokenA, _tokenB))}(
-            _fee,
-            _tickSpacing,
-            _activeTick,
-            _lookback,
-            _protocolFeeRatio,
-            _tokenA,
-            _tokenB,
-            _tokenAScale,
-            _tokenBScale,
-            _position
-        );
-    }
-}
diff --git a/maverick-v1/contracts/models/Factory.sol b/maverick-v1/contracts/models/Factory.sol
index d33d91f..afb7e3b 100644
--- a/maverick-v1/contracts/models/Factory.sol
+++ b/maverick-v1/contracts/models/Factory.sol
@@ -7,11 +7,11 @@ import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
 import "../models/Pool.sol";
 import "../interfaces/IFactory.sol";
 import "../interfaces/IPool.sol";
+import "../interfaces/IPoolDeployer.sol";
 import "../interfaces/IPosition.sol";
-import "../libraries/Deployer.sol";
 import "../libraries/Math.sol";
 
-contract Factory is IFactory {
+contract Factory is IFactory, IPoolDeployer  {
     /// @notice mapping elements are [fee][tickSpacing][lookback][tokenA][tokenB]
     mapping(uint256 => mapping(uint256 => mapping(uint32 => mapping(IERC20 => mapping(IERC20 => IPool))))) pools;
     /// @notice mapping of pools created by this factory
@@ -22,6 +22,7 @@ contract Factory is IFactory {
     uint16 public protocolFeeRatio;
     IPosition public immutable position;
     address public owner;
+    IPoolDeployer.Parameters private parameters;
 
     constructor(uint16 _protocolFeeRatio, IPosition _position) {
         require(_protocolFeeRatio <= ONE_3_DECIMAL_SCALE, "Factory:PROTOCOL_FEE_CANNOT_EXCEED_ONE");
@@ -48,6 +49,10 @@ contract Factory is IFactory {
         return pools[_fee][_tickSpacing][_lookback][tokenA][tokenB];
     }
 
+    function getParameters() external view override returns (IPoolDeployer.Parameters memory) {
+        return parameters;
+    }
+
     /// @notice creates new pool
     /// @param _fee is a rate in prbmath 60x18 decimal format
     /// @param _tickSpacing  1.0001^tickSpacing is the bin width
@@ -63,18 +68,20 @@ contract Factory is IFactory {
 
         require(pools[_fee][_tickSpacing][_lookback][_tokenA][_tokenB] == IPool(address(0)), "Factory:POOL_ALREADY_EXISTS");
 
-        pool = Deployer.deploy(
-            _fee,
-            _tickSpacing,
-            _activeTick,
-            _lookback,
-            protocolFeeRatio,
-            _tokenA,
-            _tokenB,
-            Math.scale(IERC20Metadata(address(_tokenA)).decimals()),
-            Math.scale(IERC20Metadata(address(_tokenB)).decimals()),
-            position
-        );
+        parameters = IPoolDeployer.Parameters({
+            _fee: _fee,
+            _tickSpacing: _tickSpacing,
+            _activeTick: _activeTick,
+            _lookback: _lookback,
+            _protocolFeeRatio: protocolFeeRatio,
+            _tokenA: address(_tokenA),
+            _tokenB: address(_tokenB),
+            _tokenAScale: Math.scale(IERC20Metadata(address(_tokenA)).decimals()),
+            _tokenBScale: Math.scale(IERC20Metadata(address(_tokenB)).decimals()),
+            _position: address(position)
+        });
+        pool = new Pool{salt: keccak256(abi.encode(_fee, _tickSpacing, _lookback, _tokenA, _tokenB))}();
+        delete parameters;
         isFactoryPool[pool] = true;
 
         emit PoolCreated(address(pool), _fee, _tickSpacing, _activeTick, _lookback, protocolFeeRatio, _tokenA, _tokenB);
diff --git a/maverick-v1/contracts/models/Pool.sol b/maverick-v1/contracts/models/Pool.sol
index 9cc0680..065d4fd 100644
--- a/maverick-v1/contracts/models/Pool.sol
+++ b/maverick-v1/contracts/models/Pool.sol
@@ -15,6 +15,7 @@ import "../libraries/SafeERC20Min.sol";
 import "../interfaces/ISwapCallback.sol";
 import "../interfaces/IAddLiquidityCallback.sol";
 import "../interfaces/IPool.sol";
+import "../interfaces/IPoolDeployer.sol";
 import "../interfaces/IPoolAdmin.sol";
 import "../interfaces/IFactory.sol";
 import "../interfaces/IPosition.sol";
@@ -57,29 +58,19 @@ contract Pool is IPool, IPoolAdmin {
     uint128 public override binBalanceA;
     uint128 public override binBalanceB;
 
-    constructor(
-        uint256 _fee,
-        uint256 _tickSpacing,
-        int32 _activeTick,
-        uint16 _lookback,
-        uint16 _protocolFeeRatio,
-        IERC20 _tokenA,
-        IERC20 _tokenB,
-        uint256 _tokenAScale,
-        uint256 _tokenBScale,
-        IPosition _position
-    ) {
+    constructor() {
         factory = IFactory(msg.sender);
-        fee = _fee;
-        state.protocolFeeRatio = _protocolFeeRatio;
-        tickSpacing = _tickSpacing;
-        state.activeTick = _activeTick;
-        twa.lookback = _lookback;
-        tokenA = _tokenA;
-        tokenB = _tokenB;
-        position = _position;
-        tokenAScale = _tokenAScale;
-        tokenBScale = _tokenBScale;
+        IPoolDeployer.Parameters memory params = IPoolDeployer(msg.sender).getParameters();
+        fee = params._fee;
+        state.protocolFeeRatio = params._protocolFeeRatio;
+        tickSpacing = params._tickSpacing;
+        state.activeTick = params._activeTick;
+        twa.lookback = params._lookback;
+        tokenA = IERC20(params._tokenA);
+        tokenB = IERC20(params._tokenB);
+        position = IPosition(params._position);
+        tokenAScale = params._tokenAScale;
+        tokenBScale = params._tokenBScale;
         state.status = NO_EMERGENCY_UNLOCKED;
     }
 
diff --git a/maverick-v1/contracts/test/TestFactoryPool.sol b/maverick-v1/contracts/test/TestFactoryPool.sol
index 93ab11d..049f933 100644
--- a/maverick-v1/contracts/test/TestFactoryPool.sol
+++ b/maverick-v1/contracts/test/TestFactoryPool.sol
@@ -6,14 +6,20 @@ import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
 import "../interfaces/ISwapCallback.sol";
 import "../interfaces/IAddLiquidityCallback.sol";
 import "../models/Pool.sol";
+import "../interfaces/IPoolDeployer.sol";
 import "../interfaces/IPosition.sol";
 import "../libraries/Bin.sol";
 
-contract TestDeployPool is ISwapCallback, IAddLiquidityCallback {
+contract TestDeployPool is ISwapCallback, IAddLiquidityCallback, IPoolDeployer {
     Pool public pool;
 
     address public owner;
 
+    IPoolDeployer.Parameters private parameters;
+    function getParameters() external view override returns (IPoolDeployer.Parameters memory) {
+        return parameters;
+    }
+
     constructor(
         uint256 _fee,
         uint256 _sqrtTickSpacing,
@@ -27,7 +33,20 @@ contract TestDeployPool is ISwapCallback, IAddLiquidityCallback {
         IPosition _position
     ) {
         owner = msg.sender;
-        pool = new Pool(_fee, _sqrtTickSpacing, _activeTick, _lookback, _protocolFeeRatio, _tokenA, _tokenB, Math.scale(_tokenADecimals), Math.scale(_tokenBDecimals), _position);
+        parameters = IPoolDeployer.Parameters({
+            _fee: _fee,
+            _tickSpacing: _sqrtTickSpacing,
+            _activeTick: _activeTick,
+            _lookback: _lookback,
+            _protocolFeeRatio: _protocolFeeRatio,
+            _tokenA: address(_tokenA),
+            _tokenB: address(_tokenB),
+            _tokenAScale: Math.scale(_tokenADecimals),
+            _tokenBScale: Math.scale(_tokenBDecimals),
+            _position: address(_position)
+        });
+        pool = new Pool();
+        delete parameters;
     }
 
     struct SwapCallbackData {
diff --git a/maverick-v1/test/shared/deploy.ts b/maverick-v1/test/shared/deploy.ts
index f421e90..40793d1 100644
--- a/maverick-v1/test/shared/deploy.ts
+++ b/maverick-v1/test/shared/deploy.ts
@@ -113,7 +113,7 @@ export const deployFactory = async ({
     "Factory",
     mock,
     {
-      Deployer: (await deployDeployer({})).address,
+      // Deployer: (await deployDeployer({})).address,
     },
     protocolFeeRatio,
     position
```
