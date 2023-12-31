## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-23

# [Governance NFT holder, whose NFT was minted before `Trading._handleOpenFees` function is called, can lose deserved rewards after `Trading._handleOpenFees` function is called](https://github.com/code-423n4/2022-12-tigris-findings/issues/649) 

# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L689-L750
https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L762-L810
https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/GovNFT.sol#L287-L294


# Vulnerability details

## Impact
Calling the following `Trading._handleOpenFees` function does not approve the `GovNFT` contract for spending any of the `Trading` contract's `_tigAsset` balance, which is unlike calling the `Trading._handleCloseFees` function below that executes `IStable(_tigAsset).approve(address(gov), type(uint).max)`. Due to this lack of approval, when calling the `Trading._handleOpenFees` function without the `Trading._handleCloseFees` function being called for the same `_tigAsset` beforehand, the `GovNFT.distribute` function's execution of `IERC20(_tigAsset).transferFrom(_msgSender(), address(this), _amount)` in the `try...catch...` block will not transfer any `_tigAsset` amount as the trade's DAO fees to the `GovNFT` contract. In this case, although the Governance NFT holder, whose NFT was minted before the `Trading._handleOpenFees` function is called, deserves the rewards from the DAO fees generated by the trade, this holder does not have any pending rewards after such `Trading._handleOpenFees` function call because none of the DAO fees were transferred to the `GovNFT` contract. Hence, this Governance NFT holder loses the rewards that she or he is entitled to.

https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L689-L750
```solidity
    function _handleOpenFees(
        uint _asset,
        uint _positionSize,
        address _trader,
        address _tigAsset,
        bool _isBot
    )
        internal
        returns (uint _feePaid)
    {
        ...
        unchecked {
            uint _daoFeesPaid = _positionSize * _fees.daoFees / DIVISION_CONSTANT;
            ...
            IStable(_tigAsset).mintFor(address(this), _daoFeesPaid);
        }
        gov.distribute(_tigAsset, IStable(_tigAsset).balanceOf(address(this)));
    }
```

https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L762-L810
```solidity
    function _handleCloseFees(
        uint _asset,
        uint _payout,
        address _tigAsset,
        uint _positionSize,
        address _trader,
        bool _isBot
    )
        internal
        returns (uint payout_)
    {
        ...
        IStable(_tigAsset).mintFor(address(this), _daoFeesPaid);
        IStable(_tigAsset).approve(address(gov), type(uint).max);
        gov.distribute(_tigAsset, _daoFeesPaid);
        return payout_;
    }
```

https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/GovNFT.sol#L287-L294
```solidity
    function distribute(address _tigAsset, uint _amount) external {
        if (assets.length == 0 || assets[assetsIndex[_tigAsset]] == address(0) || totalSupply() == 0 || !_allowedAsset[_tigAsset]) return;
        try IERC20(_tigAsset).transferFrom(_msgSender(), address(this), _amount) {
            accRewardsPerNFT[_tigAsset] += _amount/totalSupply();
        } catch {
            return;
        }
    }
```

## Proof of Concept
Functions like `Trading.initiateMarketOrder` further call the `Trading._handleOpenFees` function so this POC uses the `Trading.initiateMarketOrder` function.

Please add the following test in the `Signature verification` `describe` block in `test\07.Trading.js`. This test will pass to demonstrate the described scenario. Please see the comments in this test for more details.
```typescript
    it.only("Governance NFT holder, whose NFT was minted before initiateMarketOrder function is called, can lose deserved rewards after initiateMarketOrder function is called", async function () {
      let TradeInfo = [parseEther("1000"), MockDAI.address, StableVault.address, parseEther("10"), 0, true, parseEther("30000"), parseEther("10000"), ethers.constants.HashZero];
      let PriceData = [node.address, 0, parseEther("20000"), 0, 2000000000, false];
      let message = ethers.utils.keccak256(
        ethers.utils.defaultAbiCoder.encode(
          ['address', 'uint256', 'uint256', 'uint256', 'uint256', 'bool'],
          [node.address, 0, parseEther("20000"), 0, 2000000000, false]
        )
      );
      let sig = await node.signMessage(
        Buffer.from(message.substring(2), 'hex')
      );
      
      let PermitData = [permitSig.deadline, ethers.constants.MaxUint256, permitSig.v, permitSig.r, permitSig.s, true];

      // one Governance NFT is minted to owner before initiateMarketOrder function is called
      const GovNFT = await deployments.get("GovNFT");
      const govnft = await ethers.getContractAt("GovNFT", GovNFT.address);
      await govnft.connect(owner).mint();

      // calling initiateMarketOrder function attempts to send 10000000000000000000 tigAsset as DAO fees to GovNFT contract
      await expect(trading.connect(owner).initiateMarketOrder(TradeInfo, PriceData, sig, PermitData, owner.address))
        .to.emit(trading, 'FeesDistributed')
        .withArgs(stabletoken.address, "10000000000000000000", "0", "0", "0", ethers.constants.AddressZero);

      // another Governance NFT is minted to owner and then transferred to user after initiateMarketOrder function is called
      await govnft.connect(owner).mint();
      await govnft.connect(owner).transferFrom(owner.getAddress(), user.getAddress(), 1);

      // user's pending reward amount should be 0 because her or his Governance NFT was minted after initiateMarketOrder function was called
      expect(await govnft.pending(user.getAddress(), stabletoken.address)).to.equal("0");

      // owner's Governance NFT was minted before initiateMarketOrder function was called so her or his pending reward amount should be 10000000000000000000.
      // However, owner's pending reward amount is still 0 because DAO fees were not transferred to GovNFT contract successfully.
      expect(await govnft.pending(owner.getAddress(), stabletoken.address)).to.equal("0");
    });
```

Furthermore, as a suggested mitigation, please add `IStable(_tigAsset).approve(address(gov), type(uint).max);` in the `_handleOpenFees` function as follows in line 749 of `contracts\Trading.sol`.
```solidity
689:     function _handleOpenFees(
690:         uint _asset,
691:         uint _positionSize,
692:         address _trader,
693:         address _tigAsset,
694:         bool _isBot
695:     )
696:         internal
697:         returns (uint _feePaid)
698:     {
699:         IPairsContract.Asset memory asset = pairsContract.idToAsset(_asset);
...
732:         unchecked {
733:             uint _daoFeesPaid = _positionSize * _fees.daoFees / DIVISION_CONSTANT;
734:             _feePaid =
735:                 _positionSize
736:                 * (_fees.burnFees + _fees.botFees) // get total fee%
737:                 / DIVISION_CONSTANT // divide by 100%
738:                 + _daoFeesPaid;
739:             emit FeesDistributed(
740:                 _tigAsset,
741:                 _daoFeesPaid,
742:                 _positionSize * _fees.burnFees / DIVISION_CONSTANT,
743:                 _referrer != address(0) ? _positionSize * _fees.referralFees / DIVISION_CONSTANT : 0,
744:                 _positionSize * _fees.botFees / DIVISION_CONSTANT,
745:                 _referrer
746:             );
747:             IStable(_tigAsset).mintFor(address(this), _daoFeesPaid);
748:         }
749:         IStable(_tigAsset).approve(address(gov), type(uint).max);   // @audit add this line of code for POC purpose
750:         gov.distribute(_tigAsset, IStable(_tigAsset).balanceOf(address(this)));
751:     }
```

Then, as a comparison, the following test can be added in the `Signature verification` `describe` block in `test\07.Trading.js`. This test will pass to demonstrate that the Governance NFT holder's pending rewards is no longer 0 after implementing the suggested mitigation. Please see the comments in this test for more details.
```typescript
    it.only(`If calling initiateMarketOrder function can correctly send DAO fees to GovNFT contract, Governance NFT holder, whose NFT was minted before initiateMarketOrder function is called,
             can receive deserved rewards after initiateMarketOrder function is called`, async function () {
      let TradeInfo = [parseEther("1000"), MockDAI.address, StableVault.address, parseEther("10"), 0, true, parseEther("30000"), parseEther("10000"), ethers.constants.HashZero];
      let PriceData = [node.address, 0, parseEther("20000"), 0, 2000000000, false];
      let message = ethers.utils.keccak256(
        ethers.utils.defaultAbiCoder.encode(
          ['address', 'uint256', 'uint256', 'uint256', 'uint256', 'bool'],
          [node.address, 0, parseEther("20000"), 0, 2000000000, false]
        )
      );
      let sig = await node.signMessage(
        Buffer.from(message.substring(2), 'hex')
      );
      
      let PermitData = [permitSig.deadline, ethers.constants.MaxUint256, permitSig.v, permitSig.r, permitSig.s, true];

      // one Governance NFT is minted to owner before initiateMarketOrder function is called
      const GovNFT = await deployments.get("GovNFT");
      const govnft = await ethers.getContractAt("GovNFT", GovNFT.address);
      await govnft.connect(owner).mint();

      // calling initiateMarketOrder function attempts to send 10000000000000000000 tigAsset as DAO fees to GovNFT contract
      await expect(trading.connect(owner).initiateMarketOrder(TradeInfo, PriceData, sig, PermitData, owner.address))
        .to.emit(trading, 'FeesDistributed')
        .withArgs(stabletoken.address, "10000000000000000000", "0", "0", "0", ethers.constants.AddressZero);

      // another Governance NFT is minted to owner and then transferred to user after initiateMarketOrder function is called
      await govnft.connect(owner).mint();
      await govnft.connect(owner).transferFrom(owner.getAddress(), user.getAddress(), 1);

      // user's pending reward amount should be 0 because her or his Governance NFT was minted after initiateMarketOrder function was called
      expect(await govnft.pending(user.getAddress(), stabletoken.address)).to.equal("0");

      // If calling initiateMarketOrder function can correctly send DAO fees to GovNFT contract, owner's pending reward amount should be 10000000000000000000
      //   because her or his Governance NFT was minted before initiateMarketOrder function was called.
      expect(await govnft.pending(owner.getAddress(), stabletoken.address)).to.equal("10000000000000000000");
    });
```

## Tools Used
VSCode

## Recommended Mitigation Steps
https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L749 can be updated to the following code.
```solidity
        IStable(_tigAsset).approve(address(gov), type(uint).max);
        gov.distribute(_tigAsset, IStable(_tigAsset).balanceOf(address(this)));
```