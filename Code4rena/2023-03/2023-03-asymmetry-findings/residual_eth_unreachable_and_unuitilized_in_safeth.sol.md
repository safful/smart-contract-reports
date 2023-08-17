## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- edited-by-warden
- M-11

# [Residual ETH unreachable and unuitilized in SafEth.sol](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/152) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L246
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L124-L127
https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L138-L155


# Vulnerability details

## Impact
Unlike the other three contracts in scope, SafEth.sol does not have a measure in place to utilize the residual ETH, be it:

- accidentally received, 
- zero ETH output from `unstake()` arising from recipient non-contract existence, or
- ETH sent in via `stake()` fails to deposit into a derivative due to uninitialized derivatives and weights, as I have explained in a separate submission. 

In the derivative contracts, the above issue is wiped clean via `address(this).balance` every time `withdraw()` is predominantly invoked in [`unstake()`](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L118) of SafEth.sol:

[File: WstEth.sol#L63-L66](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/WstEth.sol#L63-L66)
[File: SfrxEth.sol#L84-L87](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/SfrxEth.sol#L84-L87)
[File: Reth.sol#L110-L114](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L110-L114)

```soildity
        (bool sent, ) = address(msg.sender).call{value: address(this).balance}(
            ""
        );
        require(sent, "Failed to send Ether");
```
## Proof of Concept
As in SafETH.sol, `ethAmountBefore` and `ethAmountAfter` are used to determine `ethAmountToWithdraw` or `ethAmountToRebalance` respectively in `untake()` and `rebalanceToWeights()`:

[File: SafEth.sol](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol)

```solidity
111:        uint256 ethAmountBefore = address(this).balance;

121:        uint256 ethAmountAfter = address(this).balance;

122:        uint256 ethAmountToWithdraw = ethAmountAfter - ethAmountBefore;

139:        uint256 ethAmountBefore = address(this).balance;

144:        uint256 ethAmountAfter = address(this).balance;

145:        uint256 ethAmountToRebalance = ethAmountAfter - ethAmountBefore;
```
## Tools Used
Manual

## Recommended Mitigation Steps
Consider having `rebalanceToWeights()` refactored as follows:

[File: SafEth.sol#L138-L155](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L138-L155)

```diff
    function rebalanceToWeights() external onlyOwner {
-        uint256 ethAmountBefore = address(this).balance;
        for (uint i = 0; i < derivativeCount; i++) {
            if (derivatives[i].balance() > 0)
                derivatives[i].withdraw(derivatives[i].balance());
        }
-        uint256 ethAmountAfter = address(this).balance;
-        uint256 ethAmountToRebalance = ethAmountAfter - ethAmountBefore;
+        uint256 ethAmountToRebalance = address(this).balance;

        for (uint i = 0; i < derivativeCount; i++) {
            if (weights[i] == 0 || ethAmountToRebalance == 0) continue;
            uint256 ethAmount = (ethAmountToRebalance * weights[i]) /
                totalWeight;
            // Price will change due to slippage
            derivatives[i].deposit{value: ethAmount}();
        }
        emit Rebalanced();
    }
```
This will at least have the residual ETH harnessed and distributed to all existing stakers whenever `rebalanceToWeights()` is called.