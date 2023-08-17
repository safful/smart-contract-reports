## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-01

# [`ParticleExchange.auctionBuyNft` and `ParticleExchange.withdrawEthWithInterest` function calls can be DOS'ed](https://github.com/code-423n4/2023-05-particle-findings/issues/31) 

# Lines of code

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L688-L748
https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L192-L223


# Vulnerability details

## Impact
When `lien.borrower` is a contract, its `receive` function can be coded to conditionally revert based on a state boolean variable controlled by `lien.borrower`'s owner. As long as `payback > 0` is true, `lien.borrower`'s `receive` function would be called when calling the following `ParticleExchange.auctionBuyNft` function. In this situation, if the owner of `lien.borrower` intends to DOS the `ParticleExchange.auctionBuyNft` function call, especially when `lien.credit` is low or 0, she or he would make `lien.borrower`'s `receive` function revert.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L688-L748
```solidity
    function auctionBuyNft(
        Lien calldata lien,
        uint256 lienId,
        uint256 tokenId,
        uint256 amount
    ) external override validateLien(lien, lienId) auctionLive(lien) {
        ...

        // pay PnL to borrower
        uint256 payback = lien.credit + lien.price - payableInterest - amount;
        if (payback > 0) {
            payable(lien.borrower).transfer(payback);
        }

        ...
    }
```

Moreover, after the auction of the lien is concluded, calling the following `ParticleExchange.withdrawEthWithInterest` function can call `lien.borrower`'s `receive` function as long as `lien.credit > payableInterest` is true. In this case, the owner of `lien.borrower` can also make `lien.borrower`'s `receive` function revert to DOS the `ParticleExchange.withdrawEthWithInterest` function call.

https://github.com/code-423n4/2023-05-particle/blob/bbd1c01407a017046c86fdb483bbabfb1fb085d8/contracts/protocol/ParticleExchange.sol#L192-L223
```solidity
    function withdrawEthWithInterest(Lien calldata lien, uint256 lienId) external override validateLien(lien, lienId) {
        ...

        uint256 payableInterest = _calculateCurrentPayableInterest(lien);

        // verify that the liquidation condition has met (borrower insolvent or auction concluded)
        if (payableInterest < lien.credit && !_auctionConcluded(lien.auctionStartTime)) {
            revert Errors.LiquidationHasNotReached();
        }

        // delete lien (delete first to prevent reentrancy)
        delete liens[lienId];

        // transfer ETH with interest back to lender
        payable(lien.lender).transfer(lien.price + payableInterest);

        // transfer PnL to borrower
        if (lien.credit > payableInterest) {
            payable(lien.borrower).transfer(lien.credit - payableInterest);
        }

        ...
    }
```

The similar situations can happen if `lien.borrower` does not implement the `receive` or `fallback` function intentionally in which `lien.borrower`'s owner is willing to pay some position margin, which can be a low amount depending on the corresponding lien, to DOS the `ParticleExchange.auctionBuyNft` and `ParticleExchange.withdrawEthWithInterest` function calls.

## Proof of Concept
The following steps can occur for the described scenario for the `ParticleExchange.auctionBuyNft` function. The situation for the `ParticleExchange.withdrawEthWithInterest` function is similar.
1. Alice is the owner of `lien.borrower` for a lien.
2. The lender of the lien starts the auction for the lien.
3. Alice does not want the auction to succeed so she makes `lien.borrower`'s `receive` function revert through changing the controlled state boolean variable for launching the DOS attack to true.
4. For couple times during the auction period, some other users are willing to win the auction by supplying an NFT from the same collection but their `ParticleExchange.auctionBuyNft` function calls all revert.
5. Since no one's `ParticleExchange.auctionBuyNft` transaction is executed at the last second of the auction period, the auction is DOS'ed.

## Tools Used
VSCode

## Recommended Mitigation Steps
The `ParticleExchange.auctionBuyNft` and `ParticleExchange.withdrawEthWithInterest` functions can be updated to record the `payback` and `lien.credit - payableInterest` amounts that should belong to `lien.borrower` instead of directly sending these amounts to `lien.borrower`. Then, a function can be added to let `lien.borrower` to call and receive these recorded amounts.


## Assessed type

DoS