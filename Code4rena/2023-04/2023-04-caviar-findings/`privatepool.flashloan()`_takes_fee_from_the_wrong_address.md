## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-16

# [`PrivatePool.flashLoan()` takes fee from the wrong address](https://github.com/code-423n4/2023-04-caviar-findings/issues/56) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L623


# Vulnerability details

## Impact
Instead of taking the fee from the receiver of the flashloan callback, it pulls it from `msg.sender`.

As specified in [EIP-3156](https://eips.ethereum.org/EIPS/eip-3156#lender-specification):
> After the callback, the flashLoan function MUST take the amount + fee token from the receiver, or revert if this is not successful.

This will be an unexpected loss of funds for the caller if they have the pool pre-approved to spend funds (e.g. they previously bought NFTs) and are not the owner of the flashloan contract they use for the callback.

Additionally, for ETH pools, it expects the caller to pay the fee upfront. But, the fee is generally paid with the profits made using the flashloaned tokens.

## Proof of Concept
If `baseToken` is ETH, it expects the fee to already be sent with the call to `flashLoan()`. If it's an ERC20 token, it will pull it from `msg.sender` instead of `receiver`:
```sol
    function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 tokenId, bytes calldata data)
        external
        payable
        returns (bool)
    {
        // ...

        // calculate the fee
        uint256 fee = flashFee(token, tokenId);

        // if base token is ETH then check that caller sent enough for the fee
        if (baseToken == address(0) && msg.value < fee) revert InvalidEthAmount();
        
        // ...

        // transfer the fee from the borrower
        if (baseToken != address(0)) ERC20(baseToken).transferFrom(msg.sender, address(this), fee);

        return success;
    }
```

## Tools Used
none

## Recommended Mitigation Steps
Change to:
```sol
        uint initialBalance = address(this).balance;
        // ... 

        if (baseToken != address(0)) ERC20(baseToken).transferFrom(receiver, address(this), fee);
        else require(address(this).balance - initialBalance == fee);
```