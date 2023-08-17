## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-08

# [If the strategy incurs a loss the Active Pool will stop working until the shortfall is paid out entirely](https://github.com/code-423n4/2023-02-ethos-findings/issues/632) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L251-L252


# Vulnerability details

- Vaults are built with the idea that a [loss could happen](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L385-L386)

- The scope mentions that a [Loss scenario is in scope](https://docs.reaper.farm/ethos-reserve-bounty-hunter-documentation/#community-usdoath-issuance:~:text=stays%20fully%20capitalized.-,The%20vault%20will%20be%20farming%20the%20lowest%2Drisk%20yield%20possible%2C%20so%20you%20can%20assume%20that%20the%20principal%20will%20be%20protected%20from%20loss.%20We%20encourage%20you%20nonetheless%20to%20analyze%20the%20impact%20of%20losses%20on%20the%20system.,-Balancer%20Pool%20Staking)

This line, is written with the assumption that `sharesToAssets` will always be greater than or equal to `currentAllocated`

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L251-L252

```solidity
        vars.profit = vars.sharesToAssets.sub(vars.currentAllocated);

```

This is not the case as the Strategy MAY incur a loss.

In such cases, `_rebalance` on the ActivePool will not work until the subtraction stops underflowing `vars.sharesToAssets.sub(vars.currentAllocated);` will revert if any loss (even 1 wei) has happened.


### POC

When a loss happens, the `sharesToAssets` will decrease.

Because [`vars.currentAllocated`](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L243) tracks the amount deposited in the vault, this value will necessarily be greater than the `sharesToAsset` if any loss happened.


In that case this line will revert:
        ```vars.profit = vars.sharesToAssets.sub(vars.currentAllocated);```

In the code shown, a loss could happen if the LendingPool has accounting errors

For the in-scope codebase a loss could happen as a consequence of slashing or restructuring due to bad debt incurred by borrowers

### Coded POC

The following POC was built with brownie

Mocked contract retain the core logic, but are rid of access control and other functions to keep the logic the same but reduce complexity of setup

Setup brownie via `brownie console` (local environment is fine as I set-up mocks to make it easy)

```python
## Setup tokens
token = MockToken.deploy({"from": a[0]})

## Deploy Vault
vault = ReaperVaultV2.deploy(token, {"from": a[0]})

## Deploy ActivePool
pool = MockActivePool.deploy(token, 2000, vault, 1, {"from": a[0]})

## Add to Active
token.approve(pool, 1e18, {"from": a[0]})
pool.depositColl(1e18, {"from": a[0]})

## Rebalance to invest
pool.manualRebalance(token, 0, {"from": a[0]})

## 20% of tokens are in the vault
print(token.balanceOf(vault))
200000000000000000

## Trigger loss to vault
vault.triggerLoss(1e17, {"from": a[0]})
## Confirm the loss has happened
print(vault.balance())
100000000000000000

## Now that a loss happened, any rebalance will revert
pool.manualRebalance(token, 1, {"from": a[0]})
Transaction sent: 0x798e759783ab59dda9c294178859fec5519179a2c31b89abbfea56bd7284b0bc
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 8
  MockActivePool.manualRebalance confirmed (Integer overflow)   Block: 9   Gas used: 32821 (0.27%)

<Transaction '0x798e759783ab59dda9c294178859fec5519179a2c31b89abbfea56bd7284b0bc'>
pool.manualRebalance(token, 100, {"from": a[0]})
Transaction sent: 0x0c41ec1b74a05df6c7101522931cda6ba30139358ec239f014777d7e7e992563
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 9
  MockActivePool.manualRebalance confirmed (Integer overflow)   Block: 10   Gas used: 32821 (0.27%)

<Transaction '0x0c41ec1b74a05df6c7101522931cda6ba30139358ec239f014777d7e7e992563'>

## That's because the loss has been registered by the Vault
print(vault.convertToAssets(1e17))
50000000000000000

## But not by the Pool, triggering a revert at this line
> vars.profit = vars.sharesToAssets.sub(vars.currentAllocated);
```


#### Mocks Used

#### ActivePool.sol

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.0;

import {ReaperVaultV2} from "./ReaperVaultV2.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";


contract MockActivePool {
    using SafeMath for uint256;


    address immutable collateral;

    mapping(address => uint256) public collAmount;
    mapping(address => uint256) public yieldingPercentage; // collateral => % to use for yield farming (in BPS, <= 10k)
    mapping(address => uint256) public yieldingAmount; // collateral => actual wei amount being used for yield farming
    mapping(address => address) public yieldGenerator; // collateral => corresponding ERC4626 vault
    mapping(address => uint256) public yieldClaimThreshold; // collateral => minimum wei amount of yield to claim and redistribute

    uint256 public yieldingPercentageDrift = 100; // rebalance iff % is off by more than 100 BPS

    // Yield distribution params, must add up to 10k
    uint256 public yieldSplitTreasury = 20_00; // amount of yield to direct to treasury in BPS
    uint256 public yieldSplitSP = 40_00; // amount of yield to direct to stability pool in BPS
    uint256 public yieldSplitStaking = 40_00; // amount of yield to direct to OATH Stakers in BPS

    // Mock addresses, unused
    address public treasuryAddress = address(1);
    address public stabilityPoolAddress = address(2);
    address public lqtyStakingAddress = address(3);

    constructor(
        address _collateral,
        uint256 _yieldingPercentage,
        address _yieldGenerator,
        uint256 _yieldClaimThreshold
    ) {
        collateral = _collateral;
        yieldingPercentage[_collateral] = _yieldingPercentage;
        yieldGenerator[_collateral] = _yieldGenerator;
        yieldClaimThreshold[_collateral] = _yieldClaimThreshold;
    }

    function depositColl(uint256 amount) external {
      ERC20(collateral).transferFrom(msg.sender, address(this), amount);
      collAmount[collateral] += amount;
    }

    function manualRebalance(address _collateral, uint256 _simulatedAmountLeavingPool) external {
        _rebalance(_collateral, _simulatedAmountLeavingPool);
    }

    struct LocalVariables_rebalance {
        uint256 currentAllocated;
        ReaperVaultV2 yieldGenerator;
        uint256 ownedShares;
        uint256 sharesToAssets;
        uint256 profit;
        uint256 finalBalance;
        uint256 percentOfFinalBal;
        uint256 yieldingPercentage;
        uint256 toDeposit;
        uint256 toWithdraw;
        uint256 yieldingAmount;
        uint256 finalYieldingAmount;
        int256 netAssetMovement;
        uint256 treasurySplit;
        uint256 stakingSplit;
        uint256 stabilityPoolSplit;
    }

    function _rebalance(address _collateral, uint256 _amountLeavingPool) internal {
        LocalVariables_rebalance memory vars;

        // how much has been allocated as per our internal records?
        vars.currentAllocated = yieldingAmount[_collateral];

        // what is the present value of our shares?
        vars.yieldGenerator = ReaperVaultV2(yieldGenerator[_collateral]);
        vars.ownedShares = vars.yieldGenerator.balanceOf(address(this));
        vars.sharesToAssets = vars.yieldGenerator.convertToAssets(vars.ownedShares);

        // if we have profit that's more than the threshold, record it for withdrawal and redistribution
        vars.profit = vars.sharesToAssets.sub(vars.currentAllocated);
        if (vars.profit < yieldClaimThreshold[_collateral]) {
            vars.profit = 0;
        }

        // what % of the final pool balance would the current allocation be?
        vars.finalBalance = collAmount[_collateral].sub(_amountLeavingPool);
        vars.percentOfFinalBal =
            vars.finalBalance == 0 ? type(uint256).max : vars.currentAllocated.mul(10_000).div(vars.finalBalance);

        // if abs(percentOfFinalBal - yieldingPercentage) > drift, we will need to deposit more or withdraw some
        vars.yieldingPercentage = yieldingPercentage[_collateral];
        vars.finalYieldingAmount = vars.finalBalance.mul(vars.yieldingPercentage).div(10_000);
        vars.yieldingAmount = yieldingAmount[_collateral];
        if (
            vars.percentOfFinalBal > vars.yieldingPercentage
                && vars.percentOfFinalBal.sub(vars.yieldingPercentage) > yieldingPercentageDrift
        ) {
            // we will end up overallocated, withdraw some
            vars.toWithdraw = vars.currentAllocated.sub(vars.finalYieldingAmount);
            vars.yieldingAmount = vars.yieldingAmount.sub(vars.toWithdraw);
            yieldingAmount[_collateral] = vars.yieldingAmount;
        } else if (
            vars.percentOfFinalBal < vars.yieldingPercentage
                && vars.yieldingPercentage.sub(vars.percentOfFinalBal) > yieldingPercentageDrift
        ) {
            // we will end up underallocated, deposit more
            vars.toDeposit = vars.finalYieldingAmount.sub(vars.currentAllocated);
            vars.yieldingAmount = vars.yieldingAmount.add(vars.toDeposit);
            yieldingAmount[_collateral] = vars.yieldingAmount;
        }

        // + means deposit, - means withdraw
        vars.netAssetMovement = int256(vars.toDeposit) - int256(vars.toWithdraw) - int256(vars.profit);
        if (vars.netAssetMovement > 0) {
            ERC20(_collateral).approve(yieldGenerator[_collateral], uint256(vars.netAssetMovement));
            ReaperVaultV2(yieldGenerator[_collateral]).deposit(uint256(vars.netAssetMovement), address(this));
        } else if (vars.netAssetMovement < 0) {
            ReaperVaultV2(yieldGenerator[_collateral]).withdraw(
                uint256(-vars.netAssetMovement), address(this), address(this)
            );
        }

        // if we recorded profit, recalculate it for precision and distribute
        if (vars.profit != 0) {
            // profit is ultimately (coll at hand) + (coll allocated to yield generator) - (recorded total coll Amount in pool)
            vars.profit =
                ERC20(_collateral).balanceOf(address(this)).add(vars.yieldingAmount).sub(collAmount[_collateral]);
            if (vars.profit != 0) {
                // distribute to treasury, staking pool, and stability pool
                vars.treasurySplit = vars.profit.mul(yieldSplitTreasury).div(10_000);
                if (vars.treasurySplit != 0) {
                    ERC20(_collateral).transfer(treasuryAddress, vars.treasurySplit);
                }

                vars.stakingSplit = vars.profit.mul(yieldSplitStaking).div(10_000);
                if (vars.stakingSplit != 0) {
                    ERC20(_collateral).transfer(lqtyStakingAddress, vars.stakingSplit);
                }

                vars.stabilityPoolSplit = vars.profit.sub(vars.treasurySplit.add(vars.stakingSplit));
                if (vars.stabilityPoolSplit != 0) {
                    ERC20(_collateral).transfer(stabilityPoolAddress, vars.stabilityPoolSplit);
                }
            }
        }
    }
}

```

#### ReaperVaultV2.sol

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/AccessControlEnumerable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";

contract ReaperVaultV2 is ERC20, AccessControlEnumerable {
    uint256 totalAllocated = 0;

    IERC20Metadata public immutable token;

    constructor(address _token) ERC20("test", "TEST") {
        token = IERC20Metadata(_token);
    }

    function triggerLoss(uint256 amt) external {
        token.transfer(address(1337), amt);
    }

    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        shares = _deposit(assets, receiver);
    }

    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares) {
        revert("No op");
    }

    function convertToAssets(uint256 shares) public view returns (uint256 assets) {
        if (totalSupply() == 0) return shares;
        return (shares * _freeFunds()) / totalSupply();
    }

    function _deposit(uint256 _amount, address _receiver) internal returns (uint256 shares) {
        require(_amount != 0, "Invalid amount");
        uint256 pool = balance();

        uint256 freeFunds = _freeFunds();
        uint256 balBefore = token.balanceOf(address(this));
        token.transferFrom(msg.sender, address(this), _amount);
        uint256 balAfter = token.balanceOf(address(this));
        _amount = balAfter - balBefore;
        if (totalSupply() == 0) {
            shares = _amount;
        } else {
            shares = (_amount * totalSupply()) / freeFunds; // use "freeFunds" instead of "pool"
        }
        _mint(_receiver, shares);
    }

    function balance() public view returns (uint256) {
        return token.balanceOf(address(this)) + totalAllocated;
    }

    // No harvest, so it's not going to make a difference
    function _freeFunds() public view returns (uint256) {
        return balance();
    }
}

```

#### MockToken.sol

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockToken is ERC20 {
  constructor() ERC20("Mock", "Mock"){
    _mint(msg.sender, 1000e18);
  }
}
```


### Remediation Steps

A slashing mechanism would need to be added to account for a loss.

This should be fairly involved as to not create gotchas.

Intuitively, I believe, that the funds in the activePool would need to be mapped against the funds invested in Vaults as to reconcile the "deposited value" with the "slashed value".

Alternatively, for the time being, a "ShortFall" fund could be instituted, fully knowing that if something goes wrong, the fund will have to cover the loss