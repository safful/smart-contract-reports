## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- sponsor acknowledged
- selected for report

# [If L1GraphTokenGateway's outboundTransfer is called by a contract, the entire msg.value is blackholed, whether the ticket got redeemed or not.](https://github.com/code-423n4/2022-10-thegraph-findings/issues/294) 

# Lines of code

https://github.com/code-423n4/2022-10-thegraph/blob/309a188f7215fa42c745b136357702400f91b4ff/contracts/gateway/L1GraphTokenGateway.sol#L236


# Vulnerability details

The outboundTransfer function in L1GraphTokenGateway is used to transfer user's Graph tokens to L2. To do that it eventually calls the standard Arbitrum Inbox's createRetryableTicket. The issue is that it passes caller's address in the `submissionRefundAddress` and `valueRefundAddress`. This behaves fine if caller is an EOA, but if it's called by a contract it will lead to loss of the submissionRefund (ETH passed to outboundTransfer() minus the total submission fee), or in the event of failed L2 ticket creation, the whole submission fee. The reason it's fine for EOA is because of the fact that ETH and Arbitrum addresses are congruent. However, the calling contract probably does not exist on L2 and even in the rare case it does, it might not have a function to move out the refund.

The docs don't suggest contracts should not use the TokenGateway, and it is fair to assume it will be used in this way. Multisigs are inherently contracts, which is one of the valid use cases. Since likelihood is high and impact is medium (loss of submission fee), I believe it to be a HIGH severity find.

## Impact

If L1GraphTokenGateway's outboundTransfer is called by a contract, the entire msg.value is blackholed, whether the ticket got redeemed or not.

## Proof of Concept

Alice has a multisig wallet. She sends 100 Graph tokens to L1GraphTokenGateway, and passes X ETH for submission. She receives an L1 ticket, but since the max gas was too low, the creation failed on L2 and the funds got sent to the multisig address at L2. Therefore, Alice loses X ETH.

## Tools Used

https://github.com/OffchainLabs/arbitrum/blob/master/docs/L1_L2_Messages.md
Manual audit

## Recommended Mitigation Steps

A possible fix is to add an `isContract` flag. If sender is a contract, require the flag to be true.

Another option is to add a `refundAddr` address parameter to the API.