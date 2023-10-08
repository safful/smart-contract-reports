## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-07

# [boreWell can be frontrun/DoS-d](https://github.com/code-423n4/2023-07-basin-findings/issues/181) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/Aquifer.sol/#L48


# Vulnerability details

## Impact

The boreWell function in the Aquifer contract is responsible for creating new Wells. However, there are two critical security issues:

1. **Stealing of user's deposit amount**: The public readability of the `salt` parameter allows an attacker to frontrun a user's transaction and capture the deposit amount intended for the user's Well. By creating a Well with the same `salt` value, the attacker can receive the deposit intended for the user's Well and withdraw the funds.
2. **DoS for boreWell**: Another attack vector involves an attacker deploying a Well with the same `salt` value as the user's intended Well. This causes the user's transaction to be reverted, resulting in a denial-of-service (DoS) attack on the boreWell function. The attacker can repeatedly execute this attack, preventing users from creating new Wells.

## Proof of Concept

### Stealing of user's deposit amount

If a user intends to create a new Well and deposit funds into it, an attacker can frontrun the user's transactions and capture the deposit amount. Here is how the attack scenario unfolds:

1. The user broadcasts two transactions: the first to create a Well with a specific `salt` value, and the second to deposit funds into the newly created Well.
2. The attacker views these pending transactions and frontruns them by creating a Well for themselves using the same `salt` value.
3. The attacker's Well gets created with the same address that the user was expecting for their Well.
4. As a result, the user's create Well transaction gets reverted, but the deposit transaction successfully executes, depositing the funds into the attacker's Well.
5. Being the owner of the Well, the attacker can simply withdraw the deposited funds from the Well.

### DoS for boreWell

In this attack scenario, an attacker can forcefully revert a user's create Well transaction by deploying a Well for themselves using the user's `salt` value. Here are the steps of the attack:

1. The user broadcasts a create Well transaction with a specific `salt` value.
2. The attacker frontruns the user's transaction and creates a Well for themselves using the same `salt` value.
3. As a result, the user's original create Well transaction gets reverted since the attacker's Well already exists at the predetermined address.
4. This attack can be repeated multiple times, effectively causing a denial-of-service (DoS) attack on the boreWell function.

## Tools Used

vscode

## Recommended Mitigation Steps

To mitigate the identified security issues, it is recommended to make the upcoming Well address user-specific by combining the `salt` value with the user's address. This ensures that each user's Well has a unique address and prevents frontrunning attacks and DoS attacks. The following code snippet demonstrates the recommended modification:

```
well = implementation.cloneDeterministic(
    keccak256(abi.encode(msg.sender, `salt`))
);

```


## Assessed type

Other