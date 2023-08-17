## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- sponsor disputed
- selected for report
- M-05

# [NFT not minted when contributed via a supported payment terminal](https://github.com/code-423n4/2022-10-juicebox-findings/issues/124) 

# Lines of code

https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721Delegate.sol#L534


# Vulnerability details

## Impact
A contributor won't get an NFT they're eligible for if the payment is made through a payment terminal that's supported by the project but not by the NFT delegate.
## Proof of Concept
A Juicebox project can use multiple [payment terminals](https://info.juicebox.money/dev/learn/glossary/payment-terminal) to receive contributions ([JBController.sol#L441-L442](https://github.com/jbx-protocol/juice-contracts-v3/blob/main/contracts/JBController.sol#L441-L442)).  Payment terminals are single token payment terminals ([JBPayoutRedemptionPaymentTerminal.sol#L310](https://github.com/jbx-protocol/juice-contracts-v3/blob/main/contracts/abstract/JBPayoutRedemptionPaymentTerminal.sol#L310)) that support only one currency ([JBSingleTokenPaymentTerminal.sol#L124-L132](https://github.com/jbx-protocol/juice-contracts-v3/blob/main/contracts/abstract/JBSingleTokenPaymentTerminal.sol#L124-L132)). Since projects can have multiple terminals, they can receive payments in multiple currencies.

However, the NFT delegate supports only one currency ([JBTiered721Delegate.sol#L225](https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721Delegate.sol#L225)):
```solidity
function initialize(
  uint256 _projectId,
  IJBDirectory _directory,
  string memory _name,
  string memory _symbol,
  IJBFundingCycleStore _fundingCycleStore,
  string memory _baseUri,
  IJBTokenUriResolver _tokenUriResolver,
  string memory _contractUri,
  JB721PricingParams memory _pricing,
  IJBTiered721DelegateStore _store,
  JBTiered721Flags memory _flags
) public override {
  // Make the original un-initializable.
  require(address(this) != codeOrigin);
  // Stop re-initialization.
  require(address(store) == address(0));

  // Initialize the sub class.
  JB721Delegate._initialize(_projectId, _directory, _name, _symbol);

  fundingCycleStore = _fundingCycleStore;
  store = _store;
  pricingCurrency = _pricing.currency; // @audit only one currency is supported
  pricingDecimals = _pricing.decimals;
  prices = _pricing.prices;

  ...
}
```

When a payment is made in a currency that's supported by the project (via one of its terminals) but not by the NFT delegate, there's an attempt to convert the currency to a supported one ([JBTiered721Delegate.sol#L527-L534](https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721Delegate.sol#L527-L534)):
```solidity
if (_data.amount.currency == pricingCurrency) _value = _data.amount.value;
else if (prices != IJBPrices(address(0)))
  _value = PRBMath.mulDiv(
    _data.amount.value,
    10**pricingDecimals,
    prices.priceFor(_data.amount.currency, pricingCurrency, _data.amount.decimals)
  );
else return;
```

However, since `prices` is optional (it can be set to the zero address, as seen from the snippet), the conversion step can be skipped. When this happens, the contributor gets no NFT due to the early `return` even though the amount of their contribution might still be eligible for a tiered NFT.
## Tools Used
Manual review
## Recommended Mitigation Steps
Short term, consider reverting when a different currency is used and `prices` is not set. Long term, consider supporting multiple currencies in the NFT delegate. 