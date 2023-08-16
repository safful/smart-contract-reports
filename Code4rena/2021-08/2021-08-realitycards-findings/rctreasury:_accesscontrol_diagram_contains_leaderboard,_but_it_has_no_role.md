## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- Resolved

# [RCTreasury: AccessControl diagram contains Leaderboard, but it has no role](https://github.com/code-423n4/2021-08-realitycards-findings/issues/44) 

# Handle

hickuphh3


# Vulnerability details

### Impact

```jsx
/* setup AccessControl

                 UBER_OWNER
    ┌───────────┬────┴─────┬────────────┬─────────┐
    │           │          │            │         │
  OWNER      FACTORY    ORDERBOOK   TREASURY  LEADERBOARD
    │           │
 GOVERNOR     MARKET
    │
 WHITELIST | ARTIST | AFFILIATE | CARD_AFFILIATE
*/
```

From this diagram, one might expect the existence of a `LEADERBOARD` role, but there is no such role. It should be removed from the diagram.

### Recommended Mitigation Steps

```jsx
/* setup AccessControl

                 UBER_OWNER
    ┌───────────┬────┴─────┬────────────┐
    │           │          │            │         
  OWNER      FACTORY    ORDERBOOK   TREASURY
    │           │
 GOVERNOR     MARKET
    │
 WHITELIST | ARTIST | AFFILIATE | CARD_AFFILIATE
*/
```

