## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-14

# [No check for Individual mint amount surpassing 10% when the circulation reaches 10_000_000 in mint() of LybraEUSDVaultBase contract](https://github.com/code-423n4/2023-06-lybra-findings/issues/392) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L124
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L126


# Vulnerability details

## Impact
The mint functions in LybraEUSDVaultBase has no checks for when the supplied amount to mint is more than 10% if circulation reaches 10,000,000 as specified in the comments explaining the logic of the function. 

## Proof of Concept
Lets have a look at mint() code in LybraEUSDVaultBase contract

```
    /**
     * @notice The mint amount number of EUSD is minted to the address
     * Emits a `Mint` event.
     *
     * Requirements:
     * - `onBehalfOf` cannot be the zero address.
     * - `amount` Must be higher than 0. Individual mint amount shouldn't surpass 10% when the circulation 
          reaches 10_000_000
     */
    function mint(address onBehalfOf, uint256 amount) external {
        require(onBehalfOf != address(0), "MINT_TO_THE_ZERO_ADDRESS");
        require(amount > 0, "ZERO_MINT");
        _mintEUSD(msg.sender, onBehalfOf, amount, getAssetPrice());
    }

    function _mintEUSD(address _provider, address _onBehalfOf, uint256 _mintAmount, uint256 _assetPrice) internal virtual {
        require(poolTotalEUSDCirculation + _mintAmount <= configurator.mintVaultMaxSupply(address(this)), "ESL");
        try configurator.refreshMintReward(_provider) {} catch {}
        borrowed[_provider] += _mintAmount;

        EUSD.mint(_onBehalfOf, _mintAmount);
        _saveReport();
        poolTotalEUSDCirculation += _mintAmount;
        _checkHealth(_provider, _assetPrice);
        emit Mint(msg.sender, _onBehalfOf, _mintAmount, block.timestamp);
    }
```

From the code above, we can see there is no check prevent mint amount from being greater than 10% of 10,000,000 or more if the poolTotalEUSDCirculation is 10,000,000 or more as specified in the comments. 

## Tools Used
VS CODE

## Recommended Mitigation Steps
Add checks to the mint() to revert if mint amount is greater than 10% of the total supply, if total supply is >= 10,000,000.

```
    function mint(address onBehalfOf, uint256 amount) external {
        require(onBehalfOf != address(0), "MINT_TO_THE_ZERO_ADDRESS");
        require(amount > 0, "ZERO_MINT");

        if ( poolTotalEUSDCirculation >= 10_000_000 ) {
         require(amount <= (10 * poolTotalEUSDCirculation) / 100, 'amount greater than 10% of circulation' );
        }
        _mintEUSD(msg.sender, onBehalfOf, amount, getAssetPrice());
    }
```


## Assessed type

Error