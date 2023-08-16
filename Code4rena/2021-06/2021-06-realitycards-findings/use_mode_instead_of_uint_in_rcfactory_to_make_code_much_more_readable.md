## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [Use Mode instead of uint in RCFactory to make code much more readable](https://github.com/code-423n4/2021-06-realitycards-findings/issues/162) 

# Handle

a_delamo


# Vulnerability details

On `RCFactory` is using uint to represent the enum Mode while on RCMarket is using the enum directly. 
It would make the code much readable if RCFactory would use Mode directly.


