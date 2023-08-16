## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Don't ask for the user's signature when msg.sender == txData.user](https://github.com/code-423n4/2021-07-connext-findings/issues/57) 

# Handle

pauliax


# Vulnerability details

## Impact
I think it would make sense not to check the user's signature in recoverCancelSignature or recoverFulfillSignature if the caller is the user himself. 

## Recommended Mitigation Steps
Replace:
require(recoverCancelSignature(txData, relayerFee, signature) == txData.user, "cancel: INVALID_SIGNATURE");
require(recoverFulfillSignature(txData, relayerFee, signature) == txData.user, "fulfill: INVALID_SIGNATURE");
with:
require(msg.sender == txData.user || recoverCancelSignature(txData, relayerFee, signature) == txData.user, "cancel: INVALID_SIGNATURE");
require(msg.sender == txData.user || recoverFulfillSignature(txData, relayerFee, signature) == txData.user, "fulfill: INVALID_SIGNATURE");

