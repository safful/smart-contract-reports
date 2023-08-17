## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- M-07

# [NTokenMoonBirds Reserve Pool Cannot Receive Airdrops](https://github.com/code-423n4/2022-11-paraspace-findings/issues/286) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/NTokenMoonBirds.sol#L63


# Vulnerability details

## Proof-of-Concept

The NTokenMoonBirds Reserve Pool only allows Moonbird NFT to be sent to the pool contract as shown in Line 70 below.

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/NTokenMoonBirds.sol#L63

```solidity
File: NTokenMoonBirds.sol
63:     function onERC721Received(
64:         address operator,
65:         address from,
66:         uint256 id,
67:         bytes memory
68:     ) external virtual override returns (bytes4) {
69:         // only accept MoonBird tokens
70:         require(msg.sender == _underlyingAsset, Errors.OPERATION_NOT_SUPPORTED);
71: 
72:         // if the operator is the pool, this means that the pool is transferring the token to this contract
73:         // which can happen during a normal supplyERC721 pool tx
74:         if (operator == address(POOL)) {
75:             return this.onERC721Received.selector;
76:         }
77: 
78:         // supply the received token to the pool and set it as collateral
79:         DataTypes.ERC721SupplyParams[]
80:             memory tokenData = new DataTypes.ERC721SupplyParams[](1);
81: 
82:         tokenData[0] = DataTypes.ERC721SupplyParams({
83:             tokenId: id,
84:             useAsCollateral: true
85:         });
86: 
87:         POOL.supplyERC721FromNToken(_underlyingAsset, tokenData, from);
88: 
89:         return this.onERC721Received.selector;
90:     }
```

Note that the NTokenMoonBirds Reserve Pool is the holder/owner of all Moonbird NFTs within ParaSpace. If there is any airdrop for Moonbird NFT, the airdrop will be sent to the NTokenMoonBirds Reserve Pool.

However, due to the validation in Line 70, the NTokenMoonBirds Reserve Pool will not be able to receive any airdrop (e.g. Moonbirds Oddities NFT) other than the Moonbird NFT. In summary, if any NFT other than Moonbird NFT is sent to the pool, it will revert.

For instance, Moonbirds Oddities NFT has been airdropped to Moonbird NFT nested stakers in the past. With the nesting staking feature, more airdrops are expected in the future. In this case, any users who collateralized their nested Moonbird NFT within ParaSpacwill lose their opportunities to claim the airdrop.

## Impact

Loss of assets for the user. Users will not be able to receive any airdropped assets for their nested Moonbird NFT.

## Recommended Mitigation Steps

Update the contract to allow airdrops to be received by the NTokenMoonBirds Reserve Pool.

```diff
function onERC721Received(
	address operator,
	address from,
	uint256 id,
	bytes memory
) external virtual override returns (bytes4) {
-	// only accept MoonBird tokens
-	require(msg.sender == _underlyingAsset, Errors.OPERATION_NOT_SUPPORTED);

	// if the operator is the pool, this means that the pool is transferring the token to this contract
	// which can happen during a normal supplyERC721 pool tx
	if (operator == address(POOL)) {
		return this.onERC721Received.selector;
	}

+	if msg.sender == _underlyingAsset(
        // supply the received token to the pool and set it as collateral
        DataTypes.ERC721SupplyParams[]
            memory tokenData = new DataTypes.ERC721SupplyParams[](1);

        tokenData[0] = DataTypes.ERC721SupplyParams({
            tokenId: id,
            useAsCollateral: true
        });
+	)

	POOL.supplyERC721FromNToken(_underlyingAsset, tokenData, from);

	return this.onERC721Received.selector;
}
```