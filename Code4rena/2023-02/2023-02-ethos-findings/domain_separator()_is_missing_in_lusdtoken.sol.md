## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-07

# [DOMAIN_SEPARATOR() is missing in LUSDToken.sol](https://github.com/code-423n4/2023-02-ethos-findings/issues/638) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/LUSDToken.sol#L28


# Vulnerability details

## Impact
The DOMAIN_SEPARATOR() function in ERC2612 is an important part of the security of the standard. It is used to prevent replay attacks, which occur when a malicious user records a valid signed message and later sends it again to fraudulently perform an action on behalf of the original signer.

The DOMAIN_SEPARATOR() is generated based on specific contract parameters, including the contract's address, the chain ID, and a unique identifier. These parameters ensure that the domain separator is unique to the contract and the chain, and prevent attackers from using the same signature on a different chain or contract.

If the DOMAIN_SEPARATOR() function is missing from ERC2612, it can significantly impact the security of the standard. It can make it easier for attackers to replay valid signatures, since the domain separator provides a crucial part of the uniqueness and security of the signature.

Therefore, it's important to ensure that the DOMAIN_SEPARATOR() function is included and properly implemented in any contract that uses ERC2612.

## Proof of Concept
Output form slither:

```solidity
# Check LUSDToken

## Check functions
[✓] permit(address,address,uint256,uint256,uint8,bytes32,bytes32) is present
        [✓] permit(address,address,uint256,uint256,uint8,bytes32,bytes32) -> () (correct return type)
[✓] nonces(address) is present
        [✓] nonces(address) -> (uint256) (correct return type)
        [✓] nonces(address) is view
[ ] DOMAIN_SEPARATOR() is missing
[✓] totalSupply() is present
        [✓] totalSupply() -> (uint256) (correct return type)
        [✓] totalSupply() is view
[✓] balanceOf(address) is present
        [✓] balanceOf(address) -> (uint256) (correct return type)
        [✓] balanceOf(address) is view
[✓] transfer(address,uint256) is present
        [✓] transfer(address,uint256) -> (bool) (correct return type)
        [✓] Transfer(address,address,uint256) is emitted
[✓] transferFrom(address,address,uint256) is present
        [✓] transferFrom(address,address,uint256) -> (bool) (correct return type)
        [✓] Transfer(address,address,uint256) is emitted
[✓] approve(address,uint256) is present
        [✓] approve(address,uint256) -> (bool) (correct return type)
        [✓] Approval(address,address,uint256) is emitted
[✓] allowance(address,address) is present
        [✓] allowance(address,address) -> (uint256) (correct return type)
        [✓] allowance(address,address) is view
[✓] name() is present
        [✓] name() -> (string) (correct return type)
        [✓] name() is view
[✓] symbol() is present
        [✓] symbol() -> (string) (correct return type)
        [✓] symbol() is view
[✓] decimals() is present
        [✓] decimals() -> (uint8) (correct return type)
        [✓] decimals() is view
    )
```

## Tools Used
VS Code, Slither

## Recommended Mitigation Steps
To mitigate this risk, it is recommended to follow the ERC2612 specification strictly and ensure that the DOMAIN_SEPARATOR is correctly implemented. 