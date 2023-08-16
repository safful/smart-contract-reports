## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Missing emit of initial `ApplyOwnership` event in `GaugeController.__init__()`](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/79) 

# Handle

pants


# Vulnerability details

The function `GaugeController.__init__()` (the constructor) sets the initial admin, but it doesn't emit an appropriate `ApplyOwnership` event.

## Impact
The users won't know who's the initial admin by searching for the first `ApplyOwnership` event, although they should be able to.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit this event.

