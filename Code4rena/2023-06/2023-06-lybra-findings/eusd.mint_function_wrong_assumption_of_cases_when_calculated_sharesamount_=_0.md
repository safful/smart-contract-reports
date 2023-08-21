## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-06

# [EUSD.mint function wrong assumption of cases when calculated sharesAmount = 0](https://github.com/code-423n4/2023-06-lybra-findings/issues/106) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/token/EUSD.sol#L299-#L306
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/token/EUSD.sol#L414-#L418


# Vulnerability details

## Impact
- Mint function might calculate the sharesAmount incorrectly.
- User can profit by manipulating the protocol to enjoy 1-1 share-eUSD ratio even when share prices
 is super high.

## Proof of Concept
Currently, the function `EUSD.mint` calls function `EUSD.getSharesByMintedEUSD` to calculate the shares corresponding to the input eUSD amount:
```solidity
function mint(address _recipient, uint256 _mintAmount) external onlyMintVault MintPaused returns (uint256 newTotalShares) {
        require(_recipient != address(0), "MINT_TO_THE_ZERO_ADDRESS");

        uint256 sharesAmount = getSharesByMintedEUSD(_mintAmount);
        if (sharesAmount == 0) {
            //EUSD totalSupply is 0: assume that shares correspond to EUSD 1-to-1
            sharesAmount = _mintAmount;
        }
        ...
}
function getSharesByMintedEUSD(uint256 _EUSDAmount) public view returns (uint256) {
        uint256 totalMintedEUSD = _totalSupply;
        if (totalMintedEUSD == 0) {
            return 0;
        } else {
            return _EUSDAmount.mul(_totalShares).div(totalMintedEUSD);
        }
}
```
As you can see in the comment after `sharesAmount` is checked, `//EUSD totalSupply is 0: assume that shares correspond to EUSD 1-to-1`. The code assumes that if `sharesAmount = 0`, then `totalSupply` must be 0 and the minted share should equal to input eUSD. However, that's not always the case.

Variable `sharesAmount` could be 0 if `totalShares *_EUSDAmount` < `totalMintedEUSD` because this is integer division. If that happens, the user will profit by calling mint with a small EUSD amount and enjoys 1-1 minting proportion (1 share for each eUSD). The reason this can happen is because `EUSD` support `burnShares` feature, which remove the share of a users but keep the `totalSupply` value.

For example:
1. At the start, Bob is minted 1e18 eUSD, he receives 1e18 shares
2. Bob call `burnShares` by `1e18-1`, after this, contract contains 
    1e18 eUSD and 1 share, which mean 1 share now worth 1e18 eUSD
3. If Alice calls `mint` with 1e18 eUSD, then she receives 1 share (since 1 share worth
   1e18 eUSD)
4. However, if she then calls `mint` with 1e17 eUSD, she will receives 1e17 shares although
    1 share now worth 1e18 eUSD. This happens because `1e17 * (totalShares = 2) / (totalMintedEUSD = 2e18)` = 0

Below is POC for the above example, I use foundry to run tests, create
a folder named `test` and save this to a file named `eUSD.t.sol`, then run it using command:

`forge test --match-path test/eUSD.t.sol -vvvv`

```solidity
pragma solidity ^0.8.17;

import {Test, console2} from "forge-std/Test.sol";
import {Iconfigurator} from "contracts/lybra/interfaces/Iconfigurator.sol";
import {Configurator} from "contracts/lybra/configuration/LybraConfigurator.sol";
import {GovernanceTimelock} from "contracts/lybra/governance/GovernanceTimeLock.sol";
import {mockCurve} from "contracts/mocks/mockCurve.sol";
import {EUSD} from "contracts/lybra/token/EUSD.sol";

contract TestEUSD is Test {
    address admin = address(0x1111);
    address user1 = address(0x1);
    address user2 = address(0x2);
    address pool = address(0x3);

    Configurator configurator;
    GovernanceTimelock governanceTimeLock;
    mockCurve curve;
    EUSD eUSD;



    function setUp() public{
        // deploy curve
        curve = new mockCurve();
        // deploy governance time lock
        address[] memory proposers = new address[](1);
        proposers[0] = admin;

        address[] memory executors = new address[](1);
        executors[0] = admin;

        governanceTimeLock = new GovernanceTimelock(1, proposers, executors, admin);
        configurator = new Configurator(address(governanceTimeLock), address(curve));

        eUSD = new EUSD(address(configurator));
        // set mintVault to this address
        vm.prank(admin);
        configurator.setMintVault(address(this), true);
    }

    function testRoundingNotCheck() public {
        // Mint some tokens for user1
        eUSD.mint(user1, 1e18);

        assertEq(eUSD.balanceOf(user1), 1e18);
        assertEq(eUSD.totalSupply(), 1e18);

        //
        eUSD.burnShares(user1, 1e18-1);

        assertEq(eUSD.getTotalShares(),1);
        assertEq(eUSD.sharesOf(user1), 1);
        assertEq(eUSD.totalSupply(), 1e18);

        // After this, 1 shares worth 1e18 eUSDs
        // If mintAmount = 1e18 -> receive  1 shares

        eUSD.mint(user2, 1e18);
        assertEq(eUSD.getTotalShares(), 2);
        assertEq(eUSD.sharesOf(user2), 1);
        assertEq(eUSD.totalSupply(), 2e18);

        // However, if mintAmount = 1e17 -> receive 1e17 shares

        eUSD.mint(user2, 1e17);

        assertEq(eUSD.sharesOf(user2), 1 + 1e17);


    }

}
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
I recommend checking again in `EUSD.mint` function if `sharesAmount` is 0
and `totalSupply` is not 0, then exit the function without minting anything.











## Assessed type

Math