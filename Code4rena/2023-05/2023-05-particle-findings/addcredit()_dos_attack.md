## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-02

# [addCredit() DOS Attack](https://github.com/code-423n4/2023-05-particle-findings/issues/16) 

# Lines of code

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L518


# Vulnerability details

## Impact
DOS Attack

## Proof of Concept
`addCredit()` can be called by anyone, and the `msg.value` is as small as `1 wei`.

Users can modify Lien at a small cost, causing the value stored in `liens[lienId]=keccak256(abi.encode(lien))` to change
By front-run, the normal user's transaction `validateLien()` fails the check, thus preventing the user's transaction from being executed

The following methods will be exploited: (most methods with `validateLien()` will be affected)
For example.
1. front-run `auctionBuyNft()` is used to prevent others from bidding
2.  front-run  `startLoanAuction()` to prevent the lender from starting the auction
3.  front-run  `stopLoanAuction()` is used to stop Lender from closing the auction
etc.


## Tools Used

## Recommended Mitigation Steps

1. `addCredit() ` can execute only be the borrower
2. add the modification interval period
3. limit min of msg.value



## Assessed type

Context