## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Missing emit of initial `SetAdmin` event in `MainToken.__init__()`](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/78) 

# Handle

pants


# Vulnerability details

The function `MainToken.__init__()` (the constructor) sets the initial admin, but it doesn't emit an appropriate `SetAdmin` event.

## Impact
The users won't know who's the initial admin by searching for the first `SetAdmin` event, although they should be able to.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit this event.

