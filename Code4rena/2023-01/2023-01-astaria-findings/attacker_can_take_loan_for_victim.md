## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- H-21

# [Attacker can take loan for Victim](https://github.com/code-423n4/2023-01-astaria-findings/issues/19) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/VaultImplementation.sol#L239


# Vulnerability details

## Impact
An unapproved, non-owner of collateral can still take loan for the owner/operator of collateral even when owner did not needed any loan. This is happening due to incorrect checks as shown in POC. This leads to unintended loan and associated fees for users 

## Proof of Concept

1. A new loan is originated via `commitToLien` function by User X. Params used by User X are as below:

```
collateralId = params.tokenContract.computeId(params.tokenId) = 1

CT.ownerOf(1) = User Y

CT.getApproved(1) = User Z

CT.isApprovedForAll(User Y, User X) = false

receiver = User Y
```

2. This internally make call to `_requestLienAndIssuePayout` which then calls `_validateCommitment` function for signature verification
3. Lets see the signature verification part in `_validateCommitment` function

```
  function _validateCommitment(
    IAstariaRouter.Commitment calldata params,
    address receiver
  ) internal view {
    uint256 collateralId = params.tokenContract.computeId(params.tokenId);
    ERC721 CT = ERC721(address(COLLATERAL_TOKEN()));
    address holder = CT.ownerOf(collateralId);
    address operator = CT.getApproved(collateralId);
    if (
      msg.sender != holder &&
      receiver != holder &&
      receiver != operator &&
      !CT.isApprovedForAll(holder, msg.sender)
    ) {
      revert InvalidRequest(InvalidRequestReason.NO_AUTHORITY);
    }
    VIData storage s = _loadVISlot();
    address recovered = ecrecover(
      keccak256(
        _encodeStrategyData(
          s,
          params.lienRequest.strategy,
          params.lienRequest.merkle.root
        )
      ),
      params.lienRequest.v,
      params.lienRequest.r,
      params.lienRequest.s
    );
    if (
      (recovered != owner() && recovered != s.delegate) ||
      recovered == address(0)
    ) {
      revert IVaultImplementation.InvalidRequest(
        InvalidRequestReason.INVALID_SIGNATURE
      );
    }
  }
```

4. Ideally the verification should fail since :

a. User X is not owner of passed collateral
b. User X is not approved for this collateral
c. User X is not approved for all of User Y token

5. But observe the below if condition doing the required check:

```
uint256 collateralId = params.tokenContract.computeId(params.tokenId);
    ERC721 CT = ERC721(address(COLLATERAL_TOKEN()));
    address holder = CT.ownerOf(collateralId);
    address operator = CT.getApproved(collateralId);
if (
      msg.sender != holder &&
      receiver != holder &&
      receiver != operator &&
      !CT.isApprovedForAll(holder, msg.sender)
    ) {
      revert InvalidRequest(InvalidRequestReason.NO_AUTHORITY);
    }
```

6. In our case this if condition does not execute since receiver = holder

```
if (
      msg.sender != holder && // true since User X is not the owner
      receiver != holder && // false since attacker passed receiver as User Y which is owner of collateral, thus failing this if condition 
      receiver != operator &&
      !CT.isApprovedForAll(holder, msg.sender)
    ) {
      revert InvalidRequest(InvalidRequestReason.NO_AUTHORITY);
    }
```

7. This means the signature verification passes and loan is issued for collateral owner without his wish

## Recommended Mitigation Steps
Revise the condition as shown below:

```
if (
      msg.sender != holder &&
      msg.sender != operator &&
      !CT.isApprovedForAll(holder, msg.sender)
    ) {
      revert InvalidRequest(InvalidRequestReason.NO_AUTHORITY);
    }

if (
      receiver != holder &&
      receiver != operator 
    ) {
      revert InvalidRequest(InvalidRequestReason.NO_AUTHORITY);
    }
```