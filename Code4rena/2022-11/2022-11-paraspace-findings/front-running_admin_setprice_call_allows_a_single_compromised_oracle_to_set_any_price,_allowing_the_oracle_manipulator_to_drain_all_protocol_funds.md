## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- M-05

# [Front-running admin setPrice call allows a single compromised oracle to set any price, allowing the oracle manipulator to drain all protocol funds](https://github.com/code-423n4/2022-11-paraspace-findings/issues/267) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L195-L216
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L356-L364
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L402-L407
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L376-L386


# Vulnerability details

## Vulnerability details
The only way to update an NFT price is through [_finalizePrice](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L376-L386), which is called just by function ```setPrice```.

Current code forces the admin to call function ```setPrice``` in order to update the price, but to call this function the current implementation requires that [the asset is not paused](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L199).

What would happen if the admin ```setPrice``` was frontrunned by a feeder? The feeder who make the call will be allowed to set any price for the asset without any restriction. This obligates the protocol to consider next scenario:
* One feeder key or control has been compromised
* There is a new asset that the admin want to add to start feeding
* The new asset is being sold right now in a marketplace

Then, after admin calls ```addAssets``` and ```setPause``` functions, ```setPrice``` function can be frontrunned by the compromised feeder in order to set the price of the new asset to any price they want. This would allow the feeder's address controller to drain all protocol funds.

## Impact
Oracle decentralization can be bypassed, allowing ```setPrice``` function to be frontrunned by potencial oracle feeder (or person in control of oracle feeder), allowing the frontrunner to drain all protocol funds

This can happen because oracle decentralization control measures (requiring more than 3 oracle feeder to be deployed due to **MIN_ORACLES_NUM** value) can be bypassed.

## POC
1. Eve gets full control of one NFTFloorOracle feeder
1. Eve programs a bot to monitor when the admin of NFTFloorOracle contract calls ```addAssets``` and ```setPause``` function
1. ApeBoard lunch a new NFT collection called MONKEY
1. Some MONKEY collection tokens are in sale in Opean Sea
1. Paraspace decide to allow MONKEY collection to be used as collateral in the protocol and calls ```addAssets``` to add the new asset feed
1. Paraspace admin decide to call ```setPause``` to be able to call ```setPrice``` function
1. Eve's program detect the new NFT addition and the correspondin unpausing, then proceeds to:
    1. Buy one token of MONKEY collection
    1. Call ```setPrice``` through the feeder he control, and set the asset price to the maximum value possible
    1. Deposit the NFT in the protocol
    1. Borrow all the funds from the protocol

It is important to notice:
```solidity
function setPrice(address _asset, uint256 _twap)
    public
    onlyRole(UPDATER_ROLE)
    onlyWhenAssetExisted(_asset)
    whenNotPaused(_asset)
{
    bool dataValidity = false;
    // @audit This if block won't be executed by controlled feeder
    if (hasRole(DEFAULT_ADMIN_ROLE, msg.sender)) {
        _finalizePrice(_asset, _twap);
        return;
    }
    // @audit This can be bypassed giving the default twap value of the asset is zero
    dataValidity = _checkValidity(_asset, _twap);
    require(dataValidity, "NFTOracle: invalid price data");
    // @audit add price to raw feeder storage, never reverts
    _addRawValue(_asset, _twap);
    uint256 medianPrice;
    // set twap price only when median value is valid
    // @audit _combine will return (true, _twap)
    (dataValidity, medianPrice) = _combine(_asset, _twap);
    if (dataValidity) {
        // @audit Here the price is set, given dataValidity== true
        _finalizePrice(_asset, medianPrice);
    }
}
```

```solidity
function _checkValidity(address _asset, uint256 _twap)
    internal
    view
    returns (bool)
{
    // @audit the attacker will send a value higher than zero
    require(_twap > 0, "NFTOracle: price should be more than 0");
    PriceInformation memory assetPriceMapEntry = assetPriceMap[_asset];
    uint256 _priorTwap = assetPriceMapEntry.twap;
    uint256 _updatedAt = assetPriceMapEntry.updatedAt;
    uint256 priceDeviation;
    //@audit first price is always valid as long as _twap > 0, which is the case for the attacker who wants to maximize NFT value in order to drain protocol funds
    if (_priorTwap == 0 || _updatedAt == 0) {
        // @audit dataValidity in setPrice will be set to true
        return true;
    }
    // Rest of function code...
}
```

```solidity
function _combine(address _asset, uint256 _twap)
    internal
    view
    returns (bool, uint256)
{
    FeederRegistrar storage feederRegistrar = assetFeederMap[_asset];
    uint256 currentBlock = block.number;
    //@audit first time the asset price is set any input parameter will return (true, _twap)
    if (assetPriceMap[_asset].twap == 0) {
        return (true, _twap);
    }
    //Rest of function...
}
```

## Mitigation steps
1. Do not allow to unpause the asset unless its prices is different from zero.
1. Force the admin to set the first price for all new assets added. 

First step is easy, just a simple modification to setPause function:
```diff
function setPause(address _asset, bool _flag)
    external
    onlyRole(DEFAULT_ADMIN_ROLE)
{
    assetFeederMap[_asset].paused = _flag;
+   if(_flag){
+       require(assetFeederMap[_asset].twap != 0, ERROR_CANNOT_UNPAUSE_IF_PRICE_EQ_ZERO);
+   }
    
    emit AssetPaused(_asset, _flag);
}
```

This step forces to modify ```setPrice``` function and add a new function which might be called ```setEmergencyOrFirstPrice```
```solidity
// NftFloorOracle.sol
// Function to add
function setEmergencyOrFirstPrice(address _asset, uint256 _twap)
    public
    onlyRole(DEFAULT_ADMIN_ROLE)
    onlyWhenAssetExisted(_asset)
{
    require(_twap > 0, ERROR_TWAP_CANNOT_BE_ZERO());
    _finalizePrice(_asset, _twap);
}
```

Step 2 is included in the previous solution, giving we only allow to unpause the asset in case ```assetFeederMap[_asset].twap != 0```.

Doing this allows to simplify setPrice in order to save gas:

```diff
// New setPrice function
function setPrice(address _asset, uint256 _twap)
        public
        onlyRole(UPDATER_ROLE)
        onlyWhenAssetExisted(_asset)
        whenNotPaused(_asset)
    {        
-       bool dataValidity = false;
-       if (hasRole(DEFAULT_ADMIN_ROLE, msg.sender)) {
-           _finalizePrice(_asset, _twap);
-           return;
-       }
-       dataValidity = _checkValidity(_asset, _twap);
        bool dataValidity = _checkValidity(_asset, _twap);
        require(dataValidity, "NFTOracle: invalid price data");
        // add price to raw feeder storage
        _addRawValue(_asset, _twap);
        uint256 medianPrice;
        // set twap price only when median value is valid
        (dataValidity, medianPrice) = _combine(_asset, _twap);
        if (dataValidity) {
            _finalizePrice(_asset, medianPrice);
        }
    }
```

## Notes
* The scenario is focus on describing how front running setPrice function call lead to as many single points failures as feeders registered in the contract. Single point failures are described in  [this Chainlink document](https://20755222.fs1.hubspotusercontent-na1.net/hubfs/20755222/guides/the-ultimate-guide-to-blockchain-oracle-security.pdf) (point 3)
* If there is at least one feeder who works almost independently from ParaSpace protocol and is multisigned by people (not a governance contract) I consider this issue should be considered as high, given that all of them can have realistic motivations to drain protocol funds in their favour. 