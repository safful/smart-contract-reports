## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-03

# [Users can fail to unstake and lose their deserved ETH because malfunctioning or untrusted derivative cannot be removed](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/703) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L165-L175
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L108-L129
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/WstEth.sol#L56-L67
https://etherscan.io/address/0xDC24316b9AE028F1497c275EB9192a3Ea0f67022#code#L441


# Vulnerability details

## Impact
Calling the following `SafEth.adjustWeight` function can update the weight for an existing derivative to 0. However, there is no way to remove an existing derivative. If the external contracts that an existing derivative depends on malfunction or get hacked, this protocol's functionalities that need to loop through the existing derivatives can behave unexpectedly. Users can fail to unstake and lose their deserved ETH as one of the severest consequences.

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L165-L175
```solidity
    function adjustWeight(
        uint256 _derivativeIndex,
        uint256 _weight
    ) external onlyOwner {
        weights[_derivativeIndex] = _weight;
        uint256 localTotalWeight = 0;
        for (uint256 i = 0; i < derivativeCount; i++)
            localTotalWeight += weights[i];
        totalWeight = localTotalWeight;
        emit WeightChange(_derivativeIndex, _weight);
    }
```

For example, calling the following `SafEth.unstake` function would loop through all of the existing derivatives and call the corresponding derivative's `withdraw` function. When the `WstEth` contract is one of these derivatives, the `WstEth.withdraw` function would be called, which further calls `IStEthEthPool(LIDO_CRV_POOL).exchange(1, 0, stEthBal, minOut)`. If `self.is_killed` in the stETH-ETH pool contract corresponding to `LIDO_CRV_POOL` becomes true, especially after such pool contract becomes compromised or hacked, calling such `exchange` function would always revert. In this case, calling the `SafEth.unstake` function reverts even though all other derivatives that are not the `WstEth` contract are still working fine. Because the `SafEth.unstake` function is DOS'ed, users cannot unstake and withdraw ETH that they are entitled to.

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L108-L129
```solidity
    function unstake(uint256 _safEthAmount) external {
        require(pauseUnstaking == false, "unstaking is paused");
        uint256 safEthTotalSupply = totalSupply();
        uint256 ethAmountBefore = address(this).balance;

        for (uint256 i = 0; i < derivativeCount; i++) {
            // withdraw a percentage of each asset based on the amount of safETH
            uint256 derivativeAmount = (derivatives[i].balance() *
                _safEthAmount) / safEthTotalSupply;
            if (derivativeAmount == 0) continue; // if derivative empty ignore
            derivatives[i].withdraw(derivativeAmount);
        }
        ...
    }
```

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/WstEth.sol#L56-L67
```solidity
    function withdraw(uint256 _amount) external onlyOwner {
        IWStETH(WST_ETH).unwrap(_amount);
        uint256 stEthBal = IERC20(STETH_TOKEN).balanceOf(address(this));
        IERC20(STETH_TOKEN).approve(LIDO_CRV_POOL, stEthBal);
        uint256 minOut = (stEthBal * (10 ** 18 - maxSlippage)) / 10 ** 18;
        IStEthEthPool(LIDO_CRV_POOL).exchange(1, 0, stEthBal, minOut);
        ...
    }
```

https://etherscan.io/address/0xDC24316b9AE028F1497c275EB9192a3Ea0f67022#code#L441
```solidity
def exchange(i: int128, j: int128, dx: uint256, min_dy: uint256) -> uint256:
    ...
    assert not self.is_killed  # dev: is killed
```

## Proof of Concept
The following steps can occur for the described scenario.
1. The `WstEth` contract is one of the existing derivatives. For the `WstEth` contract, the stETH-ETH pool contract corresponding to `LIDO_CRV_POOL` has been hacked in which its `self.is_killed` has been set to true.
2. Alice calls the `SafEth.unstake` function but such function call reverts because calling the stETH-ETH pool contract's `exchange` function reverts for the `WstEth` derivative.
3. Although all other derivatives that are not the `WstEth` contract are still working fine, Alice is unable to unstake. As a result, she cannot withdraw and loses her deserved ETH.

## Tools Used
VSCode

## Recommended Mitigation Steps
The `SafEth` contract can be updated to add a function, which would be only callable by the trusted admin, for removing an existing derivative that already malfunctions or is untrusted.