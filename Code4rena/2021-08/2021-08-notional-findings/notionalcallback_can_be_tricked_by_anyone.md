## Tags

- bug
- duplicate
- 3 (High Risk)
- sponsor confirmed

# [notionalCallback can be tricked by anyone](https://github.com/code-423n4/2021-08-notional-findings/issues/45) 

# Handle

pauliax


# Vulnerability details

## Impact
Anyone can call function notionalCallback with arbitrary params and pass the auth check. The only auth check can be easily bypassed by setting sender param to the address of this contract. It allows to choose any parameter that I want:
    function notionalCallback(
        address sender,
        address account,
        bytes calldata callbackData
    ) external returns (uint256) {
        require(sender == address(this), "Unauthorized callback");

## Recommended Mitigation Steps
It needs to check that msg.sender is Notional.

