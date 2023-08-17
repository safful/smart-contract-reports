## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-11

# [Liquidator reward is not taken into account when calculating potential debt](https://github.com/code-423n4/2023-01-astaria-findings/issues/400) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L471-L489


# Vulnerability details

## Impact
Liquidator reward is not taken into account when calculating potential debt. When liquidationInitialAsk is set to the bare minimum, liquidator reward will always come at the expense of vaults late on in the stack.

## Proof of Concept
Consider the following test where the vault loses more than 3 tokens.
```
pragma solidity =0.8.17;

import "forge-std/Test.sol";

import {Authority} from "solmate/auth/Auth.sol";
import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";
import {MockERC721} from "solmate/test/utils/mocks/MockERC721.sol";
import {
  MultiRolesAuthority
} from "solmate/auth/authorities/MultiRolesAuthority.sol";

import {ERC721} from "gpl/ERC721.sol";
import {SafeCastLib} from "gpl/utils/SafeCastLib.sol";

import {IAstariaRouter, AstariaRouter} from "../AstariaRouter.sol";
import {VaultImplementation} from "../VaultImplementation.sol";
import {PublicVault} from "../PublicVault.sol";
import {TransferProxy} from "../TransferProxy.sol";
import {WithdrawProxy} from "../WithdrawProxy.sol";
import {Vault} from "../Vault.sol";

import {Strings2} from "./utils/Strings2.sol";

import "./TestHelpers.t.sol";
import {OrderParameters} from "seaport/lib/ConsiderationStructs.sol";

contract AstariaTest is TestHelpers {
  using FixedPointMathLib for uint256;
  using CollateralLookup for address;
  using SafeCastLib for uint256;

  event NonceUpdated(uint256 nonce);
  event VaultShutdown();

  function testProfitFromLiquidatorFee() public {

    TestNFT nft = new TestNFT(1);
    address tokenContract = address(nft);
    uint256 tokenId = uint256(0);

    address publicVault = _createPublicVault({
      strategist: strategistOne,
      delegate: strategistTwo,
      epochLength: 7 days
    });
    _lendToVault(
      Lender({addr: address(1), amountToLend: 60 ether}),
      publicVault
    );

    (, ILienToken.Stack[] memory stack) = _commitToLien({
      vault: publicVault,
      strategist: strategistOne,
      strategistPK: strategistOnePK,
      tokenContract: tokenContract,
      tokenId: tokenId,
      lienDetails: ILienToken.Details({
        maxAmount: 50 ether,
        rate: (uint256(1e16) * 150) / (365 days),
        duration: 10 days,
        maxPotentialDebt: 0 ether,
        liquidationInitialAsk: 42 ether
      }),
      amount: 40 ether,
      isFirstLien: true
    });

    vm.warp(block.timestamp + 10 days);
    OrderParameters memory listedOrder = ASTARIA_ROUTER.liquidate(
      stack,
      uint8(0)
    );

    _bid(Bidder(bidder, bidderPK), listedOrder, 42 ether);
    assertTrue(WETH9.balanceOf(publicVault) < 57 ether);
  }
}
```


## Tools Used
VSCode, Foundry

## Recommended Mitigation Steps
Include liquidator reward in the calculation of potential debt.