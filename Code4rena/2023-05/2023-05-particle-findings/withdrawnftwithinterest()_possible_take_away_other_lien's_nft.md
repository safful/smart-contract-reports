## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-05

# [withdrawNftWithInterest() possible take away other Lien's NFT](https://github.com/code-423n4/2023-05-particle-findings/issues/13) 

# Lines of code

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L183


# Vulnerability details

## Impact
Possible take away other Lien's NFT

## Proof of Concept
`withdrawNftWithInterest()` Used to retrieve NFT
The only current restriction is that if you can transfer out of NFT, it means an inactive loan

```solidity
    function withdrawNftWithInterest(Lien calldata lien, uint256 lienId) external override validateLien(lien, lienId) {
        if (msg.sender != lien.lender) {
            revert Errors.Unauthorized();
        }

        // delete lien
        delete liens[lienId];

        // transfer NFT back to lender
        /// @dev can withdraw means NFT is currently in contract without active loan,
        /// @dev the interest (if any) is already accured to lender at NFT acquiring time
        IERC721(lien.collection).safeTransferFrom(address(this), msg.sender, lien.tokenId);
...

```

However, the current protocol does not restrict the existence of only one Lien in the same NFT

For example, the following scenario.

1. alice transfer NFT_A and supply Lien[1]
2. bob executes `sellNftToMarket()`
3. jack buys NFT_A from the market
4. jack transfers NFT_A and supply Lien[2]
5. alice executing withdrawNftWithInterest(1) is able to get NFT_A successfully (because step 4 NFT_A is already in the contract)
This results in the deletion of lien[1], and Lien[2]'s NFT_A is transferred away

The result is: jack's NFT is lost, and bob's funds are also lost


## Tools Used

## Recommended Mitigation Steps
Need to determine whether there is a Loan
```solidity
    function withdrawNftWithInterest(Lien calldata lien, uint256 lienId) external override validateLien(lien, lienId) {
        if (msg.sender != lien.lender) {
            revert Errors.Unauthorized();
        }

+       require(lien.loanStartTime == 0,"Active Loan");
```


## Assessed type

Context