## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-01

# [[H1]  Low data feed frequency from Tellor makes you protocol vulnerable to flash loan attacks](https://github.com/code-423n4/2023-02-ethos-findings/issues/772) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/Dependencies/TellorCaller.sol#L44


# Vulnerability details

# [H1]  Low data feed frequency from Tellor makes you protocol vulnerable to flash loan attacks

 

## Impact 

​          An attacker can stale Tellor Oracle for several hours cheaply and perform a flash loan attack to profit.      

## PoC

To explain this issue I will first compare Chainlink to Tellor.

Most ERC-20 tokens are in general much more volatile than ETH and BTC.     In Chainlink,  there are triggers of 0,5% for BTC and ETH and 1% for other assets.     This is to ensure that you are cutting error by those values.

Tellor, on the other hand, is an *optimistic oracle*.    Stakers use the oracle system to put data on chain  `submitValue(..)` that are directly shown in the oracle.     The security lies the fact that data consumers should  wait some  **dispute windows** in order to give time to others to dispute data and remove incorrect or malicious data.

This is what happened in a Liquity bug found last year, they were reading instant data.   [2] 

Being explained this and back to your code you have essentially two bugs

 

## First bug :  Default disputetime  = 20 minutes

In `TellorCaller.sol` you have the following statement

   `(bytes memory data, uint256 timestamp) = getDataBefore(_queryId, block.timestamp - 20 minutes)`

Maybe you are using 20 minutes because this is the default value in Tellor documentation.    However, in Liquity they are using 15 minutes for ETH because they say that have been made an analysis of  ETH  volatility behaviour.[2]

Basically there is a tradeoff between the volatility of an asset and the dispute time.    More time is safer to have time to dipute but more likely to read a so old value.     Less dispute time you have less error but no time to dispute can put you at risk of reading a manipulated value.

​     In your case, you are using more volatile assets so in theory, if Liquity analysis is correct, you should be using less time for ERC20 assets.  

​	Of course this requires a deeper analysis but I am not doing it because the second bug makes this unnecessary as it has a higher impact.



## Second bug :   Data feed frequency in Tellor is very low so it is cheap to break




 I will briefly explain some Tellor security designs.

Tellor bases his security design in an exponential cost to dispute.       They have a several-round voting to dispute  a single value but we are interested in the *Cost of Stalling* (CoS) the System.   

To stale the system we need  to dipute every single value for a given period, for a given asset.

According to whitepaper cost starts at  `baseFee`  and increase with the following formula



​                           𝑑𝑖𝑠𝑝𝑢𝑡𝑒𝐹𝑒𝑒𝑖𝑑,𝑡,𝑟>1 = 𝑑𝑖𝑠𝑝𝑢𝑡𝑒𝐹𝑒𝑒𝑖 × 2 𝑑𝑖𝑠𝑝𝑢𝑡𝑒𝑅𝑜𝑢𝑛𝑑𝑠𝑖𝑑,𝑡−1  

Where 

​                                𝑑𝑖𝑠𝑝𝑢𝑡𝑒𝐹𝑒𝑒𝑖 is the initial dispute fee (`baseFee`) 

​                               𝑑𝑖𝑠𝑝𝑢𝑡𝑒𝑅𝑜𝑢𝑛𝑑𝑠𝑖𝑑 is the number of disputes open for a specific ID



In Ethereum, there is a block every 15 seconds so stalling the system for 8 minutes (32 blocks) will cost and attacker around   2^32 * 10 TRB  = 687 Billons of dollars! ... (10TRB = 160USD).   Not bad at all.    

Tellor team has similar values in different docs around internet. 

​       However, this is nice if we **always assume that one data is sent every block (a ideal system)**.

and here is where the real nightmare comes.    Current frequency for  data in Tellor is very low,  **that you are reading data once an hour or less!!**.      

Even worse for Optimism and Polygon `basedisputeFee` is only 1TRB .  ( vs 10 TRB in Ethereum)  

  This design was thought considering that these chains are faster  so if you data is sent every block then breaking the system would be prohibitively expensive.    Again, security depends on the frequency of data

In our real word, Tellor is producing data in Optimism as low as in Ethereum so in the end it is 10 times cheaper to break.



## Cost to Stale ETH/USD pair in Optimism 

Lets calculate the CoS ETH/USD pair for 4 hours

Watching this Tellor contracts we can get that baseFee is 1TRB

https://optimistic.etherscan.io/address/0x46038969d7dc0b17bc72137d07b4ede43859da45#readContract       ==> `getDataFee()` = 1TRB        



Now, read data in a four hour range using the function.

 

`getMultipleValuesBefore()`



Parameters passed

`queryId =  0x83a7f3d48786ac2667503a61e8c415438ed2922eb86a2906e4ee66d9a2ce4992` (ID for asking ETH/USD pair value)

 `timestamp =  1678147200`                    (7 march 2023 at 0:00)

`_max age = 14400`                                    (4 hours earlier = 14400 seconds)

`_maxCount = 1000`                                    (doesn't really matter)



We get only 4 values with the following timestamps [1678135628,1678139237,1678142833,1678146437]



 The *CoS* Tellor ETH/USD pair for these four hours  would have been

 

1TRB + 2TRB + 4TRB + 8 TRB  = 15TRB  

15TRB * 16USD = 240 USD



This means that for a little 240 bucks you can Stale 4 hours the Oracle which is not acceptable at all as I will show you an attacking scenario.  

​       Ethereum has moved only 1%, not so critical this time, however volatille ERC20 used as a collateral can have much bigger changes.    

You can query more data with different timestamps



### Attacking Scenario:  Flash Loan to profit



Steps:

  1.   Write a contract that checks if Chainlink is working

  2.   Meanwhile a second script that tracks values for all your collateral assets

  3. When Chainlink is broken do

  4.  Stall all your collateral data from Tellor using `dispute`.   (10 collateral for $2500)

  5.  Suppose that you see an increase of 10% one of the colaterral  call it ABC and ETH not moving so much

  6. Ask for a Flash Loan ETH in Uniswap

  7. Mint LUSD for ETH at Ethos

  8. Redeem LUSD for collateral ABC.    You get a 10% discount because Oracle is staled 4 hours ago

9. Exchange LUSD for ETHEREUM in Uniswap.

10. Return ETH to the flash loan plus interest

11.  Enjoy!

​    

Note that this attack can be improved if you perform the loan on a falling collateral to mint more LUSD.    

The only level of protection you have is the fact that Chainlink is working,   in Liquity it is more difficult because it should be an important change of ETH value in those 4 hours.



## Recommended

 There are two solutions in my opinion 




*Solution 1 :   Tellor tip mechanism*

Tellor whitepaper:  

*"Parties who wish to build reporter support for their query should follow best practices when selecting data for their query (publish data specification on github, promote/ educate in the community), but will also need to tip a higher amount to incentivize activity"*

​       This means that in order to use data safely you need to pay to be sure that frequency is  secure taking into account the impact of the volatility and the time to dispute.

   I din't mention earlier but the cost to dispute is exponential but capped by the staking amount of the reporter, so no real billons of dollars in fact.

​           Here is the documentation how you can fund for a feed https://docs.tellor.io/tellor/getting-data/funding-a-feed  


*Solution 2. Do not use Tellor*

​         Unlike Liquity, you need to fund several feeds so I don't know if this is cost effective but you have options to fund fees only when Chainlink is broken but you need to investigate on that.

​           In any case you have a function to set the oracle that I am reporting as medium so no sure if you need to use two oracles. 

   

### References.

  1. Tellor White Paper   https://tellor.io/whitepaper/
  2. Liquity Tellor issue 2022   https://www.liquity.org/blog/tellor-issue-and-fix

