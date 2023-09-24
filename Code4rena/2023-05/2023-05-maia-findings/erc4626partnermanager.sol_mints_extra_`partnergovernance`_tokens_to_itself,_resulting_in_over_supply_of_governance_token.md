## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-39

# [ERC4626PartnerManager.sol mints extra `partnerGovernance` tokens to itself, resulting in over supply of governance token](https://github.com/code-423n4/2023-05-maia-findings/issues/191) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L226


# Vulnerability details

## Impact
ERC4626PartnerManager mints more tokens than needed when `bHermesRate` increased.
1) I suppose it can break voting in which this token is used. Because totalSupply is increased, more and more tokens are stuck in contract after every increasing of `bHermesRate`.
2) The second concern is that all this tokens are approved to `partnerVault` contract and can be extracted. But implementation of partnerVault is out of scope and I don't know is it possible. But this token excess exists in ERC4626PartnerManager.sol

## Proof of Concept
Token amount to mint is difference between totalSupply * newRate and balance of this contract.
```solidity
    function increaseConversionRate(uint256 newRate) external onlyOwner {
        if (newRate < bHermesRate) revert InvalidRate();

        if (newRate > (address(bHermesToken).balanceOf(address(this)) / totalSupply)) {
            revert InsufficientBacking();
        }

        bHermesRate = newRate;

        partnerGovernance.mint(
            address(this), totalSupply * newRate - address(partnerGovernance).balanceOf(address(this))
        );
        bHermesToken.claimOutstanding();
    }
```
However it is wrong to account balance of address(this) because it decreases every claim. Let me explain.

Suppose bHermesRate = 10, balance of bHermes is 50, totalSupply is 0 (nobody interacted yet)
1) User1 deposits 5 MAIA, therefore mints 5 vMAIA and mints 5 * 10 = 50 `govToken`
```solidity
    function _mint(address to, uint256 amount) internal virtual override {
        if (amount > maxMint(to)) revert ExceedsMaxDeposit();
        bHermesToken.claimOutstanding();

        ERC20MultiVotes(partnerGovernance).mint(address(this), amount * bHermesRate);
        super._mint(to, amount);
    }
```
2) Admin calls `increaseConversionRate(11)`, ie increases rate by 1. This function will mint 5 * 11 - 0 = 55 tokens, but should mint only 5 * (11 - 10) = 5
```solidity
        partnerGovernance.mint(
            address(this), totalSupply * newRate - address(partnerGovernance).balanceOf(address(this))
        );
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Refactor function to
```solidity
    function increaseConversionRate(uint256 newRate) external onlyOwner {
        if (newRate < bHermesRate) revert InvalidRate();

        if (newRate > (address(bHermesToken).balanceOf(address(this)) / totalSupply)) {
            revert InsufficientBacking();
        }

        partnerGovernance.mint(
            address(this), totalSupply * (newRate - bHermesRate)
        );

        bHermesRate = newRate;

        bHermesToken.claimOutstanding();
    }
```


## Assessed type

ERC20