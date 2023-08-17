## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor disputed
- H-01

# [selfdestruct may cause the funds to be lost](https://github.com/code-423n4/2022-12-escher-findings/issues/296) 

# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/FixedPrice.sol#L110
https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/OpenEdition.sol#L122


# Vulnerability details

## Impact

After the contract is destroyed, the subsequent execution of the contract's #buy() will always success, the msg.value will be locked in this address

## Proof of Concept

When FixedPrice.sol and OpenEdition.sol are finished, selfdestruct() will be executed to destroy the contract.
But there is a problem with this:
Suppose when alice and bob execute the purchase transaction at the same time, the transactions is in the memory pool (or alice executes the transaction, but bob is still operating the purchase in the UI, the UI does not know that the contract has been destroyed)

If alice meets the finalId, the contract is destroyed after her transaction ends
Note: "When there is no code at the address, the transaction will succeed, and the msg.value will be stored in the contract. Although no code is executed."
after that  bob's transaction be execute 
This way the msg.value passed by bob is lost and locked forever in the address of this empty code

Suggestion: don't use selfdestruct, use modified state to represent that the contract has completed the sale

## Tools Used

## Recommended Mitigation Steps

```solidity
contract FixedPrice is Initializable, OwnableUpgradeable, ISale {
...
    function _end(Sale memory _sale) internal {
        emit End(_sale);
        ISaleFactory(factory).feeReceiver().transfer(address(this).balance / 20);
-       selfdestruct(_sale.saleReceiver); 
+       sale.finalId = sale.currentId
+       sale.saleReceiver.transfer(address(this).balance);     

    }
```