## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-10

# [Owner in VaultProxy.sol is address(0)](https://github.com/code-423n4/2023-06-stader-findings/issues/133) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/VaultProxy.sol#L20-L35


# Vulnerability details

## Impact
The owner variable in the ```VaultProxy.sol``` is going to be zero because of a bug in staderConfig.
The ```intialize``` function in ```staderConfig``` grants th admin role to an admin passed in via the native ```_grantRole()```, but does not update the mapping that ```getAdmin``` reads from.
```solidity
    function initialise(
        bool _isValidatorWithdrawalVault,
        uint8 _poolId,
        uint256 _id,
        address _staderConfig
    ) external {
        if (isInitialized) {
            revert AlreadyInitialized();
        }
        UtilLib.checkNonZeroAddress(_staderConfig);
        isValidatorWithdrawalVault = _isValidatorWithdrawalVault;
        isInitialized = true;
        poolId = _poolId;
        id = _id;
        staderConfig = IStaderConfig(_staderConfig);

        //@audit-issue Admin is zero on initialize
        //address(0) is going to  be returned, wrt stader.initialize
        owner = staderConfig.getAdmin();
    }
```
https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L85-L103
https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L361-L363

The intialization of the vaults would not revert and all functions restricted to the owner would be inaccessible unless function updateAdmin is called.
https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L176-L183

The update admin does not revoke functions for the current admin in the AccessControl contract too because the address returned is address zero 
## Proof of Concept
```solidity

pragma solidity 0.8.16;

import '../../contracts/library/UtilLib.sol';

import '../../contracts/StaderConfig.sol';
import '../../contracts/VaultProxy.sol';
import '../../contracts/ValidatorWithdrawalVault.sol';
import '../../contracts/OperatorRewardsCollector.sol';

import './mocks/PoolUtilsMock.sol';
import './mocks/PenaltyMockForVault.sol';
import './mocks/SDCollateralMock.sol';
import './mocks/StakePoolManagerMock.sol';

import 'forge-std/Test.sol';
import '@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol';
import '@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol';

contract ValidatorWithdrawalVaultTest is Test {
    address staderAdmin;
    address staderManager;
    address staderTreasury;
    PoolUtilsMock poolUtils;

    uint8 poolId;
    uint256 validatorId;

    StaderConfig staderConfig;
    VaultProxy withdrawVault;
    OperatorRewardsCollector operatorRC;

    function setUp() public {
        poolId = 1;
        validatorId = 1;

        staderAdmin = vm.addr(100);
        staderManager = vm.addr(101);
        address ethDepositAddr = vm.addr(102);

        ProxyAdmin proxyAdmin = new ProxyAdmin();

        StaderConfig configImpl = new StaderConfig();
        TransparentUpgradeableProxy configProxy = new TransparentUpgradeableProxy(
            address(configImpl),
            address(proxyAdmin),
            ''
        );
        staderConfig = StaderConfig(address(configProxy));
        staderConfig.initialize(staderAdmin, ethDepositAddr);

        OperatorRewardsCollector operatorRCImpl = new OperatorRewardsCollector();
        TransparentUpgradeableProxy operatorRCProxy = new TransparentUpgradeableProxy(
            address(operatorRCImpl),
            address(proxyAdmin),
            ''
        );
        operatorRC = OperatorRewardsCollector(address(operatorRCProxy));
        operatorRC.initialize(staderAdmin, address(staderConfig));

        poolUtils = new PoolUtilsMock(address(staderConfig));
        PenaltyMockForVault penaltyContract = new PenaltyMockForVault();
        SDCollateralMock sdCollateral = new SDCollateralMock();
        ValidatorWithdrawalVault withdrawVaultImpl = new ValidatorWithdrawalVault();

        vm.startPrank(staderAdmin);
        staderConfig.updatePoolUtils(address(poolUtils));
        staderConfig.updatePenaltyContract(address(penaltyContract));
        staderConfig.updateSDCollateral(address(sdCollateral));
        staderConfig.updateOperatorRewardsCollector(address(operatorRC));
        staderConfig.updateValidatorWithdrawalVaultImplementation(address(withdrawVaultImpl));

        vm.stopPrank();

        withdrawVault = new VaultProxy();
        withdrawVault.initialise(true, poolId, validatorId, address(staderConfig));
    }

    function testAdminRights() public {
       vm.startPrank(staderAdmin);
      staderConfig.grantRole(staderConfig.MANAGER(), staderManager); 
      vm.stopPrank();
    }
    function testFetchAdminAndVaultOwner() public{
        address _admin = staderConfig.getAdmin();
        assertEq(_admin, address(0));
        address _owner = withdrawVault.owner();
        assertEq(_owner, address(0));
    }
    
}
```

## Tools Used
Foundry, manual review

## Recommended Mitigation Steps
```StaderConfig.initialize``` should add ```setAccount(ADMIN, _admin);``` a last line 


## Assessed type

Access Control