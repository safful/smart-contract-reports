## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- Resolved

# [RCLeaderboard: Erroneous comment](https://github.com/code-423n4/2021-08-realitycards-findings/issues/43) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The comments above the event declarations were probably copied over from RCOrderbook. They should be modified to refer to the leaderboard.

### Recommended Mitigation Steps

```jsx
/// @dev emitted every time a user is added to the leaderboard
event LogAddToLeaderboard(address _user, address _market, uint256 _card);
/// @dev emitted every time a user is removed from the leaderboard
event LogRemoveFromLeaderboard(
    address _user,
    address _market,
    uint256 _card
);
```

