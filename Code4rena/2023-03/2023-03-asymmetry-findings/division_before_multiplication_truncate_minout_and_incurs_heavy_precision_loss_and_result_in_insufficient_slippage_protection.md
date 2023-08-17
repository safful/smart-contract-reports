## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- M-01

# [Division before multiplication truncate minOut and incurs heavy precision loss and result in insufficient slippage protection](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/1078) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L173
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L74


# Vulnerability details

## Impact

When Calcuting the minOut before doing trade,

Division before multiplication truncate minOut and incurs heavy precision loss, then very sub-optimal amount of the trade output can result in loss of fund from user because of the insufficient slippage protection

## Proof of Concept

In the current implementation,

slippage can be set by calling

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L206

```solidity
/**
	@notice - Sets the max slippage for a certain derivative index
	@param _derivativeIndex - index of the derivative you want to update the slippage
	@param _slippage - new slippage amount in wei
*/
function setMaxSlippage(
	uint _derivativeIndex,
	uint _slippage
) external onlyOwner {
	derivatives[_derivativeIndex].setMaxSlippage(_slippage);
	emit SetMaxSlippage(_derivativeIndex, _slippage);
}
```

which calls the corresponding derivative contract

### Case 1

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L58

```solidity
/**
	@notice - Owner only function to set max slippage for derivative
	@param _slippage - new slippage amount in wei
*/
function setMaxSlippage(uint256 _slippage) external onlyOwner {
	maxSlippage = _slippage;
}
```

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L173

calling

```solidity
uint256 minOut = ((((rethPerEth * msg.value) / 10 ** 18) *
	((10 ** 18 - maxSlippage))) / 10 ** 18);

IWETH(W_ETH_ADDRESS).deposit{value: msg.value}();
uint256 amountSwapped = swapExactInputSingleHop(
	W_ETH_ADDRESS,
	rethAddress(),
	500,
	msg.value,
	minOut
);
```

As we can see, the division before multiplication happens in the line of code

```solidity
uint256 minOut = ((((rethPerEth * msg.value) / 10 ** 18) *
	((10 ** 18 - maxSlippage))) / 10 ** 18);
```

for example, if maxSlippage is 10 ** 17

(10 ** 18 - 10 ** 17) / (10 ** 18) = 0

then minOut is 0, slippage control is disabled because of the division before multipcation.

### Case 2

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L51

```solidity
/**
	@notice - Owner only function to set max slippage for derivative
*/
function setMaxSlippage(uint256 _slippage) external onlyOwner {
	maxSlippage = _slippage;
}
```

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L74

the divison before multipilcation happens below

```solidity
uint256 minOut = (((ethPerDerivative(_amount) * _amount) / 10 ** 18) *
	(10 ** 18 - maxSlippage)) / 10 ** 18;

IFrxEthEthPool(FRX_ETH_CRV_POOL_ADDRESS).exchange(
	1,
	0,
	frxEthBalance,
	minOut
);
```

for example, if maxSlippage is 10 ** 17

(10 ** 18 - 10 ** 17) / (10 ** 18) = 0

then minOut is 0, slippage control is disabled because of the division before multipcation.

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the protocol avoid division before multiplcation when calcaluting the minOut to enable slippage protection and avoid front-running.