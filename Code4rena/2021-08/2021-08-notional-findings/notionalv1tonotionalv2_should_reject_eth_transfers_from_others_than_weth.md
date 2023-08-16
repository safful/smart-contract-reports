## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [NotionalV1ToNotionalV2 should reject ETH transfers from others than WETH](https://github.com/code-423n4/2021-08-notional-findings/issues/48) 

# Handle

pauliax


# Vulnerability details

## Impact
contract NotionalV1ToNotionalV2 has an empty receive function which allows it to receive Ether. I suppose this was needed to receive ETH when withdrawing from WETH. As there is no way to send out accidentally sent ETH from this contract, I suggest adding an auth check to this receive function to only accept ETH from WETH contract.

## Recommended Mitigation Steps
require(msg.sender == address(WETH), "Not WETH");

