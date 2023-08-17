## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- M-02

# [The tier setting parameter are unsafely downcasted from type uint256 to type uint80 / uint48 / uint40 / uint16](https://github.com/code-423n4/2022-10-juicebox-findings/issues/31) 

# Lines of code

https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721Delegate.sol#L240
https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721DelegateStore.sol#L628
https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721DelegateStore.sol#L689


# Vulnerability details

## Impact

The tier setting parameter are unsafely downcasted from uint256 to uint80 / uint48 / uint16

the tier is setted by owner is crucial because the parameter affect how nft is minted.

the 

the callstack is 

JBTiered721Delegate.sol#initialize -> Store#recordAddTiers

```solidity
function recordAddTiers(JB721TierParams[] memory _tiersToAdd)
```

what does the struct JB721TierParams look like? all parameter in JB721TierParams is uint256 type

```solidity
struct JB721TierParams {
  uint256 contributionFloor;
  uint256 lockedUntil;
  uint256 initialQuantity;
  uint256 votingUnits;
  uint256 reservedRate;
  address reservedTokenBeneficiary;
  bytes32 encodedIPFSUri;
  bool allowManualMint;
  bool shouldUseBeneficiaryAsDefault;
}
```

however in side the function

```solidity
// Record adding the provided tiers.
if (_pricing.tiers.length > 0) _store.recordAddTiers(_pricing.tiers);
```

 all uint256 parameter are downcasted.

```solidity
// Add the tier with the iterative ID.
_storedTierOf[msg.sender][_tierId] = JBStored721Tier({
contributionFloor: uint80(_tierToAdd.contributionFloor),
lockedUntil: uint48(_tierToAdd.lockedUntil),
remainingQuantity: uint40(_tierToAdd.initialQuantity),
initialQuantity: uint40(_tierToAdd.initialQuantity),
votingUnits: uint16(_tierToAdd.votingUnits),
reservedRate: uint16(_tierToAdd.reservedRate),
allowManualMint: _tierToAdd.allowManualMint
});
```

uint256 contributionFloor is downcasted to uint80,

uint256 lockedUntil is downcasted to uint48

uint256 initialQuantity and initialQuantity are downcasted to uint40

uint256 votingUnits and uint256 reservedRate are downcasted to uint16

this means the original setting is greatly trancated. 

For example, the owner wants to set the initial supply to a number larger than uint40, but the supply is trancated to type(uint40).max

The owner wants to set the contribution floor price above uint80,but the contribution floor price is trancated to type(uint80).max, the user may underpay the price and get the NFT price at a discount.

## Proof of Concept

We can add POC

https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/forge-test/NFTReward_Unit.t.sol#L1689

```solidity
 function testJBTieredNFTRewardDelegate_mintFor_mintArrayOfTiers_downcast_POC() public {
    uint256 nbTiers = 1;

    vm.mockCall(
      mockJBProjects,
      abi.encodeWithSelector(IERC721.ownerOf.selector, projectId),
      abi.encode(owner)
    );

    JB721TierParams[] memory _tiers = new JB721TierParams[](nbTiers);
    uint16[] memory _tiersToMint = new uint16[](nbTiers);

    // Temp tiers, will get overwritten later (pass the constructor check)
    uint256 originalFloorPrice = 10000000000000000000000000 ether;
  
    for (uint256 i; i < nbTiers; i++) {
      _tiers[i] = JB721TierParams({
        contributionFloor: originalFloorPrice,
        lockedUntil: uint48(0),
        initialQuantity: 20,
        votingUnits: uint16(0),
        reservedRate: uint16(0),
        reservedTokenBeneficiary: reserveBeneficiary,
        encodedIPFSUri: tokenUris[i],
        allowManualMint: true, // Allow this type of mint
        shouldUseBeneficiaryAsDefault: false
      });

      _tiersToMint[i] = uint16(i)+1;
      _tiersToMint[_tiersToMint.length - 1 - i] = uint16(i)+1;
    }

    ForTest_JBTiered721DelegateStore _ForTest_store = new ForTest_JBTiered721DelegateStore();
    ForTest_JBTiered721Delegate _delegate = new ForTest_JBTiered721Delegate(
      projectId,
      IJBDirectory(mockJBDirectory),
      name,
      symbol,
      IJBFundingCycleStore(mockJBFundingCycleStore),
      baseUri,
      IJBTokenUriResolver(mockTokenUriResolver),
      contractUri,
      _tiers,
      IJBTiered721DelegateStore(address(_ForTest_store)),
      JBTiered721Flags({
        lockReservedTokenChanges: false,
        lockVotingUnitChanges: false,
        lockManualMintingChanges: true,
        pausable: true
      })
    );

    _delegate.transferOwnership(owner);

    uint256 floorPrice = _delegate.test_store().tier(address(_delegate), 1).contributionFloor;
    console.log("original floor price");
    console.log(originalFloorPrice);
    console.log("truncated floor price");
    console.log(floorPrice);

}
```

note, our initial contribution floor price setting is 

```solidity
uint256 originalFloorPrice = 10000000000000000000000000 ether;

for (uint256 i; i < nbTiers; i++) {
  _tiers[i] = JB721TierParams({
	contributionFloor: originalFloorPrice,
```

then we run our test

```solidity
forge test -vv --match testJBTieredNFTRewardDelegate_mintFor_mintArrayOfTiers_downcast_POC
```

the result is 

```solidity
[PASS] testJBTieredNFTRewardDelegate_mintFor_mintArrayOfTiers_downcast_POC() (gas: 7601212)
Logs:
  original floor price
  10000000000000000000000000000000000000000000
  truncated floor price
  863278115882885135204352

Test result: ok. 1 passed; 0 failed; finished in 10.43ms
```

clearly the floor price is unsafed downcasted and trancated.

## Tools Used

Foundry, Manual Review

## Recommended Mitigation Steps

We recommend the project either change the data type in the struct

```solidity
struct JB721TierParams {
  uint256 contributionFloor;
  uint256 lockedUntil;
  uint256 initialQuantity;
  uint256 votingUnits;
  uint256 reservedRate;
  address reservedTokenBeneficiary;
  bytes32 encodedIPFSUri;
  bool allowManualMint;
  bool shouldUseBeneficiaryAsDefault;
}
```

or safely downcast the number to make sure the number is not shortened unexpectedly.

https://docs.openzeppelin.com/contracts/3.x/api/utils#SafeCast