## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-06

# [Public to all funds escape](https://github.com/code-423n4/2022-11-looksrare-findings/issues/277) 

# Lines of code

https://github.com/code-423n4/2022-11-looksrare/blob/main/contracts/TokenRescuer.sol#L22
https://github.com/code-423n4/2022-11-looksrare/blob/main/contracts/TokenRescuer.sol#L34
https://github.com/code-423n4/2022-11-looksrare/blob/main/contracts/LooksRareAggregator.sol#L27
https://github.com/code-423n4/2022-11-looksrare/blob/main/contracts/LooksRareAggregator.sol#L108
https://github.com/code-423n4/2022-11-looksrare/blob/main/contracts/LooksRareAggregator.sol#L109
https://github.com/code-423n4/2022-11-looksrare/blob/main/contracts/LooksRareAggregator.sol#L245
https://github.com/code-423n4/2022-11-looksrare/blob/main/contracts/lowLevelCallers/LowLevelETH.sol#L43


# Vulnerability details

## Description

The `LooksRareAggregator` smart contract implements a bunch of functions to escape funds by the contract owner (see `rescueETH`, `rescueERC20`, `rescueERC721`, and `rescueERC1155`). In this way, any funds that were accidentally sent to the contract or were locked due to incorrect contract implementation can be returned to the owner. However, locked funds can be rescued by anyone without the owner's permission. This is completely contrary to the idea of having rescue functions.

In order to withdraw funds from the contract, a user may just call the `execute` function in the `ERC20EnabledLooksRareAggregator` with `tokenTransfers` that contain the addresses of tokens to be withdrawn. Thus, after the order execution `_returnERC20TokensIfAny` and `_returnETHIfAny` will be called, and the whole balance of provided ERC20 tokens and Ether will be returned to `msg.sender`.

Please note, that means that the owner can be front-ran with `rescue` functions and an attacker will receive funds instead.

## Impact

Useless of rescue functionality and vulnerability to jamming funds.

## Recommended Mitigation Steps

`_returnETHIfAny` and `_returnERC20TokensIfAny` should return the amount of the token that was deposited.
