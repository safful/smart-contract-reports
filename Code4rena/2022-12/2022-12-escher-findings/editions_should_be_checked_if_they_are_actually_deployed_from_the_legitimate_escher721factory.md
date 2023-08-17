## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- M-04

# [Editions should be checked if they are actually deployed from the legitimate Escher721Factory](https://github.com/code-423n4/2022-12-escher-findings/issues/176) 

# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/FixedPriceFactory.sol#L29
https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/LPDAFactory.sol#L29
https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/minters/OpenEditionFactory.sol#L29


# Vulnerability details

## Impact

For all kinds of sales, the creators create new sales contracts with arbitrary sale data, and the edition is not properly checked.
Malicious creators can create fake contracts that implemented `IEscher721` and fake buyers to get free earnings.

## Proof of Concept

Sales contracts can be created by any creator and the sale data is filled with the one provided by the creator.
The protocol does not validate the `sale.edition` provided by the creator and malicious creators can effectively use their fake contract address that implemented `IEscher721`.
In the worst case, buyers will not get anything after their payments.

```solidity
// FixedPriceFactory.sol#L29
function createFixedSale(FixedPrice.Sale calldata _sale) external returns (address clone) {
  require(IEscher721(_sale.edition).hasRole(bytes32(0x00), msg.sender), "NOT AUTHORIZED"); //@audit need to check edition is actually from the legimate escher721 factory
  require(_sale.startTime >= block.timestamp, "START TIME IN PAST");
  require(_sale.finalId > _sale.currentId, "FINAL ID BEFORE CURRENT");

  clone = implementation.clone();
  FixedPrice(clone).initialize(_sale);

  emit NewFixedPriceContract(msg.sender, _sale.edition, clone, _sale);
}

```

Malicious creators can use a fake contract as an edition to steal funds from users.

## Tool used

Manual Review

## Recommended Mitigation Steps

Track all the deployed `Escher721` contracts in the `Escher721Factory.sol` and validate the `sale.edition` before creating sales contracts.