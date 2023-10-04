## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- MR-H-05

# [H-05 Unmitigated](https://github.com/code-423n4/2023-08-pooltogether-mitigation-findings/issues/31) 

# Lines of code




# Vulnerability details

### Issue not mitigated

### About the problem
`sponsor` function allows caller to delegate his shares to the special address. In this case caller losses ability to win prizes. Previous version of code had `sponsor` function, which allowed to deposit funds on behalf of owner and delegate all user's funds to the `SPONSORSHIP_ADDRESS`. As result, attacker had ability to deposit small amount of funds(or 0) on behalf of user and make him delegate to the `SPONSORSHIP_ADDRESS`.
### Solution
Pool together team has fixed that issue by removing [this function](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol#L480-L482). So now attacker can't directly call `sponsor` for any address. But there is another function: `sponsorWithPermit`. This function allows anyone to provide valid permit for asset and then do sponsoring.
It's possible that user will create permit for other purposes(to just `depositWithPermit` function). In this case, attacker(which can be some other web portal) can reuse this permit and make user delegate all balance to the `SPONSORSHIP_ADDRESS` instead of just depositing.

This issues stands, because currently there user can't be sure, how his signed message(permit) will be used. User can sign permit to deposit, by it will be used to deposit and delegate to the sponsorship address.