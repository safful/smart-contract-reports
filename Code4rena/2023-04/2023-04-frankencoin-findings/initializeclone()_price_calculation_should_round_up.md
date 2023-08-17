## Tags

- bug
- 2 (Med Risk)
- low quality report
- selected for report
- M-08

# [initializeClone() price calculation should round up](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/663) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Position.sol#L80


# Vulnerability details

## Impact
`initializeClone()` Price calculations use `round down`, which only works if divisible , If precision is lost, it will revert

## Proof of Concept
When clone positon, `_initialCollateral` and `_initialMint` are used to calculate the `price` of the clone position

The code is as follows:
```solidity
    function initializeClone(address owner, uint256 _price, uint256 _limit, uint256 _coll, uint256 _mint) external onlyHub {
        if(_coll < minimumCollateral) revert InsufficientCollateral();
        setOwner(owner);
        
        price = _mint * ONE_DEC18 / _coll;   //<----------use round down

        if (price > _price) revert InsufficientCollateral();
        limit = _limit;
        mintInternal(owner, _mint, _coll);

        emit PositionOpened(owner, original, address(zchf), address(collateral), _price);
    }

```

1. The price calculation formula `price = _mint * ONE_DEC18 / _coll`，  use `round down`

2. In the next step `mintInternal()` will execute mint, and internally will call `checkCollateral()`

```solidity
    function checkCollateral(uint256 collateralReserve, uint256 atPrice) internal view {
        if (collateralReserve * atPrice < minted * ONE_DEC18) revert InsufficientCollateral();
    }
```

`checkCollateral()` will check `collateralReserve * atPrice < minted * ONE_DEC18`

This has a problem, when calculating the price and there is a precision loss in price (round down), then `checkCollateral()` will definitely revert

Because if precision loss occurs, `collateralReserve * atPrice` will be 1 less than `minted * ONE_DEC18`


For example:

_initialCollateral = 101e18;
_initialMint = 100e18;

Due to round down price = 0.99e18

Then `checkCollateral () ` will revert because `101e18 * 0.99e18 < 100e18 * 1e18`

Here is the demo code:

will revert `InsufficientCollateral` in `checkCollateral () ` 
add to GeneralTest.t.sol

```solidity
    function testCloneRevert() external {

        // 0.get 1000 for open bad position
        alice.obtainFrankencoins(swap, 1000 ether);
        col.mint(address(alice), 1001);  

        // 1. open new position
        vm.startPrank(address(alice));
        col.approve(address(hub), 1001);
        uint256 oldPrice = 1 * (10 ** 36);
        Position pos = Position(hub.openPosition(address(col), 100, 1001, 1000000 ether, 100 days, 1 days, 25000, oldPrice, 200000));        
        skip(7 * 86_400 + 60);

        console.log("0.pos price:",pos.price());

        // 2.pass _initialMint  _initialCollateral ,  will  round down
        uint256 _initialCollateral = 1001;
        uint256 _initialMint = 1000 * 10**18;

        uint256 newPrice = _initialMint * 10**18 / _initialCollateral;

        console.log("1.new price:",newPrice);
        console.log("1.new price Bigger than old ?:",newPrice > oldPrice);

        col.mint(address(alice), _initialCollateral);
        col.approve(address(hub), _initialCollateral);

        // 3. clonePosition will revert ,  Although the parameters are all legal
        hub.clonePosition(address(pos),_initialCollateral , _initialMint);
        vm.stopPrank();
    }

}
```

```console
$ forge test --match testCloneRevert -vvv

Running 1 test for test/GeneralTest.t.sol:GeneralTest
[FAIL. Reason: InsufficientCollateral()] testCloneRevert() (gas: 2230875)
Logs:
  0.pos price: 1000000000000000000000000000000000000
  1.new price: 999000999000999000999000999000999000
  1.new price Bigger than old ?: false

Traces:
  [2131375] GeneralTest::testCloneRevert() 
```

It is recommended to use round up.

## Tools Used

## Recommended Mitigation Steps

```solidity
    function initializeClone(address owner, uint256 _price, uint256 _limit, uint256 _coll, uint256 _mint) external onlyHub {
...

        price = _mint * ONE_DEC18 / _coll;

+       if ( price * _coll != _mint * ONE_DEC18 ) price += 1; // round up, avoid mintInternal() revert InsufficientCollateral
```