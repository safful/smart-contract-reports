## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- EmergencyBrake

# [EmergencyBrake.sol: Permissions cannot be re-planned after termination](https://github.com/code-423n4/2021-08-yield-findings/issues/21) 

# Handle

hickuphh3


# Vulnerability details

### Impact

Given a configuration of target, contacts and permissions, calling `terminate()` will permanently prevent this configuration from being used again because the state becomes `State.TERMINATED`. All other functions require the configuration to be in the other states (UNKNOWN, PLANNED, or EXECUTED).

In other words, the removal of the restoring option for the configuration through `EmergencyBrake` is permanent.

### Recommended Mitigation Steps

Since `EmergencyBrake` cannot reinstate permissions after termination, it would be better to have terminate change its state to UNKNOWN. The TERMINATED state can therefore be removed.

