## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-08

# [Return value of low level `call` not checked.](https://github.com/code-423n4/2023-08-goodentry-findings/issues/83) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L156
https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L174
https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L192


# Vulnerability details

## Impact
Detailed description of the impact of this finding.

The return value of low level `call` is not checked. 
If the `msg.sender` is a contract and its `receive()` function has the potential to revert, the code `payable(msg.sender).call{value:wad}("")` could potentially return a false result, which is not being verified. As a result, the calling functions may exit without successfully returning ethers to senders. 

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L147C1-L158C6
https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L160C1-L176C6
https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L178C1-L194C6

POC using Foundry:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

contract CallerContract {

    VaultContract private vaultAddress;
    constructor(VaultContract _vaultAddr) payable {
        vaultAddress = _vaultAddr;
    }

    receive() payable external {
        revert();
    }

    function claim(uint256 amount) public {
        vaultAddress.claimEther(amount);
    }

}

contract VaultContract {
    CallerContract private callerContract;
    constructor() payable {}

    receive() payable external {
        revert();
    }

    function claimEther(uint256 amount) public {
        require(amount <= 3 ether, "Cannot claim more than 3 ether");
        msg.sender.call{value:amount}("");
        //(bool sent,) = msg.sender.call{value: amount}("");
        //require(sent, "sent failed");
    }

    function sendEither() payable external {
        callerContract.claim(3 ether);
    }

    function setCaller(CallerContract _caller) public {
        callerContract = _caller;
    }

}

contract CallReturnValueNotCheckedTest is Test {

    address public Bob;
    VaultContract public vaultContract;
    CallerContract public callerContract;

    function setUp() public {
        Bob = makeAddr("Bob");
        vaultContract = new VaultContract();
        callerContract = new CallerContract(vaultContract);
        vaultContract.setCaller(callerContract);
        deal(Bob, 10 ether);
    }

    function test_callReturnNotChecked() public {
        console2.log("Before calling: Bob balance is %d ether", Bob.balance / 1e18);
        console2.log("Before calling: Vault balance is %d ether", address(vaultContract).balance / 1e18);
        console2.log("Bob sends 3 ether");
        vm.startPrank(Bob);
        //vm.expectRevert();
        vaultContract.sendEither{value: 3 ether}();
        console2.log("After  calling: Bob balance is %d ether", Bob.balance / 1e18);
        console2.log("After  calling: Vault balance is %d ether", address(vaultContract).balance / 1e18);
        vm.stopPrank();

    }
}

```
The output is:
```log
Running 1 test for test/ReturnValueTest.t.sol:CallReturnValueNotCheckedTest
[PASS] test_callReturnNotChecked() (gas: 42207)
Logs:
  Before calling: Bob balance is 10 ether
  Before calling: Vault balance is 0 ether
  Bob sends 3 ether
  After  calling: Bob balance is 7 ether
  After  calling: Vault balance is 3 ether

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 762.13µs
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The caller contract fails to receive ether, but the whole transaction is not reverted. As a result, the ether will be locked in the `Vault` contract. 

This issue can be prevented by adding logic to check the return value of `call`.
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

contract CallerContract {

    VaultContract private vaultAddress;
    constructor(VaultContract _vaultAddr) payable {
        vaultAddress = _vaultAddr;
    }

    receive() payable external {
        revert();
    }

    function claim(uint256 amount) public {
        vaultAddress.claimEther(amount);
    }

}

contract VaultContract {
    CallerContract private callerContract;
    constructor() payable {}

    receive() payable external {
        revert();
    }

    function claimEther(uint256 amount) public {
        require(amount <= 3 ether, "Cannot claim more than 3 ether");
        //msg.sender.call{value:amount}("");
        (bool sent,) = msg.sender.call{value: amount}("");
        require(sent, "sent failed");
    }

    function sendEither() payable external {
        callerContract.claim(3 ether);
    }

    function setCaller(CallerContract _caller) public {
        callerContract = _caller;
    }

}

contract CallReturnValueNotCheckedTest is Test {

    address public Bob;
    VaultContract public vaultContract;
    CallerContract public callerContract;

    function setUp() public {
        Bob = makeAddr("Bob");
        vaultContract = new VaultContract();
        callerContract = new CallerContract(vaultContract);
        vaultContract.setCaller(callerContract);
        deal(Bob, 10 ether);
    }

    function test_callReturnNotChecked() public {
        console2.log("Before calling: Bob balance is %d ether", Bob.balance / 1e18);
        console2.log("Before calling: Vault balance is %d ether", address(vaultContract).balance / 1e18);
        console2.log("Bob sends 3 ether");
        vm.startPrank(Bob);
        //vm.expectRevert();
        vaultContract.sendEither{value: 3 ether}();
        console2.log("After  calling: Bob balance is %d ether", Bob.balance / 1e18);
        console2.log("After  calling: Vault balance is %d ether", address(vaultContract).balance / 1e18);
        vm.stopPrank();

    }

    function test_callReturnChecked_revert() public {
        console2.log("Before calling: Bob balance is %d ether", Bob.balance / 1e18);
        console2.log("Before calling: Vault balance is %d ether", address(vaultContract).balance / 1e18);
        console2.log("Bob sends 3 ether");
        vm.startPrank(Bob);
        vm.expectRevert();
        vaultContract.sendEither{value: 3 ether}();
        console2.log("After  calling: Bob balance is %d ether", Bob.balance / 1e18);
        console2.log("After  calling: Vault balance is %d ether", address(vaultContract).balance / 1e18);
        vm.stopPrank();

    }
}
```

Output of modified code:
```log
Running 1 test for test/ReturnValueTest.t.sol:CallReturnValueNotCheckedTest
[PASS] test_callReturnChecked_revert() (gas: 42788)
Logs:
  Before calling: Bob balance is 10 ether
  Before calling: Vault balance is 0 ether
  Bob sends 3 ether
  After  calling: Bob balance is 10 ether
  After  calling: Vault balance is 0 ether

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.60ms
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
Now the ether is returned back if call is failed. 

## Tools Used
Foundry

## Recommended Mitigation Steps
It's recommended to check the return value to be true or just use OpenZeppelin `Address` library `sendValue()` function for ether transfer. See https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.9.3/contracts/utils/Address.sol#L64 .




## Assessed type

call/delegatecall