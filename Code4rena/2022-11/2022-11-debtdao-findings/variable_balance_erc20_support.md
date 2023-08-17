## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- sponsor disputed
- selected for report
- M-09

# [Variable balance ERC20 support](https://github.com/code-423n4/2022-11-debtdao-findings/issues/367) 

# Lines of code

https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/utils/EscrowLib.sol#L94-L96
https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/utils/EscrowLib.sol#L75-L79
https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/credit/LineOfCredit.sol#L273-L280
https://github.com/debtdao/Line-of-Credit/blob/e8aa08b44f6132a5ed901f8daa231700c5afeb3a/contracts/modules/credit/LineOfCredit.sol#L487-L493


# Vulnerability details

## Impact

Some ERC20 may be tricky for the balance. Such as:
- fee on transfer (STA, USDT also has this mode)
- rebasing (aToken from AAVE)
- variable balance (stETH, balance could go up and down)

For these tokens, the balance can change over time, even without `transfer()/transferFrom()`. But current accounting stores the spot balance of the asset. 

The impacts include:
- the calculation of collateral value could be inaccurate
- protocol could lose fund due to the deposit/repay amount might be less than the actual transferred amount after fee
- the amount user withdraw collateral when `_close()` will be inaccurate
    - some users could lose fund due to under value
    - some fund could be locked due to the balance inflation
    - some fund might be locked due to the balance deflation


## Proof of Concept

The spot new deposit amount is stored in the mapping `self.deposited[token].amount` and `credit.deposit`, and later used to calculate the collateral value and withdraw amount.
```solidity
// Line-of-Credit/contracts/utils/EscrowLib.sol
    function addCollateral(EscrowState storage self, address oracle, uint256 amount, address token) {
        // ...
        LineLib.receiveTokenOrETH(token, msg.sender, amount);

        self.deposited[token].amount += amount;
        // ...
    }

    function _getCollateralValue(EscrowState storage self, address oracle) public returns (uint256) {
            // ...
            d = self.deposited[token];
                // ...
                collateralValue += CreditLib.calculateValue(
                  o.getLatestAnswer(d.asset),
                  deposit,
                  d.assetDecimals
                );
            // ...
    }

// Line-of-Credit/contracts/modules/credit/LineOfCredit.sol
    function increaseCredit(bytes32 id, uint256 amount) {
        // ...
        Credit memory credit = credits[id];
        credit = _accrue(credit, id);

        credit.deposit += amount;
        
        credits[id] = credit;

        LineLib.receiveTokenOrETH(credit.token, credit.lender, amount);

        // ...
    }

    function _close(Credit memory credit, bytes32 id) internal virtual returns (bool) {
        // ...
        if (credit.deposit + credit.interestRepaid > 0) {
            LineLib.sendOutTokenOrETH(
                credit.token,
                credit.lender,
                credit.deposit + credit.interestRepaid
            );
        }
```

However, if the balance changed later, the returned collateral value will be inaccurate. And the amount used when withdraw collateral in `_close()` is also wrong.


## Tools Used
Manual analysis.

## Recommended Mitigation Steps

- checking the before and after balance of token transfer
- recording the relative shares of each user instead of specific amount
- if necessary, call `ERC20(token).balanceOf()` to confirm the balance
- disallow such kind of tokens
