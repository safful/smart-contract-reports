## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [StakedCitadel doesn't use correct balance for internal accounting](https://github.com/code-423n4/2022-04-badger-citadel-findings/issues/74) 

# Lines of code

https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/StakedCitadel.sol#L291-L295
https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/StakedCitadel.sol#L772-L776
https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/StakedCitadel.sol#L881-L893


# Vulnerability details

## Impact
The StakedCitadel contract's `balance()` function is supposed to return the balance of the vault + the balance of the strategy. But, it only returns the balance of the vault. The balance is used to determine the number of shares that should be minted when depositing funds into the vault and the number of shares that should be burned when withdrawing funds from it.

Since most of the funds will be located in the strategy, the vault's balance will be very low. Some of the issues that arise from this:

**You can't deposit to a vault that already minted shares but has no balance of the underlying token**:

1. fresh vault with 0 funds and 0 shares
2. Alice deposits 10 tokens. She receives 10 shares back (https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/StakedCitadel.sol#L887-L888)
3. Vault's tokens are deposited into the strategy (now `balance == 0` and `totalSupply == 10`)
4. Bob tries to deposit but the transaction fails because the contract tries to divide by zero: https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/StakedCitadel.sol#L890 (`pool == balance()`)

**You get more shares than you should**
1. fresh vault with 0 funds and 0 shares
2. Alice deposits 10 tokens. She receives 10 shares back (https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/StakedCitadel.sol#L887-L888)
3. Vault's tokens are deposited into the strategy (now `balance == 0` and `totalSupply == 10`)
4. Bob now first transfers 1 token to the vault so that the balance is now `1` instead of `0`.
5. Bob deposits 5 tokens. He receives `5 * 10 / 1 == 50` shares: https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/StakedCitadel.sol#L890

Now, the vault received 15 tokens. 10 from Alice and 5 from Bob. But Alice only has 10 shares while Bob has 50. Thus, Bob can withdraw more tokens than he should be able to.

It simply breaks the whole accounting of the vault.

## Proof of Concept
The comment says that it should be vault's + strategy's balance:
https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/StakedCitadel.sol#L291-L295

Here's another vault from the badger team where the function is implemented correctly: https://github.com/Badger-Finance/badger-vaults-1.5/blob/main/contracts/Vault.sol#L262

## Tools Used
none

## Recommended Mitigation Steps
Add the strategy's balance to the return value of the`balance()` function like [here](https://github.com/Badger-Finance/badger-vaults-1.5/blob/main/contracts/Vault.sol#L262).

