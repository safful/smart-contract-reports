## Tags

- bug
- 3 (High Risk)
- resolved
- sponsor confirmed
- old-submission-method

# [Zero strike call options can be systemically used to steal premium from the taker](https://github.com/code-423n4/2022-06-putty-findings/issues/418) 

# Lines of code

https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L435-L437


# Vulnerability details

Some non-malicious ERC20 do not allow for zero amount transfers and order.baseAsset can be such an asset. Zero strike calls are valid and common enough derivative type. However, the zero strike calls with such baseAsset will not be able to be exercised, allowing maker to steal from the taker as a malicious maker can just wait for expiry and withdraw the assets, effectively collecting the premium for free. The premium of zero strike calls are usually substantial.

Marking this as high severity as in such cases malicious maker knowing this specifics can steal from taker the whole premium amount. I.e. such orders will be fully valid for a taker from all perspectives as inability to exercise is a peculiarity of the system which taker in the most cases will not know beforehand.

## Proof of Concept

Currently system do not check the strike value, unconditionally attempting to transfer it:

https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L435-L437

```solidity
            } else {
                ERC20(order.baseAsset).safeTransferFrom(msg.sender, address(this), order.strike);
            }
```

As a part of call exercise logic:

https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L422-L443

```solidity
    function exercise(Order memory order, uint256[] calldata floorAssetTokenIds) public payable {
        ...

        if (order.isCall) {
            // -- exercising a call option

            // transfer strike from exerciser to putty
            // handle the case where the taker uses native ETH instead of WETH to pay the strike
            if (weth == order.baseAsset && msg.value > 0) {
                // check enough ETH was sent to cover the strike
                require(msg.value == order.strike, "Incorrect ETH amount sent");

                // convert ETH to WETH
                // we convert the strike ETH to WETH so that the logic in withdraw() works
                // - because withdraw() assumes an ERC20 interface on the base asset.
                IWETH(weth).deposit{value: msg.value}();
            } else {
                ERC20(order.baseAsset).safeTransferFrom(msg.sender, address(this), order.strike);
            }

            // transfer assets from putty to exerciser
            _transferERC20sOut(order.erc20Assets);
            _transferERC721sOut(order.erc721Assets);
            _transferFloorsOut(order.floorTokens, positionFloorAssetTokenIds[uint256(orderHash)]);
        }
```

Some tokens do not allow zero amount transfers:

https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers

This way for such a token and zero strike option the maker can create short call order, receive the premium:

https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L327-L339

```solidity
            if (weth == order.baseAsset && msg.value > 0) {
                // check enough ETH was sent to cover the premium
                require(msg.value == order.premium, "Incorrect ETH amount sent");

                // convert ETH to WETH and send premium to maker
                // converting to WETH instead of forwarding native ETH to the maker has two benefits;
                // 1) active market makers will mostly be using WETH not native ETH
                // 2) attack surface for re-entrancy is reduced
                IWETH(weth).deposit{value: msg.value}();
                IWETH(weth).transfer(order.maker, msg.value);
            } else {
                ERC20(order.baseAsset).safeTransferFrom(msg.sender, order.maker, order.premium);
            }
```

Transfer in the assets:

https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L366-L371

```solidity
        // filling short call: transfer assets from maker to contract
        if (!order.isLong && order.isCall) {
            _transferERC20sIn(order.erc20Assets, order.maker);
            _transferERC721sIn(order.erc721Assets, order.maker);
            return positionId;
        }
```

And wait for expiration, knowing that all attempts to exercise will revert:

https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L435-L437

```solidity
            } else {
                ERC20(order.baseAsset).safeTransferFrom(msg.sender, address(this), order.strike);
            }
```

Then recover her assets:

https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L508-L519

```solidity
        // transfer assets from putty to owner if put is exercised or call is expired
        if ((order.isCall && !isExercised) || (!order.isCall && isExercised)) {
            _transferERC20sOut(order.erc20Assets);
            _transferERC721sOut(order.erc721Assets);

            // for call options the floor token ids are saved in the long position in fillOrder(),
            // and for put options the floor tokens ids are saved in the short position in exercise()
            uint256 floorPositionId = order.isCall ? longPositionId : uint256(orderHash);
            _transferFloorsOut(order.floorTokens, positionFloorAssetTokenIds[floorPositionId]);

            return;
        }
```

## Recommended Mitigation Steps

Consider checking that strike is positive before transfer in all the cases, for example:

```solidity
            } else {
+               if (order.strike > 0) {
                    ERC20(order.baseAsset).safeTransferFrom(msg.sender, address(this), order.strike);
+               }
            }
```

