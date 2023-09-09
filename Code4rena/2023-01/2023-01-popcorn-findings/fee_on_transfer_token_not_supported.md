## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-14

# [Fee on transfer token not supported](https://github.com/code-423n4/2023-01-popcorn-findings/issues/503) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn//blob/main/src/utils/MultiRewardEscrow.sol#L100


# Vulnerability details

## Impact
If you are making a Lock fund for escrow using a fee on transfer token then contract will receive less amount (X-fees) but will record full amount (X). This becomes a problem as when claim is made then call will fail due to lack of funds. Worse, one user will unknowingly take the missing fees part from another user deposited escrow fund

## Proof of Concept
1. User locks token X as escrow which take fee on transfer
2. For same, he uses `lock` function which transfer funds from user to contract

```
 function lock(
    IERC20 token,
    address account,
    uint256 amount,
    uint32 duration,
    uint32 offset
  ) external {
...
 token.safeTransferFrom(msg.sender, address(this), amount);
...
escrows[id] = Escrow({
      token: token,
      start: start,
      end: start + duration,
      lastUpdateTime: start,
      initialBalance: amount,
      balance: amount,
      account: account
    });
...
}
```

3. Since token has fee on transfer so contract receives only `amount-fees` but the escrow object is created for full `amount`

4. Lets say escrow duration is over and claim is made using `claimRewards` function

```
function claimRewards(bytes32[] memory escrowIds) external {
...
 uint256 claimable = _getClaimableAmount(escrow);
...    
 escrow.token.safeTransfer(escrow.account, claimable);
...
}
```

5. Since full duration is over so claimable amount is `amount`. But this fails on transfer to account since contract has only `amount-fees`

## Recommended Mitigation Steps
Compute the balance before and after transfer and subtract them to get the real amount. Also use nonReentrant while using this to prevent from reentrancy in ERC777 tokens