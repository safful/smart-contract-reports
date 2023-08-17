## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- M-02

# [Solmate's ERC20 does not check for token contract's existence, which opens up possibility for a honeypot attack](https://github.com/code-423n4/2022-11-size-findings/issues/48) 

# Lines of code

https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L163


# Vulnerability details

## Description

When bidding, the contract pulls the quote token from the bidder to itself.

https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L163

```solidity=
SafeTransferLib.safeTransferFrom(ERC20(a.params.quoteToken), msg.sender, address(this), quoteAmount);
```

However, since the contract uses Solmate's [SafeTransferLib](https://github.com/transmissions11/solmate/blob/main/src/utils/SafeTransferLib.sol#L9)

> /// @dev Note that none of the functions in this library check that a token has code at all! That responsibility is delegated to the caller.

Therefore if the token address is empty, the transfer will succeed silently, but not crediting the contract with any tokens. 

This error opens up room for a honeypot attack similar to the [Qubit Finance hack](https://halborn.com/explained-the-qubit-hack-january-2022/) in January 2022.

In particular, it has became popular for protocols to deploy their token across multiple networks using the same deploying address, so that they can control the address nonce and ensuring a consistent token address across different networks.

E.g. **1INCH** has the same token address on Ethereum and BSC, and GEL token has the same address on Ethereum, Fantom and Polygon. There are other protocols have same contract addresses across different chains, and it's not hard to imagine such thing for their protocol token too, if any.

## Proof of Concept

Assuming that Alice is the attacker, Bob is the victim. Alice has two accounts, namely Alice1 and Alice2. Denote token Q as the quote token.

0. **Token Q has been launched on other chains, but not on the local chain yet.** It is expected that token Q will be launched on the local chain with a consistent address as other chains.
1. Alice1 places an auction with some `baseToken`, and a token Q as the `quoteToken`. 
2. Alice2 places a bid with `1000e18` quote tokens.
    - The transfer succeeds, but the contract is not credited any tokens.
3. **Token Q is launched on the local chain.**
4. Bob places a bid with `1001e18` quote tokens.
5. Alice2 cancels her own bid, recovering her (supposedly) `1000e18` refund bid back.
6. Alice1 cancels her auction, recovering her base tokens back.

As a result, the contract's Q token balance is `1e18`, Alice gets away with all her base tokens and `1000e18` Q tokens that are Bob's. Alice has stolen Bob's funds.

## Tools Used

Manual review

## Recommended Mitigation Steps

Consider using OpenZeppelin's SafeERC20 instead, which has checks that an address does indeed have a contract.
