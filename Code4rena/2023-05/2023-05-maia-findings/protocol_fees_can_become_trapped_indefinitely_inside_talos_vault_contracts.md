## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-17

# [Protocol fees can become trapped indefinitely inside Talos vault contracts](https://github.com/code-423n4/2023-05-maia-findings/issues/583) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L394-L415


# Vulnerability details

## Impact

Talos strategy contracts all inherit logic from `TalosBaseStrategy`, including the function `collectProtocolFees`. This function is used by the owner to receive fees earned by the contract.

Talos vault contracts should be expected to work properly for any token that has a sufficiently liquid Uniswap pool. However certain ERC20 tokens [do not revert on failed transfer](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#SafeERC20), and instead return `false`. In `TalosBaseStrategy#collectProtocolFees`, tokens are transferred from the contract to the owner using `transfer`, and the return value is not checked. This means that the transfer could fail silently, in which case `protocolFees0` and `protocolFees1` would be updated without the tokens leaving the contract. This function is inherited by any Talos vault contract.

This accounting discrepancy causes the tokens to be irretrievably trapped in the contract.

## Proof of Concept

```solidity
    function collectProtocolFees(uint256 amount0, uint256 amount1) external nonReentrant onlyOwner {
        uint256 _protocolFees0 = protocolFees0;
        uint256 _protocolFees1 = protocolFees1;

        if (amount0 > _protocolFees0) {
            revert Token0AmountIsBiggerThanProtocolFees();
        }
        if (amount1 > _protocolFees1) {
            revert Token1AmountIsBiggerThanProtocolFees();
        }
        ERC20 _token0 = token0;
        ERC20 _token1 = token1;
        uint256 balance0 = _token0.balanceOf(address(this));
        uint256 balance1 = _token1.balanceOf(address(this));
        require(balance0 >= amount0 && balance1 >= amount1);
        if (amount0 > 0) _token0.transfer(msg.sender, amount0); // @audit should use `safeTransfer`
        if (amount1 > 0) _token1.transfer(msg.sender, amount1); // @audit should use `safeTransfer`

        protocolFees0 = _protocolFees0 - amount0;
        protocolFees1 = _protocolFees1 - amount1;
        emit RewardPaid(msg.sender, amount0, amount1);
    }
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L394-L415

## Tools Used
Manual review

## Recommended Mitigation Steps
Use [OpenZeppelin's SafeERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) library for ERC20 transfers.


## Assessed type

ERC20