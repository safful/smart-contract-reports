## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-24

# [BasketHandler: Users might not be able to redeem their rToken when protocol is paused due to refreshBasket function](https://github.com/code-423n4/2023-01-reserve-findings/issues/39) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L439-L514
https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L448
https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BasketHandler.sol#L183-L192


# Vulnerability details

## Impact
The Reserve protocol allows redemption of rToken even when the protocol is `paused`.  

The `docs/system-design.md` documentation describes the `paused` state as:  

>all interactions disabled EXCEPT RToken.redeem + RToken.cancel + ERC20 functions + StRSR.stake

Redemption of rToken should only ever be prohibited when the protocol is in the `frozen` state.  

You can see that the `RToken.redeem` function ([https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L439-L514](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L439-L514)) has the `notFrozen` modifier so it can be called when the protocol is in the `paused` state.  

The issue is that this function relies on the `BasketHandler.status()` to not be `DISABLED` ([https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L448](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L448)).  

The `BasketHandler.refreshBasket` function ([https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BasketHandler.sol#L183-L192](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BasketHandler.sol#L183-L192)) however, which must be called to get the basket out of the `DISABLED` state, cannot be called by any user when the protocol is paused.  

When the protocol is paused it can only be called by the governance (OWNER) address.  

So in case the basket is `DISABLED` and the protocol is paused, it is the governance that must call `refreshBasket` to allow redemption of rToken.  

This is dangerous because redemption of rToken should not rely on governance to perform any actions such that users can get out of the protocol when there is something wrong with the governance technically or if the governance behaves badly.  

## Proof of Concept
The `RToken.redeem` function has the `notFrozen` modifier so it can be called when the protocol is paused ([https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L439](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/RToken.sol#L439)).  

The `BasketHandler.refreshBasket` function can only be called by the governance when the protocol is paused:  

[https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BasketHandler.sol#L186-L190](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/BasketHandler.sol#L186-L190)  
```solidity
require(
    main.hasRole(OWNER, _msgSender()) ||
        (status() == CollateralStatus.DISABLED && !main.pausedOrFrozen()),
    "basket unrefreshable"
);
```

Therefore the situation exists where rToken redemption should be possible but it is blocked by the `BasketHandler.refreshBasket` function.  

## Tools Used
VSCode

## Recommended Mitigation Steps
The `BasketHandler.refreshBasket` function should be callable by anyone when the `status()` is `DISABLED` and the protocol is `paused`.  

So the above `require` statement can be changed like this:  

```
diff --git a/contracts/p1/BasketHandler.sol b/contracts/p1/BasketHandler.sol
index f74155b1..963e29de 100644
--- a/contracts/p1/BasketHandler.sol
+++ b/contracts/p1/BasketHandler.sol
@@ -185,7 +185,7 @@ contract BasketHandlerP1 is ComponentP1, IBasketHandler {
 
         require(
             main.hasRole(OWNER, _msgSender()) ||
-                (status() == CollateralStatus.DISABLED && !main.pausedOrFrozen()),
+                (status() == CollateralStatus.DISABLED && !main.frozen()),
             "basket unrefreshable"
         );
         _switchBasket();
```

It was discussed with the sponsor that they might even allow rToken redemption when the basket is `DISABLED`.  

In other words only disallow it when the protocol is `frozen`.  

This however needs further consideration by the sponsor as it might negatively affect other aspects of the protocol that are beyond the scope of this report.  