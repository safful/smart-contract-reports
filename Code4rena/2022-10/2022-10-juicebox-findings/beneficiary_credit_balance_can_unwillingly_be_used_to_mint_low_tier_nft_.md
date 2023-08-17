## Tags

- bug
- 2 (Med Risk)
- sponsor acknowledged
- satisfactory
- selected for report
- M-06

# [Beneficiary credit balance can unwillingly be used to mint low tier NFT ](https://github.com/code-423n4/2022-10-juicebox-findings/issues/160) 

# Lines of code

https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721Delegate.sol#L524


# Vulnerability details

## Impact
In the function `_processPayment()`, it will use provided `JBDidPayData` from `JBPaymentTerminal` to mint to the beneficiary. The `_value` from `JBDidPayData` will be sum up with previous `_credits` balance of beneficiary. There are 2 cases that beneficiary credit balance is updated in previous payment:

1. The payment received does not meet a minting threshold or is in excess of the minted tiers, the leftover amount will be stored as credit for future minting.
2. Clients may want to accumulate to mint higher tier NFT, they might specify that the previous payment should not mint anything. (Currently it's incorrectly implemented in case `_dontMint=true`, but sponsor confirmed that it's a bug)

In both cases, an attacker can pay a small amount (just enough to mint lowest tier NFT) and specify the victim to be the beneficiary. Function `__processPayment()` will use credit balance of beneficiary from previous payment to mint low-value tier.

For example, there are 2 tiers 
1. Tier A: mintingThreshold = 20 ETH, votingUnits = 100
2. Tier B: mintingThreshold = 10 ETH, votingUnits = 10

Obviously tier A is much more better than tier B in term of voting power, so Alice (the victim) might want to accumulate her credit to mint tier A. 

Assume current credit balance `creditsOf[Alice] = 19 ETH`. Now Bob (the attacker) can pay `1 ETH` and specify Alice as beneficiary and mint `2` Tier B NFT. Alice will have to receive `2` Tier B NFT with just `20 voting power` instead of `100 voting power` for a Tier A NFT.

Since these NFTs can be used in a governance system, it may create much higher impact if this governance is used to make important decision. E.g: minting new tokens, transfering funds of community.

## Proof of Concept
Function `didPay()` only check that the caller is a terminal of the project

```solidity
function didPay(JBDidPayData calldata _data) external payable virtual override {
    // Make sure the caller is a terminal of the project, and the call is being made on behalf of an interaction with the correct project.
    if (
      msg.value != 0 ||
      !directory.isTerminalOf(projectId, IJBPaymentTerminal(msg.sender)) ||
      _data.projectId != projectId
    ) revert INVALID_PAYMENT_EVENT();

    // Process the payment.
    _processPayment(_data);
}
```

Attacker can specify any beneficiary and use previous credit balance
```solidity
// Keep a reference to the amount of credits the beneficiary already has.
uint256 _credits = creditsOf[_data.beneficiary];

// Set the leftover amount as the initial value, including any credits the beneficiary might already have.
uint256 _leftoverAmount = _value + _credits;
```


## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider adding a config param to allow others from using beneficiary's credit balance. Its value can be default to `false` for every address. And if beneficiary want to, they can toggle this state for their address to allow other using their credit balance.
