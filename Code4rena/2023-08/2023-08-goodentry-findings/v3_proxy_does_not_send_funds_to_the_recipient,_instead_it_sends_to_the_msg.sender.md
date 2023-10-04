## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-01

# [V3 Proxy does not send funds to the recipient, instead it sends to the msg.sender](https://github.com/code-423n4/2023-08-goodentry-findings/issues/463) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L112-L194


# Vulnerability details

 ## Impact
The functions above can be used to swap tokens, however the swaps are not sent to the provided address. Instead they are sent to the msg.sender. 
This could cause issues if the user has been blacklisted on a token. Or if the user has a compromised signature/allowance of the target token and they attempt to swap to the token, the user looses all value even though they provided an destination adress 

## Proof of Concept
```solidity
//@audit-H does not send tokens to the required address

    function swapExactTokensForTokens(uint amountIn, uint amountOutMin, address[] calldata path, address to, uint deadline) external returns (uint[] memory amounts) {
        require(path.length == 2, "Direct swap only");
        ERC20 ogInAsset = ERC20(path[0]);
        ogInAsset.safeTransferFrom(msg.sender, address(this), amountIn);
        ogInAsset.safeApprove(address(ROUTER), amountIn);
        amounts = new uint[](2);
        amounts[0] = amountIn;         
//@audit-issue it should be the to address not msg.sender
        amounts[1] = ROUTER.exactInputSingle(ISwapRouter.ExactInputSingleParams(path[0], path[1], feeTier, msg.sender, deadline, amountIn, amountOutMin, 0));
        ogInAsset.safeApprove(address(ROUTER), 0);
        emit Swap(msg.sender, path[0], path[1], amounts[0], amounts[1]); 
    }
```
 Here is one of the many functions with this issue, As we can see after the swap is completed, tokens are sent back to the msg.sender from the router not to the ```to``` address

## Tools Used
Manual Review. 
Uniswap Router: https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol
## Recommended Mitigation Steps
The uniswap router supports inputting of a destination address. Hence the router should be called with the to address not the msg.sender. 
Else remove the ```address to``` from the parameter list





## Assessed type

Uniswap