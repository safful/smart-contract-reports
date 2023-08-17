## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- selected for report
- investigate
- fix token (sponsor)
- M-20

# [TokenggAVAX: maxDeposit and maxMint return wrong value when contract is paused](https://github.com/code-423n4/2022-12-gogopool-findings/issues/144) 

# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L156-L162


# Vulnerability details

## Impact
The `TokenggAVAX` contract ([https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L24](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L24)) can be paused.  

The `whenTokenNotPaused` modifier is applied to the following functions ([https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L225-L239](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L225-L239)):  
`previewDeposit`, `previewMint`, `previewWithdraw` and `previewRedeem`  

Thereby any calls to functions that deposit or withdraw funds revert.  

There are two functions (`maxWithdraw` and `maxRedeem`) that calculate the max amount that can be withdrawn or redeemed respectively.  

Both functions return `0` if the `TokenggAVAX` contract is paused.  

The issue is that `TokenggAVAX` does not override the `maxDeposit` and `maxMint` functions ([https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L156-L162](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L156-L162)) in the `ERC4626Upgradable` contract like it does for `maxWithdraw` and `maxRedeem`.  

Thereby these two functions return a value that cannot actually be deposited or minted.  

This can cause any components that rely on any of these functions to return a correct value to malfunction.  

So `maxDeposit` and `maxMint` should return the value `0` when `TokenggAVAX` is paused.  

## Proof of Concept
1. The `TokenggAVAX` contract is paused by calling `Ocyticus.pauseEverything` ([https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Ocyticus.sol#L37-L43](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Ocyticus.sol#L37-L43))
2. `TokenggAVAX.maxDeposit` returns `type(uint256).max` ([https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L157](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L157))
3. However `deposit` cannot be called with this value because it is paused (`previewDeposit` reverts because of the `whenTokenNotPaused` modifier) ([https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L44](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L44))

## Tools Used
VSCode

## Recommended Mitigation Steps
The `maxDeposit` and `maxMint` functions should be overridden by `TokenggAVAX` just like `maxWithdraw` and `maxRedeem` are overridden and return `0` when the contract is paused ([https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L206-L223](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L206-L223)).  

So add these two functions to the `TokenggAVAX` contract:  
```solidity
function maxDeposit(address) public view override returns (uint256) {
    if (getBool(keccak256(abi.encodePacked("contract.paused", "TokenggAVAX")))) {
        return 0;
    }
    return return type(uint256).max;
}

function maxMint(address) public view override returns (uint256) {
    if (getBool(keccak256(abi.encodePacked("contract.paused", "TokenggAVAX")))) {
        return 0;
    }
    return return type(uint256).max;
}
```