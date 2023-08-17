## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-05

# [Position owners can deny liquidations](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/481) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Position.sol#L159
https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Position.sol#L307


# Vulnerability details

## Impact
The owner of a vulnerable position can deny being liquidated by setting the price to be `type(uint256).max`, making every call to `tryAvertChallenge` fail due to an overflow.

This means that if it's advantageous enough the owner can choose to keep `zchf` and leave the collateral stuck. This could happen in any scenario where a collateral is likely to loose it's value, for example, de-pegs, runs on the bank, etc.

## Test Proof
Here's a snippet that can be pasted on `GeneralTest.t.sol`:
```solidity
    function test_liquidationDenial() public {
        test01Equity(); // ensure there is some equity to burn
        address posAddress = initPosition();
        Position pos = Position(posAddress);

        skip(15 * 86_400 + 60);

        alice.mint(address(pos), 1001);

        vm.prank(address(alice));
        pos.adjustPrice(type(uint256).max);

        col.mint(address(bob), 1001);
        uint256 first = bob.challenge(hub, posAddress, 1001);

        bob.obtainFrankencoins(swap, 55_000 ether);

        vm.expectRevert();
        bob.bid(hub, first, 10_000 ether); 

        skip(7 * 86_400 + 60);

        vm.expectRevert();
        hub.end(first, false);
    }
```
