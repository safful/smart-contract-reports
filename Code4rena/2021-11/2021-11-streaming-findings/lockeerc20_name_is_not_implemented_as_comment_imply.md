## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [LockeERC20 name is not implemented as comment imply](https://github.com/code-423n4/2021-11-streaming-findings/issues/110) 

# Handle

wuwe1


# Vulnerability details

```solidity
// locke + depositTokenName + streamId = lockeUSD Coin-1
name = string(abi.encodePacked("locke", ERC20(depositToken).name(), ": ", toString(streamId)));
```

As the comment imply, the `": "` should be `"-"`

## Recommended Mitigation Steps

Consider change the comment or the code.

