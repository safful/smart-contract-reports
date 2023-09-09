## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- selected for report
- sponsor confirmed
- M-20

# [erc777 cross function re-entrancy](https://github.com/code-423n4/2023-01-popcorn-findings/issues/453) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol?plain=1#L153


# Vulnerability details

## Summary

When an erc777 is used as asset for the vault, you can re-enter the _mint function by minting the double you would have minted with a normal erc20 token.

## Vulnerability Detail
The following 2 functions allow minting in the vault. One is for depositing a certain amount of assets, in this case, erc777 and getting shares in exchange, and the second one is to calculate the number of assets you have to send to mint "x" shares. The problem relies on the following lines:

deposit function:

       _mint(receiver, shares);

        asset.safeTransferFrom(msg.sender, address(this), assets);

        adapter.deposit(assets, address(this));


The erc777 has a callback to the owner of the tokens before doing transferFrom. In this case, it is a vulnerable function because it is minting the shares before we transfer the tokens. So, on the callback that transferFrom makes to our attacker contract, we directly call the other mint function that is also publicly callable by anyone. The reason why we can't re-enter the deposit is that it has a `nonReentrant` modifier, so we have to perform a cross-function re-entrancy.

mint function:

        _mint(receiver, shares);

        asset.safeTransferFrom(msg.sender, address(this), assets);

        adapter.deposit(assets, address(this));


So eventually, you will be able to get twice as much shares everytime you deposit.


https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol?plain=1#L134-L157
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol?plain=1#L170-L198


## Impact
Eventually you will be able to mint twice as much as you provide assets.


## Tool used
Manual Review

## Recommendation
The changes that have to be made are 2:

Either state clearly that erc777 are not supported, or
change the flow of the function, transferring first the assets and then getting the shares.

       
       asset.safeTransferFrom(msg.sender, address(this), assets); //changed to the top of the function
        _mint(receiver, shares);
        adapter.deposit(assets, address(this));