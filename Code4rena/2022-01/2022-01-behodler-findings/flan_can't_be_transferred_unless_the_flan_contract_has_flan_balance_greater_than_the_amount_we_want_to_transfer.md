## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- resolved
- sponsor confirmed

# [flan can't be transferred unless the flan contract has flan balance greater than the amount we want to transfer](https://github.com/code-423n4/2022-01-behodler-findings/issues/160) 

# Handle

CertoraInc


# Vulnerability details

## Flan.sol (`safeTransfer()` function)
The flan contract must have balance (and must have more flan then we want to transfer) in order to allow flan transfers. If it doesn't have any balance, the safeTransfer, which is the only way to transfer flan, will call `_transfer()` function with `amount = 0`. It should check `address(msg.sender)`'s balance instead of `address(this)`'s balance.

```sol
function safeTransfer(address _to, uint256 _amount) external {
       uint256 flanBal = balanceOf(address(this)); // the problem is in this line
       uint256 flanToTransfer = _amount > flanBal ? flanBal : _amount;
       _transfer(_msgSender(), _to, flanToTransfer);
   }
```

