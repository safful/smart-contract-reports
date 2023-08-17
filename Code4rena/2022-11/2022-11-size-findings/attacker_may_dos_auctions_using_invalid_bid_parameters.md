## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-06

# [Attacker may DOS auctions using invalid bid parameters](https://github.com/code-423n4/2022-11-size-findings/issues/237) 

# Lines of code

https://github.com/code-423n4/2022-11-size/blob/706a77e585d0852eae6ba0dca73dc73eb37f8fb6/src/SizeSealed.sol#L258-L263
https://github.com/code-423n4/2022-11-size/blob/706a77e585d0852eae6ba0dca73dc73eb37f8fb6/src/SizeSealed.sol#L157-L159
https://github.com/code-423n4/2022-11-size/blob/706a77e585d0852eae6ba0dca73dc73eb37f8fb6/src/SizeSealed.sol#L269-L280


# Vulnerability details

## Description

Buyers submit bids to SIZE using the bid() function. There's a max of 1000 bids allowed per auction in order to stop DOS attacks (Otherwise it could become too costly to execute the finalize loop). However, setting a max on number of bids opens up other DOS attacks.

In the finalize loop this code is used:

```
ECCMath.Point memory sharedPoint = ECCMath.ecMul(b.pubKey, sellerPriv);
// If the bidder public key isn't on the bn128 curve
if (sharedPoint.x == 1 && sharedPoint.y == 1) continue;
bytes32 decryptedMessage = ECCMath.decryptMessage(sharedPoint, b.encryptedMessage);
// If the bidder didn't faithfully submit commitment or pubkey
// Or the bid was cancelled
if (computeCommitment(decryptedMessage) != b.commitment) continue;
```

Note that pubKey is never validated. Therefore, attacker may pass pubKey = (0,0).
In ecMul, the code will return (1,1):
`if (scalar == 0 || (point.x == 0 && point.y == 0)) return Point(1, 1);`
As a result, we enter the continue block and the bid is ignored. Later, the bidder may receive their quote amount back using refund().

Another way to reach `continue` is by passing a 0 commitment.

One last way to force the auction to reject a bid is a low quotePerBase:

```
uint256 quotePerBase = FixedPointMathLib.mulDivDown(b.quoteAmount, type(uint128).max, baseAmount);
// Only fill if above reserve price
if (quotePerBase < data.reserveQuotePerBase) continue;
```

baseAmount is not validated as it is encrypted to seller's public key. Therefore, buyer may pass baseAmount = 0, making quotePerBase = 0.

So, attacker may submit 1000 invalid bids and completely DOS the auction.

## Impact

Attacker may DOS auctions using invalid bid parameters.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Implement a slashing mechanism. If the bid cannot be executed due to user's fault, some % their submitted quote tokens will go to the protocol and auction creator.