## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- selected for report
- sponsor acknowledged
- upgraded by judge
- H-03

# [Risk of silent overflow in reserves update](https://github.com/code-423n4/2023-04-caviar-findings/issues/167) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L230-L231
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L323-L324


# Vulnerability details

The [`buy()`](https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L211) and [`sell()`](https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L301) functions update the `virtualBaseTokenReserves` and `virtualNftReserves` variables during each trade. However, these two variables are of type `uint128`, while the values that update them are of type `uint256`. This means that casting to a lower type is necessary, but this casting is performed without first checking that the values being cast can fit into the lower type. As a result, there is a risk of a silent overflow occurring during the casting process. 

```solidity
    function buy(uint256[] calldata tokenIds, uint256[] calldata tokenWeights, MerkleMultiProof calldata proof) 
        public
        payable
        returns (uint256 netInputAmount, uint256 feeAmount, uint256 protocolFeeAmount)
    {
        // ~~~ Checks ~~~ //

        // calculate the sum of weights of the NFTs to buy
        uint256 weightSum = sumWeightsAndValidateProof(tokenIds, tokenWeights, proof);

        // calculate the required net input amount and fee amount
        (netInputAmount, feeAmount, protocolFeeAmount) = buyQuote(weightSum);
        ...
        // update the virtual reserves
        virtualBaseTokenReserves += uint128(netInputAmount - feeAmount - protocolFeeAmount); 
        virtualNftReserves -= uint128(weightSum);
        ...
```

## Impact

If the reserves variables are updated with a silent overflow, it can lead to a breakdown of the xy=k equation. This, in turn, would result in a totally incorrect price calculation, causing potential financial losses for users or pool owners. 

## Proof of Concept

Consider the scenario with a base token that has high decimals number described in the next test (add it to the `test/PrivatePool/Buy.t.sol`):
```solidity
    function test_Overflow() public {
        // Setting up pool and base token HDT with high decimals number - 30
        // Initial balance of pool - 10 NFT and 100_000_000 HDT
        HighDecimalsToken baseToken = new HighDecimalsToken();
        privatePool = new PrivatePool(address(factory), address(royaltyRegistry), address(stolenNftOracle));
        privatePool.initialize(
            address(baseToken),
            nft,
            100_000_000 * 1e30,
            10 * 1e18,
            changeFee,
            feeRate,
            merkleRoot,
            true,
            false
        );

        // Minting NFT on pool address
        for (uint256 i = 100; i < 110; i++) {
            milady.mint(address(privatePool), i);
        }
        // Adding 8 NFT ids into the buying array
        for (uint256 i = 100; i < 108; i++) {
            tokenIds.push(i);
        }
        // Saving K constant (xy) value before the trade
        uint256 kBefore = uint256(privatePool.virtualBaseTokenReserves()) * uint256(privatePool.virtualNftReserves());

        // Minting enough HDT tokens and approving them for pool address
        (uint256 netInputAmount,, uint256 protocolFeeAmount) = privatePool.buyQuote(8 * 1e18);
        deal(address(baseToken), address(this), netInputAmount);
        baseToken.approve(address(privatePool), netInputAmount);

        privatePool.buy(tokenIds, tokenWeights, proofs);

        // Saving K constant (xy) value after the trade
        uint256 kAfter = uint256(privatePool.virtualBaseTokenReserves()) * uint256(privatePool.virtualNftReserves());

        // Checking that K constant succesfully was changed due to silent overflow
        assertEq(kBefore, kAfter, "K constant was changed");
    }
```
Also add this contract into the end of `Buy.t.sol` file for proper test work:
```solidity
    contract HighDecimalsToken is ERC20 {
        constructor() ERC20("High Decimals Token", "HDT", 30) {}
    }
```

## Recommended Mitigation Steps

Add checks that the casting value is not greater than the `uint128` type max value:
```solidity
File: PrivatePool.sol
229:         // update the virtual reserves
+            if (netInputAmount - feeAmount - protocolFeeAmount > type(uint128).max) revert Overflow();
230:         virtualBaseTokenReserves += uint128(netInputAmount - feeAmount - protocolFeeAmount); 
+            if (weightSum > type(uint128).max) revert Overflow();
231:         virtualNftReserves -= uint128(weightSum);

File: PrivatePool.sol
322:         // update the virtual reserves
+            if (netOutputAmount + protocolFeeAmount + feeAmount > type(uint128).max) revert Overflow();
323:         virtualBaseTokenReserves -= uint128(netOutputAmount + protocolFeeAmount + feeAmount);
+            if (weightSum > type(uint128).max) revert Overflow();
324:         virtualNftReserves += uint128(weightSum);
```