## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- selected for report
- M-05

# [Inconsistency in fees](https://github.com/code-423n4/2022-12-escher-findings/issues/274) 

# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L109


# Vulnerability details

## Impact
If the seller cancel before the sale startTime then all funds should be moved to saleReceiver without any deduction (Assuming seller has sent some ETH accidentally before sell start). But in `FixedPrice.sol` contract, fees is deducted even when seller cancel before the sale startTime which could lead to loss of funds

## Proof of Concept
1. A new sale is started
2. Seller selfdestructed one of his personal contract and by mistake gave this sale contract as receiver. This forcefully sends the remaining 20 ETH to the  `FixedPrice.sol` contract 
3. Seller realizes his mistake and tries to cancel the sale as sale as not yet started using the `cancel` function

```
function cancel() external onlyOwner {
        require(block.timestamp < sale.startTime, "TOO LATE");
        _end(sale);
    }
```

4. This internally calls the _end function

```
function _end(Sale memory _sale) internal {
        emit End(_sale);
        ISaleFactory(factory).feeReceiver().transfer(address(this).balance / 20);
        selfdestruct(_sale.saleReceiver);
    }
```

5. The `_end` function deducts fees of 20/20=1 ETH even though seller has cancelled before sell start.

## Recommended Mitigation Steps
Revise the `cancel` function

```
function cancel() external onlyOwner {
        require(block.timestamp < sale.startTime, "TOO LATE");
        emit End(_sale);
        selfdestruct(_sale.saleReceiver);
    }
```