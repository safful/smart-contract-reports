## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- M-12

# [yIntercept of public vaults can overflow](https://github.com/code-423n4/2023-01-astaria-findings/issues/388) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L582-L584


# Vulnerability details

## Impact
The yIntercept of a public vault can overflow due to an unchecked addition. As a result, totalAsset will be a lot lower than the actual amount, which prevents liquidity providers from withdrawing a large fraction of their assets.

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L323-L325

The amount of assets required for this to happen is barely feasible for a regular 18 decimal ERC20 token, but can happen with ease for tokens with higher precision

## Proof of Concept

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

  function testYinterceptOverflow() public {
    
    address publicVault = _createPublicVault({
      strategist: strategistOne,
      delegate: strategistTwo,
      epochLength: 14 days
    });
    _lendToVault(
      Lender({addr: address(1), amountToLend: 309000000 ether}),
      publicVault
    );
    _lendToVault(
      Lender({addr: address(2), amountToLend: 10000000 ether}),
      publicVault
    );
    assertTrue(PublicVault(publicVault).totalAssets() < 10000000 ether);
    
  }
}
```


## Tools Used
VSCode, Foundry

## Recommended Mitigation Steps
Remove the unchecked block.