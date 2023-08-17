## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor disputed
- H-10

# [User can abuse tight stop losses and high leverage to make risk free trades](https://github.com/code-423n4/2022-12-tigris-findings/issues/622) 

# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/TradingExtension.sol#L88-L120


# Vulnerability details

## Impact

User can abuse how stop losses are priced to open high leverage trades with huge upside and very little downside

## Proof of Concept

    function limitClose(
        uint _id,
        bool _tp,
        PriceData calldata _priceData,
        bytes calldata _signature
    )
        external
    {
        _checkDelay(_id, false);
        (uint _limitPrice, address _tigAsset) = tradingExtension._limitClose(_id, _tp, _priceData, _signature);
        _closePosition(_id, DIVISION_CONSTANT, _limitPrice, address(0), _tigAsset, true);
    }

    function _limitClose(
        uint _id,
        bool _tp,
        PriceData calldata _priceData,
        bytes calldata _signature
    ) external view returns(uint _limitPrice, address _tigAsset) {
        _checkGas();

        IPosition.Trade memory _trade = position.trades(_id);
        _tigAsset = _trade.tigAsset;
        getVerifiedPrice(_trade.asset, _priceData, _signature, 0);
        uint256 _price = _priceData.price;
        if (_trade.orderType != 0) revert("4"); //IsLimit
        if (_tp) {
            if (_trade.tpPrice == 0) revert("7"); //LimitNotSet
            if (_trade.direction) {
                if (_trade.tpPrice > _price) revert("6"); //LimitNotMet
            } else {
                if (_trade.tpPrice < _price) revert("6"); //LimitNotMet
            }
            _limitPrice = _trade.tpPrice;
        } else {
            if (_trade.slPrice == 0) revert("7"); //LimitNotSet
            if (_trade.direction) {
                if (_trade.slPrice < _price) revert("6"); //LimitNotMet
            } else {
                if (_trade.slPrice > _price) revert("6"); //LimitNotMet
            }
            //@audit stop loss is closed at user specified price NOT market price
            _limitPrice = _trade.slPrice;
        }
    }

When closing a position with a stop loss the user is closed at their SL price rather than the current price of the asset. A user could abuse this in directional markets with high leverage to make nearly risk free trades. A user could open a long with a stop loss that in $0.01 below the current price. If the price tanks immediately on the next update then they will be closed out at their entrance price, only out the fees to open and close their position. If the price goes up then they can make a large gain.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Take profit and stop loss trades should be executed at the current price rather than the price specified by the user:

             if (_trade.tpPrice == 0) revert("7"); //LimitNotSet
            if (_trade.direction) {
                if (_trade.tpPrice > _price) revert("6"); //LimitNotMet
            } else {
                if (_trade.tpPrice < _price) revert("6"); //LimitNotMet
            }
    -       _limitPrice = _trade.tpPrice;
    +       _limitPrice = _price;
        } else {
            if (_trade.slPrice == 0) revert("7"); //LimitNotSet
            if (_trade.direction) {
                if (_trade.slPrice < _price) revert("6"); //LimitNotMet
            } else {
                if (_trade.slPrice > _price) revert("6"); //LimitNotMet
            }
    -       _limitPrice = _trade.slPrice;
    +       _limitPrice = _price;