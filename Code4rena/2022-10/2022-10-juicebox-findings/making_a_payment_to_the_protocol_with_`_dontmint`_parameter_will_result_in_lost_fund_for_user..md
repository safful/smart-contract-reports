## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- selected for report
- H-01

# [Making a payment to the protocol with `_dontMint` parameter will result in lost fund for user.](https://github.com/code-423n4/2022-10-juicebox-findings/issues/45) 

# Lines of code

https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721Delegate.sol#L524-L590


# Vulnerability details

## Impact
User will have their funds lost if they tries to pay the protocol with `_dontMint = False`. A payment made with this parameter set should increase the `creditsOf[]` balance of user.

In `_processPayment()`, `creditsOf[_data.beneficiary]` is updated at the end if there are leftover funds. However, If `metadata` is provided and `_dontMint == true`, it immediately returns.
[JBTiered721Delegate.sol#L524-L590](https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721Delegate.sol#L524-L590)
```solidity
  function _processPayment(JBDidPayData calldata _data) internal override {
    // Keep a reference to the amount of credits the beneficiary already has.
    uint256 _credits = creditsOf[_data.beneficiary];
    ...
    if (
      _data.metadata.length > 36 &&
      bytes4(_data.metadata[32:36]) == type(IJB721Delegate).interfaceId
    ) {
      ...
      // Don't mint if not desired.
      if (_dontMint) return;
      ...
    }
    ...
    // If there are funds leftover, mint the best available with it.
    if (_leftoverAmount != 0) {
      _leftoverAmount = _mintBestAvailableTier(
        _leftoverAmount,
        _data.beneficiary,
        _expectMintFromExtraFunds
      );

      if (_leftoverAmount != 0) {
        // Make sure there are no leftover funds after minting if not expected.
        if (_dontOverspend) revert OVERSPENDING();

        // Increment the leftover amount.
        creditsOf[_data.beneficiary] = _leftoverAmount;
      } else if (_credits != 0) creditsOf[_data.beneficiary] = 0;
    } else if (_credits != 0) creditsOf[_data.beneficiary] = 0;
  }
```

## Proof of Concept
I've wrote a coded POC to illustrate this. It uses the same Foundry environment used by the project. Simply copy this function to `E2E.t.sol` to verify.

```solidity
  function testPaymentNotAddedToCreditsOf() public{
    address _user = address(bytes20(keccak256('user')));
    (
      JBDeployTiered721DelegateData memory NFTRewardDeployerData,
      JBLaunchProjectData memory launchProjectData
    ) = createData();

    uint256 projectId = deployer.launchProjectFor(
      _projectOwner,
      NFTRewardDeployerData,
      launchProjectData
    );

    // Get the dataSource
    IJBTiered721Delegate _delegate = IJBTiered721Delegate(
      _jbFundingCycleStore.currentOf(projectId).dataSource()
    );

    address NFTRewardDataSource = _jbFundingCycleStore.currentOf(projectId).dataSource();

    uint256 _creditBefore = IJBTiered721Delegate(NFTRewardDataSource).creditsOf(_user);

    // Project is initiated with 10 different tiers with contributionFee of 10,20,30,40, .... , 100

    // Make payment to mint 1 NFT
    uint256 _payAmount = 10;
    _jbETHPaymentTerminal.pay{value: _payAmount}(
      projectId,
      100,
      address(0),
      _user,
      0,
      false,
      'Take my money!',
      new bytes(0)
    );

    // Minted 1 NFT
    assertEq(IERC721(NFTRewardDataSource).balanceOf(_user), 1);

    // Now, we make the payment but supply _dontMint metadata
    bool _dontMint = true;
    uint16[] memory empty;
    _jbETHPaymentTerminal.pay{value: _payAmount}(
      projectId,
      100,
      address(0),
      _user,
      0,
      false,
      'Take my money!',
      //new bytes(0)
      abi.encode(
        bytes32(0),
        type(IJB721Delegate).interfaceId,
        _dontMint,
        false,
        false,
        empty
        )
    );

    // NFT not minted
    assertEq(IERC721(NFTRewardDataSource).balanceOf(_user), 1);

    // Check that credits of user is still the same as before even though we have made the payment
    assertEq(IJBTiered721Delegate(NFTRewardDataSource).creditsOf(_user),_creditBefore);

  }
```

## Tools Used
Foundry

## Recommended Mitigation Steps
Update the `creditsOf[]` in the `if(_dontMint)` check.

```diff
- if(_dontMint) return;
+ if(_dontMint){ creditsOf[_data.beneficiary] += _value; }
```