## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-03

# [DefaultAccount#fallback lack payable](https://github.com/code-423n4/2023-03-zksync-findings/issues/93) 

# Lines of code

https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/DefaultAccount.sol#L223-L228


# Vulnerability details

## Impact
fallback lack `payable`,will lead to differences from the mainnet, and many existing protocols may not work

## Proof of Concept
DefaultAccount Defined as follows:
```
DefaultAccount

The implementation of the default account abstraction. This is the code that is used by default for all addresses that are not in kernel space and have no contract deployed on them. This address:

Contains the minimal implementation of our account abstraction protocol. Note that it supports the built-in paymaster flows.
When anyone (except bootloader) calls/delegate calls it, it behaves in the same way as a call to an EOA, i.e. it always returns success = 1, returndatasize = 0 for calls from anyone except for the bootloader.
```
If there is no code for the address, the DefaultAccount #fallback method will be executed, which is compatible with the behavior of the mainnet

But At present, `fallback` is not `payable`
The code is as follows
```solidity
contract DefaultAccount is IAccount {
    using TransactionHelper for *;
..
    fallback() external { //<--------without payable
        // fallback of default account shouldn't be called by bootloader under no circumstances
        assert(msg.sender != BOOTLOADER_FORMAL_ADDRESS);

        // If the contract is called directly, behave like an EOA
    }

    receive() external payable {
        // If the contract is called directly, behave like an EOA
    }

```
which will lead to differences from the mainnet.

For example, the mainnet execution of the method with value will return true, and the corresponding value will be transfer
but DefaultAccount.sol will return false


It is quite common for `call ()` to with `value`. If it is not compatible, many existing protocols may not work

Mainnet code example, executing 0x0 call with value can be successful:
```solidity
  function test() external {
    vm.deal(address(this),1000);
    console.log("before value:",address(0x0).balance);
    (bool result,bytes memory datas) = address(0x0).call{value:10}("abc");
    console.log("call result:",result);
    console.log("after value:",address(0x0).balance);
  }
```
```console
$ forge test -vvv

[PASS] test() (gas: 42361)
Logs:
  before value: 0
  call result: true
  after value: 10
```

Simulate DefaultAccount #fallback without payable, it will fail:

```solidity

    contract DefaultAccount {
        fallback() external {     
        }
        receive() external payable {      
        }
    }

    function test() external {
        DefaultAccount defaultAccount = new DefaultAccount();
        vm.deal(address(this),1000);
        console.log("before value:",address(defaultAccount).balance);
        (bool result,bytes memory datas) = address(defaultAccount).call{value:10}("abc");
        console.log("call result:",result);
        console.log("after value:",address(defaultAccount).balance);
    }
```
```console
$ forge test -vvv

[PASS] test() (gas: 62533)
Logs:
  before value: 0
  call result: false
  after value: 0
```

## Tools Used

## Recommended Mitigation Steps
```solidity
- function test() external {
+ function test() external payable {
    vm.deal(address(this),1000);
    console.log("before value:",address(0x0).balance);
    (bool result,bytes memory datas) = address(0x0).call{value:10}("abc");
    console.log("call result:",result);
    console.log("after value:",address(0x0).balance);
  }
```
