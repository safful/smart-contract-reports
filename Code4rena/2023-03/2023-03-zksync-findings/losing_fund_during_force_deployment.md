## Tags

- bug
- 2 (Med Risk)
- selected for report
- sponsor confirmed
- edited-by-warden
- M-05

# [Losing fund during force deployment](https://github.com/code-423n4/2023-03-zksync-findings/issues/64) 

# Lines of code

https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L245


# Vulnerability details

## Impact
During force deployment, if the fund is not properly transferred to the to-be-force-deployed contract, the fund will remain in the contract `ContractDeployer` and can not easily be recovered.

## Proof of Concept

The function `forceDeployOnAddresses` in contract `ContractDeployer` is used only during an upgrade to set bytecodes on specific addresses. 
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L232

The ETH sent to this function will be used to initialize to-be-force-deployed contracts. The ETH sent should be equal to the aggregated value needed for each contract.
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L240

Then the function externally calls itself, and send the required value to itself.
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L245

If any of this call is unsuccessful, the whole transaction will not revert, and the loop continues to deploy all the contract on the provided `newAddress`.

If for any reason, the deployment was not successful, the transferred ETH will remain in `ContractDeployer`, and can not be used for the next deployments (because the aggregated amount is compared with `msg.value` not the ETH balance of the contract). In other words, `FORCE_DEPLOYER` fund will be in `ContractDeployer`, and it can not be easily recoverred.

The possibility of unsuccessful deployment is not low: 

It can happen if the bytecode is not known already.
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L213
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L296

It can happen during storing constructing bytecode hash.
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L214
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/AccountCodeStorage.sol#L36

It can happen during constructing contract and transferring the value.
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L223

## Tools Used

## Recommended Mitigation Steps
By using try/catch, the fund can be transferred to an address that the governor has control to be used later.
```
function forceDeployOnAddresses(ForceDeployment[] calldata _deployments)
        external
        payable
    {
        // remaining of the code

        for (uint256 i = 0; i < deploymentsLength; ++i) {
            try
                this.forceDeployOnAddress{value: _deployments[i].value}(
                    _deployments[i],
                    msg.sender
                )
            {} catch {
                ETH_TOKEN_SYSTEM_CONTRACT.transferFromTo(
                    address(this),
                    SomeAddress,
                    _deployments[i].value
                );
            }
        }
    }
```