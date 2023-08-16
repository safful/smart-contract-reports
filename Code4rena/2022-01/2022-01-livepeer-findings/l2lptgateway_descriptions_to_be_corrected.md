## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [L2LPTGateway descriptions to be corrected](https://github.com/code-423n4/2022-01-livepeer-findings/issues/157) 

# Handle

hyh


# Vulnerability details

## Proof of Concept

In L2LPTGateway contract description the @title is L1LPTGateway

In L2LPTGateway.outboundTransfer function's description there is '@param _data Contains sender and additional data to send to L1' line, while actually function allows no additional data

