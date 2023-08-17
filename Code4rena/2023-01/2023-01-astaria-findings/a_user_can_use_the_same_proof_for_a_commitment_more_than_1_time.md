## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-01

# [A user can use the same proof for a commitment more than 1 time](https://github.com/code-423n4/2023-01-astaria-findings/issues/589) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L490
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L761
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L287
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L379
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L229
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L178
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L434
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/strategies/UNI_V3Validator.sol#L80


# Vulnerability details

## Impact
A user can use the same commitment signature and merkleData more than 1 time to obtain another loan.
## Proof of Concept

A user needs to make some procedures to take a loan against an NFT.
Normally the user calls commitToLiens() in AstariaRouter.sol providing IAstariaRouter.Commitment[] as input.
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L490

The function will transfer the NFT to the CollateralToken contract to get a collateral id for the user (see CollateralToken.onERC721Received() ). 

After that, the function in the router will make an internal call _executeCommitment() for each commitment.
_executeCommitment() checks the collateral id and the commitment.lienRequest.strategy.vault and then calls commitToLien in 
commitment.lienRequest.strategy.vault with the commitment and the router address as the receiver (ie: address (this)).

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L761

_executeCommitment() in AstariaRouter.sol

```solidity
  function _executeCommitment(
    RouterStorage storage s,
    IAstariaRouter.Commitment memory c
  )
    internal
    returns (
      uint256,
      ILienToken.Stack[] memory stack,
      uint256 payout
    )
  {
    uint256 collateralId = c.tokenContract.computeId(c.tokenId);

    if (msg.sender != s.COLLATERAL_TOKEN.ownerOf(collateralId)) {
      revert InvalidSenderForCollateral(msg.sender, collateralId);
    }
    if (!s.vaults[c.lienRequest.strategy.vault]) {
      revert InvalidVault(c.lienRequest.strategy.vault);
    }
    //router must be approved for the collateral to take a loan,
    return
      IVaultImplementation(c.lienRequest.strategy.vault).commitToLien( // HERE
        c,
        address(this)
      );
  }
```

commitToLien() function in VaultImplementation:

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L287

```solidity
  function commitToLien( 
    IAstariaRouter.Commitment calldata params,
    address receiver
  )
    external
    whenNotPaused
    returns (uint256 lienId, ILienToken.Stack[] memory stack, uint256 payout)
  {
    _beforeCommitToLien(params);
    uint256 slopeAddition;
    (lienId, stack, slopeAddition, payout) = _requestLienAndIssuePayout(
      params,
      receiver
    );
    _afterCommitToLien(
      stack[stack.length - 1].point.end,
      lienId,
      slopeAddition
    );
  }
```



The function in the Vault will among other things request a lien position and issue the payout to the user requesting the loan via
_requestLienAndIssuePayout().
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L379

This function validates the commitment with the signature against the address of the owner of the vault or the delegate.
To recover the address with ecrecover an auxiliary function is used to encode the strategy data (_encodeStrategyData() )  but the strategy.vault is not being taken into account in the bytes returned nor the strategy. version.

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L229


```solidity
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



So _validateCommitment will not revert if I use it with little changes in the Commitment like strategy.vault because there are not going to be changes in _encodeStrategyData(). 
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L178

We saw that the user started the operation in Router but he could have called directly commitToLien in the vault after manually transferring the NFT to Collateral.sol and obtaining the collateral token. Then strategy.vault and strategy.version can be changed by the user breaking the invariant  ***commitment.strategy.vault == vault where commitToLiens()  is being called*** and the vault will not notice it and will consider the signature of the commitment as valid.

So the procedure will be the following:

- User takes a loan against an NFT via Router with a valid commitment

- User uses the same commitment but takes a loan manually by calling the Vault and CollateralToken contracts changing 2 things:

    1. The Commitment.lienRequest.strategy.vault, will make a different newLienId = uint256(keccak256(abi.encode(params.lien))); from the 
      previous one. If this change is not made the transaction will fail to try to mint the same id. 

    2. The other change needed to avoid the transaction from failing is the lienRequest.stack provided because the LienToken contract keeps in 
       his storage a track of collateralId previous transactions ( mapping(uint256 => bytes32) collateralStateHash;). So the only thing we need to do 
       is add to stack the previous transaction to avoid validateStack() from reverting. 

```solidity
  modifier validateStack(uint256 collateralId, Stack[] memory stack) {
    LienStorage storage s = _loadLienStorageSlot();
    bytes32 stateHash = s.collateralStateHash[collateralId];
    if (stateHash == bytes32(0) && stack.length != 0) {
      revert InvalidState(InvalidStates.EMPTY_STATE);
    }
    if (stateHash != bytes32(0) && keccak256(abi.encode(stack)) != stateHash) {
      revert InvalidState(InvalidStates.INVALID_HASH);
    }
    _;
  }
```


- The signature check via ecrecover done in VaultImplementation will succeed both times (explained before why).

There´s one more check of the commitment to verify: the StrategyValidator one with the nlrDetails and the merkle. This is done in the Router contract by _validateCommitment().

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L434

This will call validateAndParse on the validators contract (address strategyValidator = s.strategyValidators[nlrType];).

This validation and parse will depend on the validator but we have examples like UNI_V3Validator.sol

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/strategies/UNI_V3Validator.sol#L80

This function makes some checks but if those checks passed the first time they will pass the second because the changes done 
(strategy.vault and stack) are not checked here. 
The leaf returned is leaf = keccak256(params.nlrDetails), and it will be checked against the root and the proof.

A portion of code of _validateCommitment in AstariaRouter:

```solidity
(bytes32 leaf, ILienToken.Details memory details) = IStrategyValidator(
      strategyValidator
    ).validateAndParse(
        commitment.lienRequest, // validates nlrDetails
        s.COLLATERAL_TOKEN.ownerOf(
          commitment.tokenContract.computeId(commitment.tokenId)
        ),
        commitment.tokenContract,
        commitment.tokenId
      );

    if (details.rate == uint256(0) || details.rate > s.maxInterestRate) {
      revert InvalidCommitmentState(CommitmentState.INVALID_RATE);
    }

    if (details.maxAmount < commitment.lienRequest.amount) {    
      revert InvalidCommitmentState(CommitmentState.INVALID_AMOUNT);
    }

    if (
      !MerkleProofLib.verify(
        commitment.lienRequest.merkle.proof,
        commitment.lienRequest.merkle.root,
        leaf
      )
    ) {
      revert InvalidCommitmentState(CommitmentState.INVALID);
    }
```






## Tools Used

Manual Review


## Recommended Mitigation Steps

You can add a nonce for the user and increment it each time he takes a loan. 