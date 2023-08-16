## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: `RocketJoeStaking.withdraw`](https://github.com/code-423n4/2022-01-trader-joe-findings/issues/210) 

# Handle

cmichel


# Vulnerability details

`RocketJoeStaking.withdraw`: The `_safeRJoeTransfer(msg.sender, pending)` only needs to be performed if `pending > 0`.

