## Tags

- bug
- downgraded by judge
- grade-a
- QA (Quality Assurance)
- selected for report
- Q-54

# [Lack of reasonable boundary for parameter setting in fee setting and liquidation auction length and refinance setting and epoch length](https://github.com/code-423n4/2023-01-astaria-findings/issues/77) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L273
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L288
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L303
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L311


# Vulnerability details

## Impact

Lack of reasonable boundary for parameter setting in fee setting and liquidation length

## Proof of Concept

According to the documentation,

https://docs.astaria.xyz/docs/protocol-mechanics/liquidations

> Auction lengths are set to 72 hours, and will follow a Dutch Auction process.

In the constructor of the AstariaRouter.sol

LInk here

```solidity
    s.auctionWindow = uint32(2 days);
    s.auctionWindowBuffer = uint32(1 days);
```

but these parameter can be adjusted with no boundary restriction in the function file

```solidity
  function _file(File calldata incoming) internal {
    RouterStorage storage s = _loadRouterSlot();
    FileType what = incoming.what;
    bytes memory data = incoming.data;
    if (what == FileType.AuctionWindow) {
      (uint256 window, uint256 windowBuffer) = abi.decode(
        data,
        (uint256, uint256)
      );
      s.auctionWindow = window.safeCastTo32();
      s.auctionWindowBuffer = windowBuffer.safeCastTo32();
    }
```

admin can adjust the auction length to very short or very long, which violates the documentation.

If the admin adjust the auction length to very short, for example, 2 hours, the auction time is too short to let people purchase off the outstanding debt, and the lender has to bear the loss.

if the auction length is too long, for example, 2000 days, this basically equal to lock the NFT auction fund and the lender will not get paid either.

According to documentation

https://docs.astaria.xyz/docs/protocol-mechanics/refinance

> An improvement in terms is considered if either of these conditions is met:

> The loan interest rate decrease by more than 0.05%.
> The loan duration increases by more than 14 days.

However, In the constructor of the AstariaRouter.sol

such parameter is not enforced.

the s.minDurationIncrease is set to 5 days, not 14 days.

Link here

```solidity
s.minDurationIncrease = uint32(5 days);
```

which impact the refinanc logic

```solidity
  function isValidRefinance(
    ILienToken.Lien calldata newLien,
    uint8 position,
    ILienToken.Stack[] calldata stack
  ) public view returns (bool) {
    RouterStorage storage s = _loadRouterSlot();
    uint256 maxNewRate = uint256(stack[position].lien.details.rate) -
      s.minInterestBPS;

    if (newLien.collateralId != stack[0].lien.collateralId) {
      revert InvalidRefinanceCollateral(newLien.collateralId);
    }
    return
      (newLien.details.rate <= maxNewRate &&
        newLien.details.duration + block.timestamp >=
        stack[position].point.end) ||
      (block.timestamp + newLien.details.duration - stack[position].point.end >=
        s.minDurationIncrease &&
        newLien.details.rate <= stack[position].lien.details.rate);
  }
```

the relevant parameter s.minInterestBPS and s.minDurationIncrease can be adjusted in the function file with no boundary setting.

```solidity
} else if (what == FileType.MinInterestBPS) {
  uint256 value = abi.decode(data, (uint256));
  s.minInterestBPS = value.safeCastTo32();
} else if (what == FileType.MinDurationIncrease) {
  uint256 value = abi.decode(data, (uint256));
  s.minDurationIncrease = value.safeCastTo32();
} 
```

The impact is that if the The loan duration increases duration is too long and the interest decreases too much, this may favor the lender too much and not fair to borrower. The payment to lender can be infinitely delayed.

If the loan duration increase duration is too short and the interest decrease is too small, the refinance become pointless.

If the admin change the protocol fee, buyout fee or the epoch length or the max interest rate with no reasonable boundary by calling Astaria#file, the impact is severe

```solidity
    } else if (what == FileType.ProtocolFee) {
      (uint256 numerator, uint256 denominator) = abi.decode(
        data,
        (uint256, uint256)
      );
      if (denominator < numerator) revert InvalidFileData();
      s.protocolFeeNumerator = numerator.safeCastTo32();
      s.protocolFeeDenominator = denominator.safeCastTo32();
    } else if (what == FileType.BuyoutFee) {
      (uint256 numerator, uint256 denominator) = abi.decode(
        data,
        (uint256, uint256)
      );
      if (denominator < numerator) revert InvalidFileData();
      s.buyoutFeeNumerator = numerator.safeCastTo32();
      s.buyoutFeeDenominator = denominator.safeCastTo32();
    }
```

and

```solidity
else if (what == FileType.MinDurationIncrease) {
  uint256 value = abi.decode(data, (uint256));
  s.minDurationIncrease = value.safeCastTo32();
} else if (what == FileType.MinEpochLength) {
  s.minEpochLength = abi.decode(data, (uint256)).safeCastTo32();
} else if (what == FileType.MaxEpochLength) {
  s.maxEpochLength = abi.decode(data, (uint256)).safeCastTo32();
} else if (what == FileType.MaxInterestRate) {
  s.maxInterestRate = abi.decode(data, (uint256)).safeCastTo88();
} 
```

the admin can charge high liqudation fee and there may not be enough fund left to pay out the outstanding debt.

the admin can charge high buyout fee, which impact the lien token buyout.

If the max interest rate is high, the interest can become unreasonable for a vault and not fair for lender to pay out the accuring debt.

if the epoch lengh is too long, the gap between withdraw is too long and the user cannot withdraw their fund on time.

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the protocol add reasonable boundary to fee setting and liqudation length setting.