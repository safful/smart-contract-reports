## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-02

# [A temporary issue shows in the staking functionality which leads to the users receiving less minted tokens.](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/1004) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L63-L101
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L156-L204
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L211-L216


# Vulnerability details

## Derivative Reth prices 
A quick explanation of the issue causing it, the problem is based on the function "ethPerDerivative" in the Reth derivative.
As you can see two statements can be triggered here, the first one "if (poolCanDeposit(_amount))" checks if the given amount + the pool balance isn't greater than the maximumDepositPoolSize and that the amount is greater than the minimum deposit in the pool. Second statement is meant to return a poolPrice which is slightly more than the regular one, because it's used in order to swap tokens in Uniswap and therefore the price per token is overpriced.

```solidity
function ethPerDerivative(uint256 _amount) public view returns (uint256) {
        if (poolCanDeposit(_amount))
            return
                RocketTokenRETHInterface(rethAddress()).getEthValue(10 ** 18);
        else return (poolPrice() * 10 ** 18) / (10 ** 18);
    }
```
``` solidity
// poolCanDeposit() returns:
      return
            rocketDepositPool.getBalance() + _amount <=
            rocketDAOProtocolSettingsDeposit.getMaximumDepositPoolSize() &&
            _amount >= rocketDAOProtocolSettingsDeposit.getMinimumDeposit();
```

Below you can see the regular price returned in the first statement - 1063960369075232250:


<img width="417" alt="Screenshot 2023-03-27 at 8 53 34" src="https://user-images.githubusercontent.com/112419701/227852484-9bf0144d-820d-414c-8cb9-e1fb44f5c806.png">

Below you can see the pool price from the second statement, supposed to be used only when a swap is made.

```solidity
else return (poolPrice() * 10 ** 18) / (10 ** 18);

// poolPrice calculates and returns
uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
        return (sqrtPriceX96 * (uint(sqrtPriceX96)) * (1e18)) >> (96 * 2);

// uint160 sqrtPriceX96 = 81935751724326368909606241317
// return (sqrtPriceX96 * (uint(sqrtPriceX96)) * (1e18)) >> (96 * 2);
// return 1069517062752670179 (pool price)

// The function "ethPerDerivative" for the else statement return (poolPrice() * 10 ** 18) / (10 ** 18);
// Which will be - 1069517062752670179
```

Difference between the regular price and the pool price:

```
regular price - 1063960369075232250
pool price -    1069517062752670179
```

## Quick Overview

What can result to users receiving less minted tokens?

The first thing the staking function does is calculating the derivative underlyingValue. This issue occurs on the Reth derivative, as we can see the staking function calls "ethPerDerivative" to get the price, but takes as account the whole Reth balance of the derivative contract. 

For example let's say the derivative Reth holds 200e18. The pool has free space for 100e18 more till it reaches its maximum pool size. As the function calls ethPerDerivative with the Reth balance of 200e18 instead of the amount being staked.
The contract will think there is no more space in the pool (even tho there is 100e18 more) and will return the pool price which is overpriced and meant for the swap in Uniswap.

```solidity
underlyingValue +=
                (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
                    derivatives[i].balance()) /
                10 ** 18;

```
```solidity
function ethPerDerivative(uint256 _amount) public view returns (uint256) {
        if (poolCanDeposit(_amount))
            return
                RocketTokenRETHInterface(rethAddress()).getEthValue(10 ** 18);
        else return (poolPrice() * 10 ** 18) / (10 ** 18);
    }

// poolCanDeposit(_amount)
return
        rocketDepositPool.getBalance() + _amount <=
        rocketDAOProtocolSettingsDeposit.getMaximumDepositPoolSize() &&
        _amount >= rocketDAOProtocolSettingsDeposit.getMinimumDeposit();
```

Lets follow what actually happens, for now we have wrong overpriced underlying value of the derivative Reth.

Next the function calculates the preDepositPrice. l will do the real calculations in the POC, but its easy to assume that if the underlyingValue is overpriced the preDepositPrice will be too based on the calculation below.

```solidity
else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
```

// Lets say the user deposits 5e18

Here comes the real problem, so far the function calculates the local variables as there will be swap to Uniswap.
As mentioned in the beginning the pool has 100e18 free space, so in the deposit function in Reth, the swap to Uniswap will be ignored as "poolCanDeposit(msg.value) == true" and the msg.value will be deposited in the rocket pool.

```solidity
uint256 depositAmount = derivative.deposit{value: ethAmount}();
```
```solidity
function deposit() external payable onlyOwner returns (uint256) {
        // Per RocketPool Docs query addresses each time it is used
        address rocketDepositPoolAddress = RocketStorageInterface(
            ROCKET_STORAGE_ADDRESS
        ).getAddress(
                keccak256(
                    abi.encodePacked("contract.address", "rocketDepositPool")
                )
            );

        RocketDepositPoolInterface rocketDepositPool = RocketDepositPoolInterface(
                rocketDepositPoolAddress
            );

        if (!poolCanDeposit(msg.value)) {
            uint rethPerEth = (10 ** 36) / poolPrice();

            uint256 minOut = ((((rethPerEth * msg.value) / 10 ** 18) *
                ((10 ** 18 - maxSlippage))) / 10 ** 18);

            IWETH(W_ETH_ADDRESS).deposit{value: msg.value}();
            uint256 amountSwapped = swapExactInputSingleHop(
                W_ETH_ADDRESS,
                rethAddress(),
                500,
                msg.value,
                minOut
            );

            return amountSwapped;
        } else {
            address rocketTokenRETHAddress = RocketStorageInterface(
                ROCKET_STORAGE_ADDRESS
            ).getAddress(
                    keccak256(
                        abi.encodePacked("contract.address", "rocketTokenRETH")
                    )
                );
            RocketTokenRETHInterface rocketTokenRETH = RocketTokenRETHInterface(
                rocketTokenRETHAddress
            );
            uint256 rethBalance1 = rocketTokenRETH.balanceOf(address(this));
            rocketDepositPool.deposit{value: msg.value}();
            uint256 rethBalance2 = rocketTokenRETH.balanceOf(address(this));
            require(rethBalance2 > rethBalance1, "No rETH was minted");
            uint256 rethMinted = rethBalance2 - rethBalance1;
            return (rethMinted);
        }
    }
```

Next the function calculates the "derivativeReceivedEthValue", this time the function ethPerDerivative(depositAmount) will return the normal price as there is space in the pool. Both "derivativeReceivedEthValue" and "totalStakeValueEth" will be calculated based on the normal price.

```solidity
uint derivativeReceivedEthValue = (derivative.ethPerDerivative(
                depositAmount
            ) * depositAmount) / 10 ** 18;
```
```solidity
totalStakeValueEth += derivativeReceivedEthValue;
```

If we take the info so far and apply it on the mintAmount calculation below, we know that "totalStakeValueEth" is calculated on the normal price and "preDepositPrice" is calculated on the overpriced pool price. So the user will actually receive less minted shares than he is supposed to get.

```solidity
uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
```


## Proof of Concept - Part 1

Will start from the start in order to get the right amounts of "totalSupply" and the Reth balance of derivative.
So l can show the issue result in POC Part 2.

// The values below are only made for the example.

Let's say we have two stakers - Bob and Kiki each depositing 100e18.
We have only one derivative which is Reth, so it will have 100% weight.

Bob deposits 100e18 as the first depositer and receives (99999999999999999932) minted tokens of safETH.

So far after Bob deposit:

totalSupply = 99999999999999999932

Reth derivative balance = 93988463204618701706

```solidity
uint256 underlyingValue = 0;
uint256 totalSupply = 0; 
uint256 preDepositPrice = 1e18

// As we have only derivative Reth in the example, it owns all of the weight.
uint256 ethAmount = (msg.value * weight) / totalWeight;
uint256 ethAmount = (100e18 * 1000) / 1000;

// not applying the deposit fee in rocketPool

uint256 depositAmount = derivative.deposit{value: ethAmount}();
uint256 depositAmount = 93988463204618701706

uint derivativeReceivedEthValue = (derivative.ethPerDerivative(depositAmount) * depositAmount) / 10 ** 18;
uint derivativeReceivedEthValue = (1063960369075232250 * 93988463204618701706) / 10 ** 18;
uint derivativeReceivedEthValue = 99999999999999999932

totalStakeValueEth = 99999999999999999932;

 
uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
uint256 mintAmount = (99999999999999999932 * 10 ** 18) / 1e18;
uint256 mintAmount = 99999999999999999932
```

Kiki deposits 100e18 as well and receives (99999999999999999932) minted tokens of safEth.

So far after Kiki's deposit:

totalSupply = 199999999999999999864;

Reth derivative balance = 187976926409237403412;


```solidity
// take the info after bob's deposit and the normal price
underlyingValue  = (derivatives[i].ethPerDerivative(derivatives[i].balance()) * derivatives[i].balance()) / 10 ** 18;
uint256 underlyingValue = (1063960369075232250 * 93988463204618701706) / 10 ** 18;
uint256 underlyingValue = 99999999999999999932;

uint256 totalSupply = 99999999999999999932; 

uint256 preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
uint256 preDepositPrice = (10 ** 18 * 99999999999999999932) / 99999999999999999932;
uint256 preDepositPrice = 1e18;


// As we have only derivative Reth in the example, it owns all of the weight.
uint256 ethAmount = (msg.value * weight) / totalWeight;
uint256 ethAmount = (100e18 * 1000) / 1000;

// not applying the deposit fee in rocketPool
uint256 depositAmount = 93988463204618701706

uint derivativeReceivedEthValue = (derivative.ethPerDerivative(depositAmount) * depositAmount) / 10 ** 18;
uint derivativeReceivedEthValue = (1063960369075232250 * 93988463204618701706) / 10 ** 18;
uint derivativeReceivedEthValue = 99999999999999999932

totalStakeValueEth = 99999999999999999932;
 
uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
uint256 mintAmount = (99999999999999999932 * 10 ** 18) / 1e18;
uint256 mintAmount = 99999999999999999932
```

## Proof of Concept - Part 2 
From the first POC, we calculated the outcome of 200e18 staked into the Reth derivative. We got the totalSupply and the Reth balance the derivative holds. So we can move onto the main POC, where l can show the difference and how much less minted tokens the user gets.

```
totalSupply = 199999999999999999864;
Reth derivative balance = 187976926409237403412;
```

First l am going to show how much minted tokens the user is supposed to get without applying the issue occurring. And after that l will do the second one and apply the issue. So we can compare the outcomes and see how much less minted tokens the user gets.

Without the issue occurring, a user deposits 5e18 by calling the staking function. The user received (4999549277935239332) minted tokens of safEth.

```solidity
uint256 underlyingValue  = (derivatives[i].ethPerDerivative(derivatives[i].balance()) * derivatives[i].balance()) / 10 ** 18;
uint256 underlyingValue  = (1063960369075232250 * 187976926409237403412) / 10 ** 18;
uint256 underlyingValue  = 199999999999999999864;


uint256 totalSupply = 199999999999999999864; 

uint256 preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
uint256 preDepositPrice = (10 ** 18 * 199999999999999999864) / 199999999999999999864;
uint256 preDepositPrice = 1e18;

// As we have only derivative Reth in the example, it owns all of the weight.
uint256 ethAmount = (msg.value * weight) / totalWeight;
uint256 ethAmount = (5e18 * 1000) / 1000;

// not applying the deposit fee in rocketPool
uint256 depositAmount = 4698999533488942411

uint derivativeReceivedEthValue = (derivative.ethPerDerivative(depositAmount) * depositAmount) / 10 ** 18;
uint derivativeReceivedEthValue = (1063960369075232250 * 4698999533488942411) / 10 ** 18;
uint derivativeReceivedEthValue = 4999549277935239332

totalStakeValueEth = 4999549277935239332;
 
uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
uint256 mintAmount = (4999549277935239332 * 10 ** 18) / 1e18;
uint256 mintAmount = 4999549277935239332
```

Stats after the deposit without the issue:

```
totalSupply = 204999549277935239196
Reth derivative balance = 192675925942726345823;
```

This time we apply the issue occurring and as the first one a user deposits 5e18 by calling the staking function. The user receives (4973574036557377784) minted tokens of saEth

```solidity
uint256 underlyingValue  = (derivatives[i].ethPerDerivative(derivatives[i].balance()) * derivatives[i].balance()) / 10 ** 18;
// the function takes as account the pool price here which is overpriced.
uint256 underlyingValue  = (1069517062752670179 * 187976926409237403412) / 10 ** 18;
uint256 underlyingValue  = 201044530198482424206

uint256 totalSupply = 199999999999999999864;

uint256 preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
uint256 preDepositPrice = (10 ** 18 * 201044530198482424206) / 199999999999999999864;
uint256 preDepositPrice = 1005222650992412121;

// As we have only derivative Reth in the example, it owns all of the weight.
uint256 ethAmount = (msg.value * weight) / totalWeight;
uint256 ethAmount = (5e18 * 1000) / 1000;

// not applying the deposit fee in rocketPool
uint256 depositAmount = 4698999533488942411

// Here the function calculates based on the normal price, as the pool has free space and the user deposits only 5e18.
uint derivativeReceivedEthValue = (derivative.ethPerDerivative(depositAmount) * depositAmount) / 10 ** 18;
uint derivativeReceivedEthValue = (1063960369075232250 * 4698999533488942411) / 10 ** 18;
uint derivativeReceivedEthValue = 4999549277935239332

totalStakeValueEth = 4999549277935239332;
 
uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
uint256 mintAmount = (4999549277935239332 * 10 ** 18) / 1005222650992412121;
uint256 mintAmount = 4973574036557377784
```

Stats after the deposit with the issue:
```
totalSupply = 204973574036557377648;
Reth derivative balance = 192675925942726345823;
```

Difference between outcomes:

```
Without the issue based on 5e18 deposit, the user receives -        4999549277935239332 minted tokens
With the issue occurring based on 5e18 deposit, the user receives - 4973574036557377784 minted tokens
```

## Proof of Concept - Plus

So far we found that this issue leads to users receiving less minted shares, but let's go even further and see how much the user losses in terms of ETH. By unstaking the minted amount.

First we apply the stats without the issue occurring.

```
totalSupply = 204999549277935239196
Reth derivative balance = 192675925942726345823;
```

```solidity
uint256 derivativeAmount = (derivatives[i].balance() * _safEthAmount) / safEthTotalSupply;
uint256 derivativeAmount = (192675925942726345823 * 4999549277935239332) / 204999549277935239196;
uint256 derivativeAmount = 4698999533488942410;

// Eth value based on the current eth price
// Reth to Eth value - 4698999533488942410 => 4.999999999999999998 - 8766.85 usd

```

Second we apply the stats with the issue occurring.

```
totalSupply = 204973574036557377648;
Reth derivative balance = 192675925942726345823;
```
```solidity
uint256 derivativeAmount = (derivatives[i].balance() * _safEthAmount) / safEthTotalSupply;
uint256 derivativeAmount = (192675925942726345823 * 4973574036557377784) / 204973574036557377648;
uint256 derivativeAmount = 4675178189396666336;

// Eth value based on the current eth price
// Reth to Eth value - 4675178189396666336 => 4.974637740558436705 - 8722.41 usd

```

## Tools Used
Manual review

## Recommended Mitigation Steps
The problem occurs with calculating the underlyingValue in the staking function. The function "ethPerDerivative" is called with all of the Reth balance, which should not be the case here. Therefore the function calls "poolCanDeposit" in order to check if the pool has space for the Reth derivative balance (Basically the contract thinks that the Reth balance in the derivative will be deposited in the pool, which is not the case here). So even if the pool has space for the depositing amount by the user, the poolCanDeposit(_amount) will return false and the contract will get the poolPrice of the reth which is supposed to be used only for the swap in Uniswap. The contract process executing the staking function with the overpriced pool price and doesn't perform any swap, but deposits the user funds to the pool.

```solidity
underlyingValue +=
                (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
                    derivatives[i].balance()) /
                10 ** 18;
```
```solidity
function ethPerDerivative(uint256 _amount) public view returns (uint256) {
        if (poolCanDeposit(_amount))
            return
                RocketTokenRETHInterface(rethAddress()).getEthValue(10 ** 18);
        else return (poolPrice() * 10 ** 18) / (10 ** 18);
    }
```
```solidity
return
        rocketDepositPool.getBalance() + _amount <=
        rocketDAOProtocolSettingsDeposit.getMaximumDepositPoolSize() &&
        _amount >= rocketDAOProtocolSettingsDeposit.getMinimumDeposit();
```

l'd recommend creating a new function in the reth derivative contract. Which converts the msg.value to reth tokens and using it instead of the whole Reth balance the derivative holds.

```solidity
function rethValue(uint256 _amount) public view returns (uint256) {
      RocketTokenRETHInterface(rethAddress()).getRethValue(amount);
    }
```

Like this we check if the msg.value converted into reth tokens is below the maximumPoolDepositSize and greater than the minimum deposit.

```solidity
underlyingValue +=
                (derivatives[i].ethPerDerivative(derivatives[i].rethValue(msg.value)) *
                    derivatives[i].balance()) /
                10 ** 18;
```