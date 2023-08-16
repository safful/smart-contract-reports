## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed
- resolved

# [Event parameters interchanged for emit of access control template addition](https://github.com/code-423n4/2021-09-sushimiso-findings/issues/73) 

# Handle

0xRajeev


# Vulnerability details

## Impact

Emission of the event AccessControlTemplateAdded(address oldAccessControl, address newAccessControl) has the old and new addresses interchanged which could confuse/trigger offchain monitoring tools or interfaces. 

This is of medium severity (instead of low) because it is related to access control template updation and critical to security of all contracts that rely on MISOAccessFactory.

The actual emit is emit AccessControlTemplateAdded(_template, accessControlTemplate); which has the parameter used in the oldAccessControl place instead of being used for the second argument, and vice-versa.

## Proof of Concept
https://github.com/sushiswap/miso/blob/2cdb1486a55ded55c81898b7be8811cb68cfda9e/contracts/Access/MISOAccessFactory.sol#L36-L37

https://github.com/sushiswap/miso/blob/2cdb1486a55ded55c81898b7be8811cb68cfda9e/contracts/Access/MISOAccessFactory.sol#L100


## Tools Used
Manual Analysis

## Recommended Mitigation Steps

Interchange the arguments in the emit.

