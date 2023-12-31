## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [Migration::withdrawContribution falsely assumes that user should get exactly his original contribution back](https://github.com/code-423n4/2022-07-fractional-findings/issues/375) 

# Lines of code

https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Migration.sol#L308
https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Migration.sol#L321
https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Migration.sol#L312
https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Migration.sol#L325


# Vulnerability details

When a user calls `withdrawContribution`, it will try to send him back his original contribution for the proposal.
But if the proposal has been committed, and other users have interacted with the buyout,
Migration will receive back a different amount of ETH and tokens.
Therefore it shouldn't send the user back his original contribution, but should send whatever his share is of whatever was received back from Buyout.

## Impact
Loss of funds for users.
Some users might not be able to withdraw their contribution at all,
and other users might withdraw funds that belong to other users. (This can also be done as a purposeful attack.)

## Proof of Concept
A summary is described at the top.

It's probably not needed, but the here's the flow in detail.
When a user joins a proposal, Migration [saves](https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Migration.sol#L124:#L135) his contribution:
```
        userProposalEth[_proposalId][msg.sender] += msg.value;
        userProposalFractions[_proposalId][msg.sender] += _amount;
```
Later when the user would want to withdraw his contribution from a failed migration, Migration would [refer](https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Migration.sol#L308:#L325) to these same variables to decide how much to send to the user:
```
        uint256 userFractions = userProposalFractions[_proposalId][msg.sender];
        IFERC1155(token).safeTransferFrom(address(this), msg.sender, id, userFractions, "");
        uint256 userEth = userProposalEth[_proposalId][msg.sender];
        payable(msg.sender).transfer(userEth);
```

But if the proposal was committed, and other users interacted with the buyout, then the amount of ETH and tokens that Buyout sends back is not the same contribution.
For example, if another user called `buyFractions` for the buyout, it [will decrease](https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Buyout.sol#L168) the amount of tokens in the pool:
```
        IERC1155(token).safeTransferFrom(address(this), msg.sender, id, _amount, "");
```
And when the proposal will end, if it has failed, Buyout will [send back](https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Buyout.sol#L228) to Migration [the amount](https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Buyout.sol#L206) of tokens in the pool:
```
        uint256 tokenBalance = IERC1155(token).balanceOf(address(this), id);
        ...
        IERC1155(token).safeTransferFrom(address(this), proposer, id, tokenBalance, "");
```
(**Same will happen for the ETH amount)

Therefore, Migration will receive back less tokens than the original contribution was.
When the user will try to call `withdrawContribution` to withdraw his contribution from the pool, Migration would [try to send](https://github.com/code-423n4/2022-07-fractional/blob/main/src/modules/Migration.sol#L310) the user's original contribution.
But there's a deficit of that.
If other users have contributed the same token, then it will transfer their tokens to the user.
If not, then the withdrawal will simply revert for insufficient balance.

## Recommended Mitigation Steps
I am not sure, but I think that the correct solution would be that upon a failed proposal's end, there should be a hook call from Buyout to the proposer - in our situation, Migration.
Migration would then see(/receive as parameter) how much ETH/tokens were received, and update the proposal with the change needed. eg. send to each user 0.5 his tokens and 1.5 his ETH.
In another issue I submitted, "User can't withdraw assets from failed migration if another buyout is going on/succeeded", I described for a different reason why such a callback to Migration might be needed. Please see there for more implementation suggestion.
I think this issue shows that indeed it is needed.

