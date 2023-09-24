## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-06

# [Universal Data Key Permissions May Be Abused During Ownership Transfers](https://github.com/code-423n4/2023-06-lukso-findings/issues/98) 

# Lines of code

https://github.com/code-423n4/2023-06-lukso/blob/9dbc96410b3052fc0fd9d423249d1fa42958cae8/contracts/LSP6KeyManager/LSP6KeyManagerCore.sol#L132-L137
https://github.com/code-423n4/2023-06-lukso/blob/9dbc96410b3052fc0fd9d423249d1fa42958cae8/contracts/LSP6KeyManager/LSP6KeyManagerCore.sol#L282-L288
https://github.com/code-423n4/2023-06-lukso/blob/9dbc96410b3052fc0fd9d423249d1fa42958cae8/contracts/LSP6KeyManager/LSP6KeyManagerCore.sol#L462


# Vulnerability details

## Impact
In LSP6KeyManager, when fetching permissions. we are looking for universal permissions (independent from the owner). If a UP owner transfers ownership to a new owner that uses a key manager, the previously set permissions (like access for a lot controller) remain intact. This can potentially enable the old owner to retain significant control over the UP, which could be abused to destabilize the contract or cause financial harm to the new owner and other participants.

## Proof of Concept
The problem arises from the inability of the smart contract to identify and manage data keys that were set by previous owners. A malicious actor could set certain permissions while they are the owner of the UP, then transfer the ownership to a new owner. The old permissions would remain in effect, allowing the old owner to maintain undue control and possibly 'rug pull' or cause other harmful actions at a later date.

## Tools Used
The issue was identified through manual review of the contract mechanisms and their potential abuse, without the use of specific security tools.


## Recommended Mitigation Steps
To prevent potential abuse through residual permissions, the data keys for permissions should be made owner-specific. The following mitigation steps can be implemented:

- Hash the permission with the owner's address: When setting permissions, they can be hashed with the owner's address. This way, the permissions are specifically associated with a particular owner and do not affect subsequent owners.

- Add a nonce upon ownership transfer: To further ensure the uniqueness and irrelevance of old permissions, a random nonce can be added each time the ownership is transferred. This nonce can be used in conjunction with the address of the LSP6 when retrieving permissions, making any permissions set by old owners irrelevant.



## Assessed type

Rug-Pull