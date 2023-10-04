## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-04

# [First depositor can break minting of liquidity shares in GeVault](https://github.com/code-423n4/2023-08-goodentry-findings/issues/367) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L271-L278
https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L420-L424


# Vulnerability details

## Impact
In [GeVault](https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/GeVault.sol), while depositing tokens in the pool, liquidity tokens are minted to the users.

Calculation of liquidity tokens to mint uses `balanceOf(address(this))` which makes it susceptible to first deposit share price manipulation attack.

`deposit` calls `getTVL`, which calls `getTickBalance`


[GeVault.deposit#L271-L278](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L271-L278)
```
    uint vaultValueX8 = getTVL();
    uint tSupply = totalSupply();
    // initial liquidity at 1e18 token ~ $1
    if (tSupply == 0 || vaultValueX8 == 0)
      liquidity = valueX8 * 1e10;
    else {
      liquidity = tSupply * valueX8 / vaultValueX8;
    }
```
[GeVault.getTVL#L392-L398](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L392-L398)
```
  function getTVL() public view returns (uint valueX8){
    for(uint k=0; k<ticks.length; k++){
      TokenisableRange t = ticks[k];
      uint bal = getTickBalance(k);
      valueX8 += bal * t.latestAnswer() / 1e18;
    }
  }
```
[GeVault.getTickBalance#L420-L424](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L420-L424)
```
  function getTickBalance(uint index) public view returns (uint liquidity) {
    TokenisableRange t = ticks[index];
    address aTokenAddress = lendingPool.getReserveData(address(t)).aTokenAddress;
    liquidity = ERC20(aTokenAddress).balanceOf(address(this));
  }
```

Although there is a condition on line [`281`](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L281) that liquidity to be minted must be greater than 0, User's funds can be at risk.
## Proof of Concept
When totalSupply is zero, an attacker can go ahead and execute following steps.

1. Calls [deposit](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L247) function with 1 wei amount of underlying as argument. To that, he will be minted some amount of liquidity share depending on the price of underlying.
2. [Withdraw](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L214) all except one wei of shares.
3. Transfer some X amount of underlying directly to pool contract address.
So, now 1 wei of share worths X underlying tokens.
Attacker won't have any problem making this X as big as possible. Because he'll always be able to redeem it with 1 wei of share.

**Impact**
1. Almost 1/4th of first deposit can be frontrun and stolen.
- Let's assume there is a first user trying to deposit with z dollars worth of tokens
- An attacker can see this transaction in mempool and carry out the above-described attack with x = (z/2 + 1).
- This means the user gets 1 Wei of share which is only worth ~ 3x/4 of tokens.
- Here, the percentage of the user funds lost depends on how much capital the attacker has. let's say a attacker keeps 2 wei in the share initially instead of 1 (this makes doubles capital requirement), they can get away with 33% of the user's funds.
2. DOS to users who tries to deposit less than X because of this [check](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L281)
```
    require(liquidity > 0, "GEV: No Liquidity Added");
```
## Tools Used
Manual Review
## Recommended Mitigation Steps
- Burn some MINIMUM_LIQUIDITY during first deposit


## Assessed type

Math