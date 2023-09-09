## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-33

# [Users lose their entire investment when making a deposit and resulting shares are zero](https://github.com/code-423n4/2023-01-popcorn-findings/issues/155) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L392
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L110-L122
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L147-L165


# Vulnerability details

## Impact
Users could receive `0` shares and thus lose their entire investment when making a deposit.

## Proof of Concept

Alice calls `deposit` with `999` assets, with herself as the receiver

```solidity
function deposit(uint256 assets, address receiver)
    public
    virtual
    override
    returns (uint256)
{
    if (assets > maxDeposit(receiver)) revert MaxError(assets);

    uint256 shares = _previewDeposit(assets);
    _deposit(_msgSender(), receiver, assets, shares);

    return shares;
}
```
Shares are calculated through `_previewDeposit`, which uses `_convertToShares` rounding down

```solidity
function _convertToShares(uint256 assets, Math.Rounding rounding)
    internal
    view
    virtual
    override
    returns (uint256 shares)
{
    uint256 _totalSupply = totalSupply();
    uint256 _totalAssets = totalAssets();
    return
        (assets == 0 || _totalSupply == 0 || _totalAssets == 0)
            ? assets
            : assets.mulDiv(_totalSupply, _totalAssets, rounding);
}
```

With specific conditions, the share calculation will round to zero. 
Let's suppose that `_totalSupply = 100_000` and  `_totalAssets = 100_000_000`, then:

```solidity
assets * _totalSupply / _totalAssets => 999 * 100_000 / 100_000_000
``` 
which rounds to zero, so total shares are `0`.

Finally, the deposit is completed, and the adapter mints `0 shares`.

```solidity
function _deposit(
    address caller,
    address receiver,
    uint256 assets,
    uint256 shares
) internal nonReentrant virtual override {
    IERC20(asset()).safeTransferFrom(caller, address(this), assets);
    
    uint256 underlyingBalance_ = _underlyingBalance();
    _protocolDeposit(assets, shares);
    // Update the underlying balance to prevent inflation attacks
    underlyingBalance += _underlyingBalance() - underlyingBalance_;

    _mint(receiver, shares);

    harvest();

    emit Deposit(caller, receiver, assets, shares);
}
```

Alice has lost `999` assets and she has received nothing in return.

## Tools Used

Manual review

## Recommended Mitigation Steps

Revert the transaction when a deposit would result in `0` shares minted.