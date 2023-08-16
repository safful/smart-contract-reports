## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- high quality report
- primary issue
- selected for report
- edited-by-warden
- M-03

# [Funding cycles that use `JBXBuybackDelegate` as a redeem data source (or any derived class) will slash all redeemable tokens](https://github.com/code-423n4/2023-05-juicebox-findings/issues/79) 

# Lines of code

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L235-L239


# Vulnerability details

## Impact

For a funding cycle that is set to use the data source for redeem and the data source is `JBXBuybackDelegate`, any and all redeemed tokens is 0 because of the returned 0 values from the empty `redeemParams` implementation. This means the beneficiary would wrongly receive 0 tokens instead of an intended amount.

While `JBXBuybackDelegate` was not designed to be used for redeem also, per protocol logic, this function should of been a pass-through (if no redeem such functionality was desired) because a redeem logic is by default used if not overwrite by `redeemParams`, as per [the documentation](https://docs.juicebox.money/dev/build/treasury-extensions/data-source/) (further detailed below).
Impact is further increased as the function is not marked as `virtual`, as such, no inheriting class can modify the faulty implementation if any such extension is desired.

## Vulnerability details

Project logic dictates that a contract can become a treasury data source by adhering to `IJBFundingCycleDataSource`, such as `JBXBuybackDelegate` is and
that there are two functions that must be implemented, `payParams(...)` and `redeemParams(...)`. 
Either one can be **left empty** if the intent is to only extend the treasury's pay functionality or redeem functionality.

But by left empty, it is specifically required to pass the input variables, not a 0 default value

https://docs.juicebox.money/dev/build/treasury-extensions/data-source/

> Return the JBRedeemParamsData.reclaimAmount value if no altered functionality is desired.
> Return the JBRedeemParamsData.memo value if no altered functionality is desired.
> Return the zero address if no additional functionality is desired.

What should of been done:

```Solidity
  // This is unused but needs to be included to fulfill IJBFundingCycleDataSource.
  function redeemParams(JBRedeemParamsData calldata _data)
    external
    pure
    override
    returns (
      uint256 reclaimAmount,
      string memory memo,
      IJBRedemptionDelegate delegate
    )
  {
    // Return the default values.
    return (_data.reclaimAmount.value, _data.memo, IJBRedemptionDelegate(address(0)));
  }
```

What is implemented:

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L235-L239
```Solidity
    function redeemParams(JBRedeemParamsData calldata _data)
        external
        override
        returns (uint256 reclaimAmount, string memory memo, JBRedemptionDelegateAllocation[] memory delegateAllocations)
    {}
```

While an overwritten `memo` with a empty default string is not such a large issue, and incidentally `delegateAllocations` is actually returned correctly as a zero address, 
the issue lies with the `reclaimAmount` amount which not is returned as 0.

Per documentation:

> `reclaimAmount` is the amount of tokens in the treasury that the terminal should send out to the redemption beneficiary as a result of burning the amount of project tokens tokens

and in case of redemptions, by default, the value is

> a reclaim amount determined by the standard protocol bonding curve based on the redemption rate the project has configured into its current funding cycle which is provided to the data source function in JBRedeemParamsData.reclaimAmount

but in this case, it will be overwritten with 0 thus the redemption beneficiary will be deprived of funds.

`redeemParams` is also lacking the `virtual` keyword, as such, no inheriting class can modify the faulty implementation if any such extension is desired.

## Tools Used

Manual analysis

## Recommended Mitigation Steps

Change the implementation of the function to respect protocol logic and add the `virtual` keyword, example:

```Solidity
  // This is unused but needs to be included to fulfill IJBFundingCycleDataSource.
  function redeemParams(JBRedeemParamsData calldata _data)
    external
    pure
    override
    virtual
    returns (
      uint256 reclaimAmount,
      string memory memo,
      IJBRedemptionDelegate delegate
    )
  {
    // Return the default values.
    return (_data.reclaimAmount.value, _data.memo, IJBRedemptionDelegate(address(0)));
  }
```

## Note for judging

Any user/developer that wishes to extend the tool cannot or, at worst, uses a faulty implementation by mistake. Historically, medium severity was awarded to various issues which rely on some user error (in this case using a faulty implementation because docs indicate that all non-redeemable data sources are either pass-through, reverts or custom implementation), are easy to check/fix and present material risk. I respect this line of thought and for the sake of consistency I believe this submission should be judged similarly. 


## Assessed type

Other