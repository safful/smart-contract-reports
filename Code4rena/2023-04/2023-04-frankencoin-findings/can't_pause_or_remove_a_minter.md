## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-13

# [Can't pause or remove a minter](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/230) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L152


# Vulnerability details

## Impact
The project is supposed to be self-governing. Token owners are able to suggest new minters and collateral. The auction mechanism allows participants to remove collateral from the system if it's deemed unhealthy. But, there's no way to remove a registered minter.

Why would you want to remove a registered minter? Because they can have bugs that could break the whole system. The current minter, MintingHub, for example, implements a novel auction mechanism to price collateral instead of choosing the industry standard Chainlink oracles. While I support the idea of having a fully decentralized system, it does add additional risk to the project. A risk that's taken not only by the protocol team but every ZCHF holder as well.

Obviously, these are just hypotheticals. But, the system is not equipped to handle a scenario where the minter malfunctions.

## Proof of Concept
After a minter is suggested you have generally 10 days to deny it. After that, there's no way to remove it:

```sol
   function denyMinter(address _minter, address[] calldata _helpers, string calldata _message) override external {
      if (block.timestamp > minters[_minter]) revert TooLate();
      reserve.checkQualified(msg.sender, _helpers);
      delete minters[_minter];
      emit MinterDenied(_minter, _message);
   }

```

## Tools Used
none

## Recommended Mitigation Steps
Implement the ability for token holders to temporarily pause a minter as well as remove it altogether.