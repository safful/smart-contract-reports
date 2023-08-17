## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- sponsor acknowledged
- selected for report
- M-09

# [Avoidable misconfiguration could lead to INVEscrow contract not minting xINV tokens](https://github.com/code-423n4/2022-10-inverse-findings/issues/379) 

# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L281-L283


# Vulnerability details

## Impact
If a user creates a market with the **INVEscrow** implementation as **escrowImplementation** and false as **callOnDepositCallback**, the deposits made by users in the escrow (through the market) would not mint **xINV** tokens for them. As **callOnDepositCallback** is an immutable variable set in the constructor, this mistake would make the market a failure and the user should deploy a new one (even worse, if the error is detected after any user has deposited funds, some sort of migration of funds should be needed).

## Proof of Concept
Both **escrowImplementation** and **callOnDepositCallback** are immutable:
```javascript
...
address public immutable escrowImplementation;
...
bool immutable callOnDepositCallback;
...
```
and its value is set at creation:
```javascript
constructor (
        address _gov,
        address _lender,
        address _pauseGuardian,
        address _escrowImplementation,
        IDolaBorrowingRights _dbr,
        IERC20 _collateral,
        IOracle _oracle,
        uint _collateralFactorBps,
        uint _replenishmentIncentiveBps,
        uint _liquidationIncentiveBps,
        bool _callOnDepositCallback
    ) {
	...
	escrowImplementation = _escrowImplementation;
	...
	callOnDepositCallback = _callOnDepositCallback;
	...
 }
```
When the user deposits collateral, if **callOnDepositCallback** is true, there is a call to the escrow's **onDeposit** callback:
```javascript
function deposit(address user, uint amount) public {
	...
	if(callOnDepositCallback) {
		escrow.onDeposit();
	}
	emit Deposit(user, amount);
}
```
This is **INVEscrow**'s onDeposit function:
```javascript
function onDeposit() public {
	uint invBalance = token.balanceOf(address(this));
	if(invBalance > 0) {
		xINV.mint(invBalance); // we do not check return value because we don't want errors to block this call
	}
}
```
The thing is if **callOnDepositCallback** is false, this function is never called and the user does not turn his/her collateral (**INV**) into **xINV**.

## Tools Used
Manual analysis.

## Recommended Mitigation Steps
Either make **callOnDepositCallback** a configurable parameter in Market.sol or always call the **onDeposit** callback (just get rid of the **callOnDepositCallback** variable) and leave it empty in case there's no extra functionality that needs to be executed for that escrow. In the case that the same escrow has to execute the callback for some markets and not for others, this solution would imply that there should be two escrows, one with the callback to be executed and another with the callback empty.