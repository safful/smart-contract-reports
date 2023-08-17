## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-08

# [Adversary can force user to pay large gas fees by transfering them collateral](https://github.com/code-423n4/2022-11-paraspace-findings/issues/321) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/libraries/logic/SupplyLogic.sol#L462-L512


# Vulnerability details

## Impact

Adversary can DOS user and make them pay more gas by sending them collateral

## Proof of Concept

    if (fromConfig.isUsingAsCollateral(reserveId)) {
        if (fromConfig.isBorrowingAny()) {
            ValidationLogic.validateHFAndLtvERC20(
                reservesData,
                reservesList,
                usersConfig[params.from],
                params.asset,
                params.from,
                params.reservesCount,
                params.oracle
            );
        }

        if (params.balanceFromBefore == params.amount) {
            fromConfig.setUsingAsCollateral(reserveId, false);
            emit ReserveUsedAsCollateralDisabled(
                params.asset,
                params.from
            );
        }

        //@audit collateral is automatically turned on for receiver
        if (params.balanceToBefore == 0) {
            DataTypes.UserConfigurationMap
                storage toConfig = usersConfig[params.to];

            toConfig.setUsingAsCollateral(reserveId, true);
            emit ReserveUsedAsCollateralEnabled(
                params.asset,
                params.to
            );
        }
    }

The above lines are executed when a user transfer collateral to another user. If the sending user currently has the collateral enabled and the receiving user doesn't have a balance already, the collateral will automatically be enabled for the receiver. Since the collateral is enabled, it will now be factored into the health check calculation. This increases gas for the receiver every time the user does anything that requires a health check (which ironically includes turning off a collateral). If enough different kinds of collateral are added to the platform it may even be enough gas to DOS the users.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Don't automatically enable the collateral for the receiver.