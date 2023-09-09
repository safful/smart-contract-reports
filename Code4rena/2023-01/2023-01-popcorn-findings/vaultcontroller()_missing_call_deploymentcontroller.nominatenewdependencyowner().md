## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-08

# [VaultController() Missing call DeploymentController.nominateNewDependencyOwner()](https://github.com/code-423n4/2023-01-popcorn-findings/issues/579) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/DeploymentController.sol#L121-L125


# Vulnerability details

## Impact
Unable to switch to a new deploymentController

## Proof of Concept

The current protocol supports the replacement of the new DeploymentController, which can be switched by VaultController.setDeploymentController()

Normally, when switching, the owner of the cloneFactory/cloneRegistry/templateRegistry in the old DeploymentController should also be switched to the new DeploymentController
DeploymentController's nominateNewDependencyOwner() implementation is as follows:
```solidity
  /**
   * @notice Nominates a new owner for dependency contracts. Caller must be owner. (`VaultController` via `AdminProxy`)
   * @param _owner The new `DeploymentController` implementation
   */
  function nominateNewDependencyOwner(address _owner) external onlyOwner {
    IOwned(address(cloneFactory)).nominateNewOwner(_owner);
    IOwned(address(cloneRegistry)).nominateNewOwner(_owner);
    IOwned(address(templateRegistry)).nominateNewOwner(_owner);
  }
```

But there is a problem here, VaultConttroler.sol does not implement the code to call old_Deployerment.nominateNewDependencyOwner(), resulting in DeploymentController can not switch properly, nominateNewDependencyOwner()'s Remarks:
``` Caller must be owner. (`VaultController` via `AdminProxy`)```
But in fact the VaultController does not have any code to call the nominateNewDependencyOwner
```solidity
  function setDeploymentController(IDeploymentController _deploymentController) external onlyOwner {
    _setDeploymentController(_deploymentController);
  }

  function _setDeploymentController(IDeploymentController _deploymentController) internal {
    if (address(_deploymentController) == address(0) || address(deploymentController) == address(_deploymentController))
      revert InvalidDeploymentController(address(_deploymentController));

    emit DeploymentControllerChanged(address(deploymentController), address(_deploymentController));

    deploymentController = _deploymentController;
    cloneRegistry = _deploymentController.cloneRegistry();
    templateRegistry = _deploymentController.templateRegistry();
  }
```


## Tools Used

## Recommended Mitigation Steps
setDeploymentController() need call nominateNewDependencyOwner()

```solidity
contract VaultController is Owned {
  function setDeploymentController(IDeploymentController _deploymentController) external onlyOwner {
    
+    //1. old deploymentController nominateNewDependencyOwner
+    (bool success, bytes memory returnData) = adminProxy.execute(
+       address(deploymentController),
+       abi.encodeWithSelector(
+         IDeploymentController.nominateNewDependencyOwner.selector,
+         _deploymentController
+       )
+     );
+     if (!success) revert UnderlyingError(returnData); 
+
+    //2. new deploymentController acceptDependencyOwnership
+    _deploymentController.acceptDependencyOwnership(); 

     _setDeploymentController(_deploymentController);        
  }

}
```