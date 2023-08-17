## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- satisfactory
- sponsor acknowledged
- selected for report
- M-18

# [Protocol's usability becomes very limited when access to Chainlink oracle data feed is blocked](https://github.com/code-423n4/2022-10-inverse-findings/issues/586) 

# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Oracle.sol#L78-L105
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Oracle.sol#L112-L144
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L344-L347
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L323-L327
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L353-L363


# Vulnerability details

## Impact
Based on the current implementation, when the protocol wants to use Chainlink oracle data feed for getting a collateral token's price, the fixed price for the token should not be set. When the fixed price is not set for the token, calling the `Oracle` contract's `viewPrice` or `getPrice` function will execute `uint price = feeds[token].feed.latestAnswer()`. As https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/ mentions, it is possible that Chainlink’s "multisigs can immediately block access to price feeds at will". When this occurs, executing `feeds[token].feed.latestAnswer()` will revert so calling the `viewPrice` and `getPrice` functions also revert, which cause denial of service when calling functions like `getCollateralValueInternal` and`getWithdrawalLimitInternal`. The `getCollateralValueInternal` and`getWithdrawalLimitInternal` functions are the key elements to the core functionalities, such as borrowing, withdrawing, force-replenishing, and liquidating; with these functionalities facing DOS, the protocol's usability becomes very limited.

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Oracle.sol#L78-L105
```solidity
    function viewPrice(address token, uint collateralFactorBps) external view returns (uint) {
        if(fixedPrices[token] > 0) return fixedPrices[token];
        if(feeds[token].feed != IChainlinkFeed(address(0))) {
            // get price from feed
            uint price = feeds[token].feed.latestAnswer();
            require(price > 0, "Invalid feed price");
            // normalize price
            uint8 feedDecimals = feeds[token].feed.decimals();
            uint8 tokenDecimals = feeds[token].tokenDecimals;
            uint8 decimals = 36 - feedDecimals - tokenDecimals;
            uint normalizedPrice = price * (10 ** decimals);
            uint day = block.timestamp / 1 days;
            // get today's low
            uint todaysLow = dailyLows[token][day];
            // get yesterday's low
            uint yesterdaysLow = dailyLows[token][day - 1];
            // calculate new borrowing power based on collateral factor
            uint newBorrowingPower = normalizedPrice * collateralFactorBps / 10000;
            uint twoDayLow = todaysLow > yesterdaysLow && yesterdaysLow > 0 ? yesterdaysLow : todaysLow;
            if(twoDayLow > 0 && newBorrowingPower > twoDayLow) {
                uint dampenedPrice = twoDayLow * 10000 / collateralFactorBps;
                return dampenedPrice < normalizedPrice ? dampenedPrice: normalizedPrice;
            }
            return normalizedPrice;

        }
        revert("Price not found");
    }
```

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Oracle.sol#L112-L144
```solidity
    function getPrice(address token, uint collateralFactorBps) external returns (uint) {
        if(fixedPrices[token] > 0) return fixedPrices[token];
        if(feeds[token].feed != IChainlinkFeed(address(0))) {
            // get price from feed
            uint price = feeds[token].feed.latestAnswer();
            require(price > 0, "Invalid feed price");
            // normalize price
            uint8 feedDecimals = feeds[token].feed.decimals();
            uint8 tokenDecimals = feeds[token].tokenDecimals;
            uint8 decimals = 36 - feedDecimals - tokenDecimals;
            uint normalizedPrice = price * (10 ** decimals);
            // potentially store price as today's low
            uint day = block.timestamp / 1 days;
            uint todaysLow = dailyLows[token][day];
            if(todaysLow == 0 || normalizedPrice < todaysLow) {
                dailyLows[token][day] = normalizedPrice;
                todaysLow = normalizedPrice;
                emit RecordDailyLow(token, normalizedPrice);
            }
            // get yesterday's low
            uint yesterdaysLow = dailyLows[token][day - 1];
            // calculate new borrowing power based on collateral factor
            uint newBorrowingPower = normalizedPrice * collateralFactorBps / 10000;
            uint twoDayLow = todaysLow > yesterdaysLow && yesterdaysLow > 0 ? yesterdaysLow : todaysLow;
            if(twoDayLow > 0 && newBorrowingPower > twoDayLow) {
                uint dampenedPrice = twoDayLow * 10000 / collateralFactorBps;
                return dampenedPrice < normalizedPrice ? dampenedPrice: normalizedPrice;
            }
            return normalizedPrice;

        }
        revert("Price not found");
    }
```

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L344-L347
```solidity
    function getCreditLimitInternal(address user) internal returns (uint) {
        uint collateralValue = getCollateralValueInternal(user);
        return collateralValue * collateralFactorBps / 10000;
    }
```

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L323-L327
```solidity
    function getCollateralValueInternal(address user) internal returns (uint) {
        IEscrow escrow = predictEscrow(user);
        uint collateralBalance = escrow.balance();
        return collateralBalance * oracle.getPrice(address(collateral), collateralFactorBps) / 1 ether;
    }
```

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L353-L363
```solidity
    function getWithdrawalLimitInternal(address user) internal returns (uint) {
        IEscrow escrow = predictEscrow(user);
        uint collateralBalance = escrow.balance();
        if(collateralBalance == 0) return 0;
        uint debt = debts[user];
        if(debt == 0) return collateralBalance;
        if(collateralFactorBps == 0) return 0;
        uint minimumCollateral = debt * 1 ether / oracle.getPrice(address(collateral), collateralFactorBps) * 10000 / collateralFactorBps;
        if(collateralBalance <= minimumCollateral) return 0;
        return collateralBalance - minimumCollateral;
    }
```

## Proof of Concept
The following steps can occur for the described scenario.
1. Chainlink oracle data feed is used for getting the collateral token's price so the fixed price for the token is not set.
2. Alice calls the `depositAndBorrow` function to deposit some of the collateral token and borrows some DOLA against the collateral.
3. Chainlink's multisigs suddenly blocks access to price feeds so executing `feeds[token].feed.latestAnswer()` will revert.
4. Alice tries to borrow more DOLA but calling the `borrow` function, which eventually executes `feeds[token].feed.latestAnswer()`, reverts.
5. Alice tries to withdraw the deposited collateral but calling the `withdraw` function, which eventually executes `feeds[token].feed.latestAnswer()`, reverts.
6. Similarly, calling the `forceReplenish` and `liquidate` functions would all revert as well.

## Tools Used
VSCode

## Recommended Mitigation Steps
The `Oracle` contract's `viewPrice` and `getPrice` functions can be updated to refactor `feeds[token].feed.latestAnswer()` into `try feeds[token].feed.latestAnswer() returns (int256 price) { ... } catch Error(string memory) { ... }`. The logic for getting the collateral token's price from the Chainlink oracle data feed should be placed in the `try` block while some fallback logic when the access to the Chainlink oracle data feed is denied should be placed in the `catch` block. If getting the fixed price for the collateral token is considered as a fallback logic, then setting the fixed price for the token should become mandatory, which is different from the current implementation. Otherwise, fallback logic for getting the token's price from a fallback oracle is needed.