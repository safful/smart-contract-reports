## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-02

# [User transactions can call system contracts directly](https://github.com/code-423n4/2023-03-zksync-findings/issues/146) 

# Lines of code

https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/libraries/EfficientCall.sol#L141-L144


# Vulnerability details

## Impact
User transaction can call system contracts directly, which shouldn't be allowed to not invoke potentially dangerous operations.
## Proof of Concept
The [DefaultAccount.executeTransaction](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/DefaultAccount.sol#L117) executes a user transaction after it was validated. The function calls [_execute](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/DefaultAccount.sol#L140) under the hood. The `_execute` function makes two different calls depending on the destination address of a transaction:
1. if the `ContractDeployer` is called, it'll [pass the call to the contract](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/DefaultAccount.sol#L149) via the system call (`ContractDeployer` is a system contract and can be executed only via system calls);
1. if any other contract is called, it'll [execute the call via `EfficientCall.rawCall`](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/DefaultAccount.sol#L151).

`EfficientCall.rawCall` in its turn also makes two different calls:
1. If `msg.value` of the transaction is 0, it'll [make a regular call](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/libraries/EfficientCall.sol#L131).
1. If there's some ETH sent with the transaction (i.e. `msg.value` is positive), it'll [pass the call to the `MsgValueSimulator` contract](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/libraries/EfficientCall.sol#L144). `MsgValueSimulator` is a system contract, thus the `isSystem` flag will be set in the far call ABI (notice the `true` in the [last argument of `_loadFarCallABIIntoActivePtr`](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/libraries/EfficientCall.sol#L134)). However, it'll also [set the forward mask](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/libraries/EfficientCall.sol#L141) to 1 (the value of [MSG_VALUE_SIMULATOR_IS_SYSTEM_BIT](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/Constants.sol#L71)). `MsgValueSimulator` will [extract the mask and will set the `isSystemCall` flag to true](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/MsgValueSimulator.sol#L27)–it'll then pass the `isSystemCall` flag to the [subsequent call](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/MsgValueSimulator.sol#L62), making the call a system one.

To sum it up, if a transaction calls a contract that's not `ContractDeployer` and sends ETH, the call will be a system one, which will let it call the system contracts. However, users shouldn't be allowed to call system contracts directly to not invoke potentially dangerous operations. As per the documentation:
> Some of the system contracts can act on behalf of the user or have a very important impact on the behavior of the account. That's why we wanted to make it clear that users can not invoke potentially dangerous operations by doing a simple EVM-like call. Whenever a user wants to invoke some of the operations which we considered dangerous, they must explicitly provide isSystem flag with them.

However, since most system contracts are harmless, there's no direct high severity impact on the system, thus I think the issue is a medium severity.
## Tools Used
Manual review
## Recommended Mitigation Steps
In the `EfficientCall.rawCall` function, consider setting the forward mask to 0. The behaviour of the function is similar to that of the `msgValueSimulatorMimicCall` function from the bootloader:
1. since the `MsgValueSimulator` contract is called, the `isSystemCall` flag should be set [only for this call](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/bootloader/bootloader.yul#L1737);
1. the `isSystemCall` flag should be forwarded by `MsgValueSimulator` [only if the destination contract is `ContractDeployer`](https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/bootloader/bootloader.yul#L1730).

It looks that the second part of the `EfficientCall.rawCall` function was copied from the `SystemContractsCaller.systemCall` function, which is intended to call system contracts and which sets the forward mask to 1 when calling `MsgValueSimulator`. However, `rawCall` shouldn't forward the `isSystemCall` flag.