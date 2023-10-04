## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- H-05

# [V3Proxy swapTokensForExactETH does not send back to the caller the unused input tokens](https://github.com/code-423n4/2023-08-goodentry-findings/issues/64) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/helper/V3Proxy.sol#L174


# Vulnerability details

The `V3Proxy` `swapTokensForExactETH` function swaps an unspecified amount of a given ERC-20 for a specified amount of the native currency. After the swap happens, however, the difference between the amount taken from the caller (`amountInMax`) and the actual swapped amount (`amounts[0]`) is not given back to the caller and remains locked in the contract.

## Impact
Any user of the `swapTokensForExactETH` will always pay `amountInMax` for swaps even if part of it was not used for the swap. This part is lost, locked in the `V3Proxy` contract.

## Proof of Concept
- call `swapTokensForExactETH` with an excessively high `amountInMax`
- check that any extra input tokens are sent back - this check will fail
```Solidity
    function testV3ProxyKeepsTheChange() public {
        IQuoter q = IQuoter(0xb27308f9F90D607463bb33eA1BeBb41C27CE5AB6);
        ISwapRouter r = ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);

        V3Proxy v3proxy = new V3Proxy(r, q, 500);
        vm.label(address(v3proxy), "V3Proxy");

        address[] memory path = new address[](2);
        path[0] = address(USDC);
        path[1] = address(WETH);

        address[] memory path2 = new address[](2);
        path2[0] = address(WETH);
        path2[1] = address(USDC);


        // fund Alice
        vm.prank(tokenWhale);
        USDC.transfer(alice, 1870e6);

        // Alice initiates a swap
        uint256[] memory amounts; 
        uint256 balanceUsdcBefore = USDC.balanceOf(alice);
        uint256 balanceBefore = alice.balance;
        vm.startPrank(alice);
        USDC.approve(address(v3proxy), 1870e6);
        amounts = v3proxy.swapTokensForExactETH(1e18, 1870e6, path, alice, block.timestamp);

        // we check if the swap was done well
        require(amounts[0] < 1870e6);
        require(amounts[1] == 1e18);
        require(alice.balance == balanceBefore + amounts[1]); 
        // the following check fails, but would pass if swapTokensForExactETH
        // sent back the excess tokens
        require(USDC.balanceOf(alice) == balanceUsdcBefore - amounts[0], 
            "Unused input tokens were not sent back!");
    }
```

## Tools Used
Code review

## Recommended Mitigation Steps
Send back the excess tokens:
```diff
    function swapTokensForExactETH(uint amountOut, uint amountInMax, address[] calldata path, address to, uint deadline) payable external returns (uint[] memory amounts) {
        require(path.length == 2, "Direct swap only");
        require(path[1] == ROUTER.WETH9(), "Invalid path");
        ERC20 ogInAsset = ERC20(path[0]);
        ogInAsset.safeTransferFrom(msg.sender, address(this), amountInMax);
        ogInAsset.safeApprove(address(ROUTER), amountInMax);
        amounts = new uint[](2);
        amounts[0] = ROUTER.exactOutputSingle(ISwapRouter.ExactOutputSingleParams(path[0], path[1], feeTier, address(this), deadline, amountOut, amountInMax, 0));         
        amounts[1] = amountOut; 
        ogInAsset.safeApprove(address(ROUTER), 0);
        IWETH9 weth = IWETH9(ROUTER.WETH9());
        acceptPayable = true;
        weth.withdraw(amountOut);
        acceptPayable = false;
        payable(msg.sender).call{value: amountOut}("");
+        ogInAsset.safeTransfer(msg.sender, amountInMax - amounts[0]);
        emit Swap(msg.sender, path[0], path[1], amounts[0], amounts[1]); 
    }
```





## Assessed type

ERC20