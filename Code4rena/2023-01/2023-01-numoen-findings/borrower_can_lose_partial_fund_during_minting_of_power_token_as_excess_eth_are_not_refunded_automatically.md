## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-05

# [Borrower can lose partial fund during minting of Power Token as excess ETH are not refunded automatically](https://github.com/code-423n4/2023-01-numoen-findings/issues/174) 

# Lines of code

https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/LendgineRouter.sol#L142
https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/LendgineRouter.sol#L87-L124
https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/LendgineRouter.sol#L119-L123
https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/Payment.sol#L44-L46


# Vulnerability details

## Impact
When the collateral/speculative token (Token1) is WETH, a borrower could mint Power Tokens and deposit the collateral tokens by sending ETH while calling the payable mint() function in LendgineRouter.sol. 

The exact collateral amount required to be deposited by the borrower is only calculated during minting (due to external swap), which could be lesser than what the borrower has sent for the mint. This means that there will be excess ETH left in LengineRouter contract and they are not automatically refunded to the borrower. 

Anyone that see this opportunity can call refundETH() to retrieve the excess ETH. 

The borrower could retrieve the remaining ETH with a separate call to refundETH(). However, as the calls are not atomic, it is possible for a MEV bot to frontrun the borrower and steal the ETH too. 

Furthermore, there are no documentation and test cases that advise or handle this issue.

## Proof of Concept

First, call payable mint() in LendgineRouter contract with the required ETH amount for collateral. 

	function mint(MintParams calldata params) external payable checkDeadline(params.deadline) returns (uint256 shares) {

https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/LendgineRouter.sol#L142

LendgineRouter.mintCallback() will be triggered, which will perform the external swap of the borrowed token0 to token1 on uniswap. The collateralSwap value (token1) is only calculated and known after the successful swap. Both swapped token1 and borrowed token1 are then sent to Lendgine contract (msg.sender).

    // swap all token0 to token1
    uint256 collateralSwap = swap(
      decoded.swapType,
      SwapParams({
        tokenIn: decoded.token0,
        tokenOut: decoded.token1,
        amount: SafeCast.toInt256(amount0),
        recipient: msg.sender
      }),
      decoded.swapExtraData
    );

    // send token1 back
    SafeTransferLib.safeTransfer(decoded.token1, msg.sender, amount1);

https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/LendgineRouter.sol#L87-L124

After that, mintCallback() will continue to calculate the remaining token1 required to be paid by the borrower (collateralIn value).  

Depending on the external swap, the collateralSwap (token1) value could be higher than expected, resulting in a lower collateralIn value. A small collateralIn value means that less ETH is required to be paid by the borrower (via the pay function), resulting in excess ETH left in the LengineRouter contract. However, the excess ETH is not automatically refunded by the mint() call.

Note: For WETH, the pay() uses the ETH balance deposited and wrap it before transferring to Lendgine contract.

    // pull the rest of tokens from the user
    uint256 collateralIn = collateralTotal - amount1 - collateralSwap;
    if (collateralIn > decoded.collateralMax) revert AmountError();

    pay(decoded.token1, decoded.payer, msg.sender, collateralIn);

https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/LendgineRouter.sol#L119-L123

A MEV bot or anyone that see this opportunity can call refundETH() to retrieve the excess ETH.

	function refundETH() external payable {
		if (address(this).balance > 0) SafeTransferLib.safeTransferETH(msg.sender, address(this).balance);
	}

https://github.com/code-423n4/2023-01-numoen/blob/main/src/periphery/Payment.sol#L44-L46

## Recommended Mitigation Steps
Automatically refund any excess ETH to the borrower.