## Tags

- bug
- 3 (High Risk)
- sponsor acknowledged
- selected for report

# [LOSS OF PRECISION RESULTING IN WRONG VALUE FOR PRICE RATIO](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/323) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/2175c044af98509261e4147edeb48e1036773771/src/oracles/PegOracle.sol#L46-L83


# Vulnerability details

## Impact
The project implements a price oracle in order to get the relative price between the pegged asset and the price of the original asset (example: stETH to ETH). If the ratio (the pegged asset divided by the original asset) is 1 the Token is pegged, otherwise is depegged. 

Bellow is a code snippet from the [PegOracle.sol](https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/oracles/PegOracle.sol) function.
```
 if (price1 > price2) {
            nowPrice = (price2 * 10000) / price1;
        } else {
            nowPrice = (price1 * 10000) / price2;
        }

        int256 decimals10 = int256(10**(18 - priceFeed1.decimals()));
        nowPrice = nowPrice * decimals10;

        return (
            roundID1,
            nowPrice / 1000000,
            startedAt1,
            timeStamp1,
            answeredInRound1
        );
    }
```

To fetch the ratio at any time, the `PegOracle.sol` performs some calculations; first the relative price is multiplied by 1e4 and then it returns the above calculation divided by 1e6.

The [Controller.sol](https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Controller.sol) file makes an external call to the [PegOracle.sol](https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/oracles/PegOracle.sol) oracle to get the relative price. After, the value returned, it is multiplied by `10**(18-(priceFeed.decimals())` and the result represents the relative price between the two assets.

The result is converted to 18 decimal points in order to be compared with the Strike Price passed by the admin on `VaultFactory.sol`.

Due to the fact that the first multiplication is first divided by `1e6` ([PegOracle.sol#L78)](https://github.com/code-423n4/2022-09-y2k-finance/blob/2175c044af98509261e4147edeb48e1036773771/src/oracles/PegOracle.sol#L78)( and then re-multiplied by `uint256 decimals = 10**(18-(priceFeed.decimals()));` ([Controller.sol#L299-L300](https://github.com/code-423n4/2022-09-y2k-finance/blob/2175c044af98509261e4147edeb48e1036773771/src/Controller.sol#L299-L300))  it leads to loss of precision. This behavior will make the relative price between the assets incorrect.

## Proof of Concept
Bellow is a test that illustrates the above issue for various oracle pairs. The calculated ratio is compared against a modified version of an example of different price denominator, provided by [Chainlink](https://docs.chain.link/docs/get-the-latest-price/#getting-a-different-price-denomination).
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.15;

import "forge-std/Test.sol";
import "../lib/AggregatorV3Interface.sol";

//run with: forge test --fork-url https://arb1.arbitrum.io/rpc -vv 

contract PegOracle {

    /***
    @dev  for example: oracle1 would be stETH / USD, while oracle2 would be ETH / USD oracle
    ***/
    address public oracle1;
    address public oracle2;

    uint8 public decimals;

    AggregatorV3Interface internal priceFeed1;
    AggregatorV3Interface internal priceFeed2;

    /** @notice Contract constructor
      * @param _oracle1 First oracle address
      * @param _oracle2 Second oracle address
      */
    constructor(address _oracle1, address _oracle2) {
        require(_oracle1 != address(0), "oracle1 cannot be the zero address");
        require(_oracle2 != address(0), "oracle2 cannot be the zero address");
        require(_oracle1 != _oracle2, "Cannot be same Oracle");
        priceFeed1 = AggregatorV3Interface(_oracle1);
        priceFeed2 = AggregatorV3Interface(_oracle2);
        require(
            (priceFeed1.decimals() == priceFeed2.decimals()),
            "Decimals must be the same"
        );

        oracle1 = _oracle1;
        oracle2 = _oracle2;

        decimals = priceFeed1.decimals();
    }

    /** @notice Returns oracle-fed data from the latest round
      * @return roundID Current round id 
      * @return nowPrice Current price
      * @return startedAt Starting timestamp
      * @return timeStamp Current timestamp
      * @return answeredInRound Round id for which answer was computed 
      */ 
    function latestRoundData()
        public
        view
        returns (
            uint80 roundID,
            int256 nowPrice,
            uint256 startedAt,
            uint256 timeStamp,
            uint80 answeredInRound
        )
    {
        (
            uint80 roundID1,
            int256 price1,
            uint256 startedAt1,
            uint256 timeStamp1,
            uint80 answeredInRound1
        ) = priceFeed1.latestRoundData();

        int256 price2 = getOracle2_Price();

        if (price1 > price2) {
            nowPrice = (price2 * 10000) / price1;
        } else {
            nowPrice = (price1 * 10000) / price2;
        }

        int256 decimals10 = int256(10**(18 - priceFeed1.decimals()));
        nowPrice = nowPrice * decimals10;

        return (
            roundID1,
            nowPrice / 1000000, //1000000,
            startedAt1,
            timeStamp1,
            answeredInRound1
        );
    }

    /* solhint-disbable-next-line func-name-mixedcase */
    /** @notice Lookup first oracle price
      * @return price Current first oracle price
      */ 
    function getOracle1_Price() public view returns (int256 price) {
        (
            uint80 roundID1,
            int256 price1,
            ,
            uint256 timeStamp1,
            uint80 answeredInRound1
        ) = priceFeed1.latestRoundData();

        require(price1 > 0, "Chainlink price <= 0");
        require(
            answeredInRound1 >= roundID1,
            "RoundID from Oracle is outdated!"
        );
        require(timeStamp1 != 0, "Timestamp == 0 !");

        return price1;
    }

    /* solhint-disbable-next-line func-name-mixedcase */
    /** @notice Lookup second oracle price
      * @return price Current second oracle price
      */ 
    function getOracle2_Price() public view returns (int256 price) {
        (
            uint80 roundID2,
            int256 price2,
            ,
            uint256 timeStamp2,
            uint80 answeredInRound2
        ) = priceFeed2.latestRoundData();

        require(price2 > 0, "Chainlink price <= 0");
        require(
            answeredInRound2 >= roundID2,
            "RoundID from Oracle is outdated!"
        );
        require(timeStamp2 != 0, "Timestamp == 0 !");

        return price2;
    }
    
    function latestRoundData2()
        public
        view
        returns (
            uint80 roundID,
            int256 nowPrice,
            uint256 startedAt,
            uint256 timeStamp,
            uint80 answeredInRound
        )
    {
        (
            uint80 roundID1,
            int256 price1,
            uint256 startedAt1,
            uint256 timeStamp1,
            uint80 answeredInRound1
        ) = priceFeed1.latestRoundData();

        price1 = scalePriceTo18(price1, priceFeed1.decimals());

        int256 price2 = scalePriceTo18(getOracle2_Price(), priceFeed1.decimals());


        return (
            roundID1,
            price1  * 1e18 / price2, 
            startedAt1,
            timeStamp1,
            answeredInRound1
        );
    }


function scalePriceTo18(int256 _price, uint8 _priceDecimals)
        internal
        pure
        returns (int256)
    {
        if (_priceDecimals < 18) {
            return _price * int256(10 ** uint256(18 - _priceDecimals));
        } else if (_priceDecimals > 18) {
            return _price * int256(10 ** uint256(_priceDecimals - 18));
        }
        return _price;
    }
} 




contract TestOracles is Test {
    address WETH = 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1;

    address tokenFRAX = 0x17FC002b466eEc40DaE837Fc4bE5c67993ddBd6F;
    address tokenMIM = 0xFEa7a6a0B346362BF88A9e4A88416B77a57D6c2A;
    address tokenFEI = 0x4A717522566C7A09FD2774cceDC5A8c43C5F9FD2;
    address tokenUSDC = 0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8;
    address tokenDAI = 0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1;
    address tokenSTETH = 0xEfa0dB536d2c8089685630fafe88CF7805966FC3;

    address oracleFRAX = 0x0809E3d38d1B4214958faf06D8b1B1a2b73f2ab8;
    address oracleMIM = 0x87121F6c9A9F6E90E59591E4Cf4804873f54A95b;
    address oracleFEI = 0x7c4720086E6feb755dab542c46DE4f728E88304d;
    address oracleUSDC = 0x50834F3163758fcC1Df9973b6e91f0F0F0434aD3;
    address oracleDAI = 0xc5C8E77B397E531B8EC06BFb0048328B30E9eCfB;
    address oracleSTETH = 0x07C5b924399cc23c24a95c8743DE4006a32b7f2a;
    address oracleETH = 0x639Fe6ab55C921f74e7fac1ee960C0B6293ba612;
    address btcEthOracle = 0xc5a90A6d7e4Af242dA238FFe279e9f2BA0c64B2e;

    PegOracle pegOracle = new PegOracle(oracleSTETH, oracleETH);
    PegOracle pegOracle2 = new PegOracle(oracleFRAX, oracleFEI);
    PegOracle pegOracle3 = new PegOracle(oracleDAI, oracleFEI);

    function setUp() public {}

    function convertBasedOnContractsLogic(int256 price, uint8 oracleDecimals) public returns(int256 newPrice){
        uint256 decimals = 10**(18- oracleDecimals );
        int256 newPrice = price * int256(decimals);
        return newPrice;
    }

    function testOraclePrices() public {
        (, int256 var1 ,,,) = pegOracle.latestRoundData();
        emit log_int(convertBasedOnContractsLogic(var1, pegOracle.decimals()));

        (, int256 var2 ,,,) = pegOracle.latestRoundData2();
        emit log_int(var2);

        (, int256 var3 ,,,) = pegOracle2.latestRoundData();
        emit log_int(convertBasedOnContractsLogic(var3, pegOracle2.decimals()));

        (, int256 var4 ,,,) = pegOracle2.latestRoundData2();
        emit log_int(var4);


        (, int256 var5 ,,,) = pegOracle3.latestRoundData();
        emit log_int(convertBasedOnContractsLogic(var5, pegOracle3.decimals()));

        (, int256 var6 ,,,) = pegOracle3.latestRoundData2();
        emit log_int(var6);
    }

}
```
Here is the output after running the with: `forge test --fork-url https://arb1.arbitrum.io/rpc -vv `:
  990500000000000000
  990544616614592905

  996300000000000000
  1003669952945847834

  996000000000000000
  1003940775578783463


## Tools Used
Manual Review

## Recommended Mitigation Steps
Since the 2 assets are required to having the same amount of decimals a formula that transforms the relative price to 1e18 could be:
`x * 1e18 / y` .

An example that Chainlink implements, that includes a `scalePrice` function, in order to find a different price denominator could be found [here](https://docs.chain.link/docs/get-the-latest-price/#getting-a-different-price-denomination)