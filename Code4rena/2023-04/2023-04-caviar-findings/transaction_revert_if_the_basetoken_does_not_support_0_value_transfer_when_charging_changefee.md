## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-13

# [Transaction revert if the baseToken does not support 0 value transfer when charging changeFee](https://github.com/code-423n4/2023-04-caviar-findings/issues/278) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L423
https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L651


# Vulnerability details

## Impact

Transaction revert if the baseToken does not support 0 value transfer when charging changeFee

## Proof of Concept

When call change via the PrivatePool.sol, the caller needs to the pay the change fee,

```solidity
	// calculate the fee amount
	(feeAmount, protocolFeeAmount) = changeFeeQuote(inputWeightSum);
}

// ~~~ Interactions ~~~ //

if (baseToken != address(0)) {
	// transfer the fee amount of base tokens from the caller
	ERC20(baseToken).safeTransferFrom(
		msg.sender,
		address(this),
		feeAmount
	);
```

calling changeFeeQuote(inputWeightSum)

```solidity
function changeFeeQuote(
	uint256 inputAmount
) public view returns (uint256 feeAmount, uint256 protocolFeeAmount) {
	// multiply the changeFee to get the fee per NFT (4 decimals of accuracy)
	uint256 exponent = baseToken == address(0)
		? 18 - 4
		: ERC20(baseToken).decimals() - 4;
	uint256 feePerNft = changeFee * 10 ** exponent;

	feeAmount = (inputAmount * feePerNft) / 1e18;
	protocolFeeAmount =
		(feeAmount * Factory(factory).protocolFeeRate()) /
		10_000;
}
```

if the feeAmount is 0,

the code below woud revert if the ERC20 token does not support 0 value transfer

```solidity
ERC20(baseToken).safeTransferFrom(
	msg.sender,
	address(this),
	feeAmount
);
```

According to https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers

> Some tokens (e.g. LEND) revert when transferring a zero value amount.

Same issue happens when charging the flashloan fee

```solidity
    function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 tokenId,
        bytes calldata data
    ) external payable returns (bool) {
        // check that the NFT is available for a flash loan
        if (!availableForFlashLoan(token, tokenId))
            revert NotAvailableForFlashLoan();

        // calculate the fee
        uint256 fee = flashFee(token, tokenId);

        // if base token is ETH then check that caller sent enough for the fee
        if (baseToken == address(0) && msg.value < fee)
            revert InvalidEthAmount();

        // transfer the NFT to the borrower
        ERC721(token).safeTransferFrom(
            address(this),
            address(receiver),
            tokenId
        );

        // call the borrower
        bool success = receiver.onFlashLoan(
            msg.sender,
            token,
            tokenId,
            fee,
            data
        ) == keccak256("ERC3156FlashBorrower.onFlashLoan");

        // check that flashloan was successful
        if (!success) revert FlashLoanFailed();

        // transfer the NFT from the borrower
        ERC721(token).safeTransferFrom(
            address(receiver),
            address(this),
            tokenId
        );

        // transfer the fee from the borrower
        if (baseToken != address(0))
            ERC20(baseToken).transferFrom(msg.sender, address(this), fee);

        return success;
    }
```

note the code:

```solidity
if (baseToken != address(0))
            ERC20(baseToken).transferFrom(msg.sender, address(this), fee);
```

if the fee is 0 and baseToken revert in 0 value transfer, the user cannot use flashloan

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the protocol check if the feeAmount is 0 before performing transfer

```solidity
if(feeAmount > 0) {
	ERC20(baseToken).safeTransferFrom(
		msg.sender,
		address(this),
		feeAmount
	);
}
```