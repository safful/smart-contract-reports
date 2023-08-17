## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor acknowledged
- H-01

# [Reentrancy in buy function for ERC777 tokens allows buying funds with considerable discount](https://github.com/code-423n4/2022-12-caviar-findings/issues/343) 

# Lines of code

https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L95
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L137
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L172
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L203


# Vulnerability details

## Description
Current implementation of functions ```add```, ```remove```, ```buy``` and ```sell``` first transfer fractional tokens, and then base tokens.

If this base token is ERC777 (extension of ERC20), we can call this function without updating the base token balance, but updating the fractional token balance.

## Impact
Allows to drain funds of a pairs which implements an ERC-777 token.


## POC
```diff
function buy(uint256 outputAmount, uint256 maxInputAmount) public payable returns (uint256 inputAmount) {
    // *** Checks *** //

    // check that correct eth input was sent - if the baseToken equals address(0) then native ETH is used
    require(baseToken == address(0) ? msg.value == maxInputAmount : msg.value == 0, "Invalid ether input");

    // calculate required input amount using xyk invariant
+   @audit Use current balances
    inputAmount = buyQuote(outputAmount);

    // check that the required amount of base tokens is less than the max amount
    require(inputAmount <= maxInputAmount, "Slippage: amount in");

    // *** Effects *** //
+   @audit Modifies just fractional balance
    // transfer fractional tokens to sender
    _transferFrom(address(this), msg.sender, outputAmount);

    // *** Interactions *** //

    if (baseToken == address(0)) {
        // refund surplus eth
        uint256 refundAmount = maxInputAmount - inputAmount;
        if (refundAmount > 0) msg.sender.safeTransferETH(refundAmount);
    } else {

        // transfer base tokens in
+       @audit If an ERC-777 token is used, we can re call buy function with the same balance of base token, but with different fractional balance
        ERC20(baseToken).safeTransferFrom(msg.sender, address(this), inputAmount);

    }
    emit Buy(inputAmount, outputAmount);
}
```

```solidity
function buyQuote(uint256 outputAmount) public view returns (uint256) {
    return (outputAmount * 1000 * baseTokenReserves()) / ((fractionalTokenReserves() - outputAmount) * 997);
}
```
The buy quote is used to calculate the amount of fractional token that the user will receive, and it should be less/equal to **maxInputAmount** sent by parameter in order to achieve a successful execution of function buy.

Current buy quote can be mathematically expressed as: $\frac{outputAmount \times 1000 \times baseTokenReserves}{fractionalTokenReserves - outPutAmount} \times 997$.

Then, about sales
```diff
function sell(uint256 inputAmount, uint256 minOutputAmount) public returns (uint256 outputAmount) {
    // *** Checks *** //

    // calculate output amount using xyk invariant
    outputAmount = sellQuote(inputAmount);

    // check that the outputted amount of fractional tokens is greater than the min amount
    require(outputAmount >= minOutputAmount, "Slippage: amount out");

    // *** Effects *** //

    // transfer fractional tokens from sender
+   //@audit fractional balance is updated
    _transferFrom(msg.sender, address(this), inputAmount);

    // *** Interactions *** //

    if (baseToken == address(0)) {
        // transfer ether out
        msg.sender.safeTransferETH(outputAmount);
    } else {
        // transfer base tokens out
+       @audit If an ERC-777 token is used, we can re call sell function with the same balance of base token, but with different fractional balance.
        ERC20(baseToken).safeTransfer(msg.sender, outputAmount);
    }

    emit Sell(inputAmount, outputAmount);
}
```

```function sellQuote(uint256 inputAmount) public view returns (uint256) {
    uint256 inputAmountWithFee = inputAmount * 997;
    return (inputAmountWithFee * baseTokenReserves()) / ((fractionalTokenReserves() * 1000) + inputAmountWithFee);
}
```

Current sellQuote function can be expressed mathematically as:

$inputAmount = \frac{inputAmount \times 997 \times baseTokenReserves}{fractionalTokenReserves \times 1000 + inputAmountWithFee}$

Then we can think next scenario to drain a pair which use an ERC-777 token as base token:
1. Let's suppose the pair has 1000 base tokens(BT777) and 1000 Fractional reserve tokens (FRT)
1. The attacker call buy function, all with next inputs:
    *  outputAmount = 50 
    * maxInputAmount = 80
1. The attacker implements a hook, that will be executed 6 times (using a counter inside a malicus contract) when a transfer is done, and call the buy function. After this 6 times the malicious contract is call again, but this times calls the sell function, doing a huge sell for the fractional reserve token obtained.

A simulation of this attack can be visualized in next table

|     Operation  | outputAmount (FRT) | maxInputAmount (BT777) | BT777 reserve | FRT reserve | inputAmount (BT777 to pay) | inputAmount < maxInputAmount |
|:-------------------|-------------------|-----------------------|---------------|-------------|-----------------------|-------------------:|
| Attaker buy 1  | 50 | 80 | 1000 | 1000 | 52 | TRUE |
| Callback buy 2 | 50 | 80 | 1000 | 950 | 55 | TRUE |
| Callback buy 3 | 50 | 80 | 1000 | 900 | 59 | TRUE |
| Callback buy 4 | 50 | 80 | 1000 | 850 | 62 | TRUE |
| Callback buy 5 | 50 | 80 | 1000 | 800 | 66 | TRUE |
| Callback buy 6 | 50 | 80 | 1000 | 750 | 71 | TRUE |
| Callback buy 7 | 50 | 80 | 1000 | 700 | 77 | TRUE |

The result of this operation is that the attaker/malicious contract has 350 FRT, while BT777 reserve still has 1000 and FRT reserve has 650 tokens. The success execution needs that the attacker pays 442 BT777 eventually.

To do this, the last operation of the malicious contract is calling sell function

| Operation | inputAmount(BT777) | minOutputAmount | BT777 reserve | FRT reserve | outputAmount (BT777 to receive) | outputAmount > minOutputAmount |
|:-------------------|-------------------|-----------------------|---------------|-------------|-----------------------|-------------------:|
| calback Sell | 350 | 442 | 1000 | 650 | 536 | TRUE |

The result is that the attacker now controls 536 BT777, the attacker use this balance to pay the debt of 442 BT77, with a profit of 94 BT77 tokens.

## Mitigation steps
Add openzeppelin nonReentrant modifier to mentioned functions, or state clear in the documentation that this protocol should not be used with ERC777 tokens.