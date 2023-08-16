## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [Misleading comment in `LaunchEvent.getReserves`](https://github.com/code-423n4/2022-01-trader-joe-findings/issues/209) 

# Handle

cmichel


# Vulnerability details

`LaunchEvent.getReserves`: The comment says: `@notice Returns the current balance of the pool`. The "of the pool" part can be misleading as the `tokenIncentivesBalance` are never part of the _pool pair_. Consider changing this to "Returns the outstanding balance of the launch event contract".

