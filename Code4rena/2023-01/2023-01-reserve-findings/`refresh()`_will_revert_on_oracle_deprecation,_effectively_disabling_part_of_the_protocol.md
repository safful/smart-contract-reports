## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-17

# [`refresh()` will revert on Oracle deprecation, effectively disabling part of the protocol](https://github.com/code-423n4/2023-01-reserve-findings/issues/234) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/plugins/assets/Asset.sol#L102
https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/plugins/assets/FiatCollateral.sol#L149
https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/plugins/assets/OracleLib.sol#L14-L31


# Vulnerability details




The `Asset.refresh()` function calls `tryPrice()` and catches all errors except errors with empty data.
As explained in the [docs](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/docs/solidity-style.md#catching-empty-data) the reason empty errors aren't caught is in order to prevent an attacker from failing the `tryPrice()` intentionally by running it out of gas.
However, an error with empty data isn't thrown only in case of out of gas, in the current way that Chainlink deprecates oracles (by setting `aggregator` to the zero address) a deprecated oracle would also throw an empty error.



## Impact
Any function that requires refreshing the assets will fail to execute (till the asset is replaced in the asset registry, passing the proposal via governance would usually take 7 days), that includes:
* Issuance
* Vesting
* Redemption
* Auctions (`manageTokens()`)
* `StRSR.withdraw()`

## Proof of Concept

The [docs](https://github.com/reserve-protocol/protocol/blob/d224c14c398d2727d39d133aa7511e1e6b161833/docs/recollateralization.md#:~:text=If%20an%20asset%27s%20oracle%20goes%20offline%20forever%2C%20its%20lotPrice()%20will%20eventually%20reach%20%5B0%2C%200%5D%20and%20the%20protocol%20will%20completely%20stop%20trading%20this%20asset.) imply in case of deprecation the protocol is expected continue to operate:
> If an asset's oracle goes offline forever, its lotPrice() will eventually reach [0, 0] and the protocol will completely stop trading this asset.

The [docs](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/docs/collateral.md#refresh-should-never-revert) also clearly state that '`refresh()` should never revert'


I've tracked a few Chainlink oracles that were deprecated on the Polygon network on Jan 11, the PoC below tests an `Asset.refresh()` call with a deprecated oracle.



File: `test/plugins/Deprecated.test.ts`
```typescript
import { Wallet, ContractFactory } from 'ethers'
import { ethers, network, waffle } from 'hardhat'
import { IConfig } from '../../common/configuration'
import { bn, fp } from '../../common/numbers'
import {
  Asset,
  ATokenFiatCollateral,
  CTokenFiatCollateral,
  CTokenMock,
  ERC20Mock,
  FiatCollateral,
  IAssetRegistry,
  RTokenAsset,
  StaticATokenMock,
  TestIBackingManager,
  TestIRToken,
  USDCMock,
} from '../../typechain'
import {
  Collateral,
  defaultFixture,
} from '../fixtures'

const createFixtureLoader = waffle.createFixtureLoader


describe('Assets contracts #fast', () => {
  // Tokens
  let rsr: ERC20Mock
  let compToken: ERC20Mock
  let aaveToken: ERC20Mock
  let rToken: TestIRToken
  let token: ERC20Mock
  let usdc: USDCMock
  let aToken: StaticATokenMock
  let cToken: CTokenMock

  // Assets
  let collateral0: FiatCollateral
  let collateral1: FiatCollateral
  let collateral2: ATokenFiatCollateral
  let collateral3: CTokenFiatCollateral

  // Assets
  let rsrAsset: Asset
  let compAsset: Asset
  let aaveAsset: Asset
  let rTokenAsset: RTokenAsset
  let basket: Collateral[]

  // Config
  let config: IConfig

  // Main
  let loadFixture: ReturnType<typeof createFixtureLoader>
  let wallet: Wallet
  let assetRegistry: IAssetRegistry
  let backingManager: TestIBackingManager

  // Factory
  let AssetFactory: ContractFactory
  let RTokenAssetFactory: ContractFactory

  const amt = fp('1e4')

  before('create fixture loader', async () => {
    ;[wallet] = (await ethers.getSigners()) as unknown as Wallet[]
    loadFixture = createFixtureLoader([wallet])
  })

  beforeEach(async () => {
    // Deploy fixture
    ;({
      rsr,
      rsrAsset,
      compToken,
      compAsset,
      aaveToken,
      aaveAsset,
      basket,
      assetRegistry,
      backingManager,
      config,
      rToken,
      rTokenAsset,
    } = await loadFixture(defaultFixture))

    // Get collateral tokens
    collateral0 = <FiatCollateral>basket[0]
    collateral1 = <FiatCollateral>basket[1]
    collateral2 = <ATokenFiatCollateral>basket[2]
    collateral3 = <CTokenFiatCollateral>basket[3]
    token = <ERC20Mock>await ethers.getContractAt('ERC20Mock', await collateral0.erc20())
    usdc = <USDCMock>await ethers.getContractAt('USDCMock', await collateral1.erc20())
    aToken = <StaticATokenMock>(
      await ethers.getContractAt('StaticATokenMock', await collateral2.erc20())
    )
    cToken = <CTokenMock>await ethers.getContractAt('CTokenMock', await collateral3.erc20())

    await rsr.connect(wallet).mint(wallet.address, amt)
    await compToken.connect(wallet).mint(wallet.address, amt)
    await aaveToken.connect(wallet).mint(wallet.address, amt)

    // Issue RToken to enable RToken.price
    for (let i = 0; i < basket.length; i++) {
      const tok = await ethers.getContractAt('ERC20Mock', await basket[i].erc20())
      await tok.connect(wallet).mint(wallet.address, amt)
      await tok.connect(wallet).approve(rToken.address, amt)
    }
    await rToken.connect(wallet)['issue(uint256)'](amt)

    AssetFactory = await ethers.getContractFactory('Asset')
    RTokenAssetFactory = await ethers.getContractFactory('RTokenAsset')
  })

  describe('Deployment', () => {
    it('Deployment should setup assets correctly', async () => {

        console.log(network.config.chainId);
        // let validOracle = '0x443C5116CdF663Eb387e72C688D276e702135C87';
        let deprecatedOracle = '0x2E5B04aDC0A3b7dB5Fd34AE817c7D0993315A8a6';
        let priceTimeout_ = await aaveAsset.priceTimeout(),
         chainlinkFeed_ = deprecatedOracle,
         oracleError_ = await aaveAsset.oracleError(),
         erc20_ = await aaveAsset.erc20(),
         maxTradeVolume_ = await aaveAsset.maxTradeVolume(),
         oracleTimeout_ = await aaveAsset.oracleTimeout();
        
        aaveAsset = await AssetFactory.deploy(priceTimeout_,
            chainlinkFeed_ ,
            oracleError_ ,
            erc20_ ,
            maxTradeVolume_ ,
            oracleTimeout_  ) as Asset;

        await aaveAsset.refresh();
      
    })
  })

})

```

Modification of `hardhat.config.ts` to set it to the Polygon network:
```diff
diff --git a/hardhat.config.ts b/hardhat.config.ts
index f1886d25..53565799 100644
--- a/hardhat.config.ts
+++ b/hardhat.config.ts
@@ -24,18 +24,19 @@ const TIMEOUT = useEnv('SLOW') ? 3_000_000 : 300_000
 const src_dir = `./contracts/${useEnv('PROTO')}`
 const settings = useEnv('NO_OPT') ? {} : { optimizer: { enabled: true, runs: 200 } }
 
+let recentBlockNumber = 38231040;
+let jan6Block = 37731612; // causes 'missing trie node' error
+
 const config: HardhatUserConfig = {
   defaultNetwork: 'hardhat',
   networks: {
     hardhat: {
       // network for tests/in-process stuff
-      forking: useEnv('FORK')
-        ? {
-            url: MAINNET_RPC_URL,
-            blockNumber: Number(useEnv('MAINNET_BLOCK', forkBlockNumber['default'].toString())),
-          }
-        : undefined,
-      gas: 0x1ffffffff,
+      forking: {
+          url: "https://rpc.ankr.com/polygon",
+          // blockNumber: recentBlockNumber
+          },
+          gas: 0x1ffffffff,
       blockGasLimit: 0x1fffffffffffff,
       allowUnlimitedContractSize: true,
     },
```

Output:
```
  1) Assets contracts #fast
       Deployment
         Deployment should setup assets correctly:
     Error: Transaction reverted without a reason string
    at Asset.refresh (contracts/plugins/assets/Asset.sol:102)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at async HardhatNode._mineBlockWithPendingTxs (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:1802:23)
    at async HardhatNode.mineBlock (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:491:16)
    at async EthModule._sendTransactionAndReturnHash (node_modules/hardhat/src/internal/hardhat-network/provider/modules/eth.ts:1522:18)
    at async HardhatNetworkProvider.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:118:18)
    at async EthersProviderWrapper.send (node_modules/@nomiclabs/hardhat-ethers/src/internal/ethers-provider-wrapper.ts:13:20)
```

Notes:
* Chainlink list deprecating oracles only till deprecation, afterwards they're removed from the website. For this reason I wasn't able to trace deprecated oracles on the mainnet
* I was trying to prove this worked before deprecation, however, I kept getting the 'missing trie node' error when forking the older block. This isn't essential for the PoC so I decided to give up on it for now (writing this PoC was hard enough on its own).

## Recommended Mitigation Steps
At `OracleLib.price()` catch the error and check if the error data is empty and the aggregator is set to the zero address, if it is a revert with a custom error. Otherwise revert with the original error data (this can be done with assembly).
Another approach might be to check in the `refresh()` function that the `tryPrice()` function didn't revert due to out of gas error by checking the gas before and after (in case of out of gas error only ~1/64 of the gas-before would be left). The advantage of this approach is that it would catch also other errors that might revert with empty data.