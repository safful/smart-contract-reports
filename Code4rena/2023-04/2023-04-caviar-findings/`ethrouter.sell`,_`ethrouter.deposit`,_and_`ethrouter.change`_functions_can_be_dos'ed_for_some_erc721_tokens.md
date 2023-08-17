## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-05

# [`EthRouter.sell`, `EthRouter.deposit`, and `EthRouter.change` functions can be DOS'ed for some ERC721 tokens](https://github.com/code-423n4/2023-04-caviar-findings/issues/776) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/EthRouter.sol#L152-L209
https://github.com/code-423n4/2023-04-caviar/blob/main/src/EthRouter.sol#L219-L248
https://github.com/code-423n4/2023-04-caviar/blob/main/src/EthRouter.sol#L254-L293
https://etherscan.io/address/0xf5b0a3efb8e8e4c201e2a935f110eaaf3ffecb8d#code#L672


# Vulnerability details

## Impact
The following `EthRouter.sell`, `EthRouter.deposit`, and `EthRouter.change` functions call the corresponding ERC721 tokens' `setApprovalForAll` functions.

https://github.com/code-423n4/2023-04-caviar/blob/main/src/EthRouter.sol#L152-L209
```solidity
    function sell(Sell[] calldata sells, uint256 minOutputAmount, uint256 deadline, bool payRoyalties) public {
        ...
        // loop through and execute the sells
        for (uint256 i = 0; i < sells.length; i++) {
            ...
            // approve the pair to transfer NFTs from the router
            ERC721(sells[i].nft).setApprovalForAll(sells[i].pool, true);
            ...
        }
        ...
    }
```

https://github.com/code-423n4/2023-04-caviar/blob/main/src/EthRouter.sol#L219-L248
```solidity
    function deposit(
        address payable privatePool,
        address nft,
        uint256[] calldata tokenIds,
        uint256 minPrice,
        uint256 maxPrice,
        uint256 deadline
    ) public payable {
        ...
        // approve pair to transfer NFTs from router
        ERC721(nft).setApprovalForAll(privatePool, true);
        ...
    }
```

https://github.com/code-423n4/2023-04-caviar/blob/main/src/EthRouter.sol#L254-L293
```solidity
    function change(Change[] calldata changes, uint256 deadline) public payable {
        ...
        // loop through and execute the changes
        for (uint256 i = 0; i < changes.length; i++) {
            Change memory _change = changes[i];
            ...
            // approve pair to transfer NFTs from router
            ERC721(_change.nft).setApprovalForAll(_change.pool, true);
            ...
        }
        ...
    }
```

For ERC721 tokens like Axie, which its `setApprovalForAll` function is shown below, calling their `setApprovalForAll` functions with the same `msg.sender`-`_operator`-`_approved` combination would revert because of requirements like `require(_tokenOperator[msg.sender][_operator] != _approved)`. For these ERC721 tokens, calling the `EthRouter.sell`, `EthRouter.deposit`, and `EthRouter.change` functions for the first time, which call such tokens' `setApprovalForAll` functions for the first time, can succeed; however, calling the `EthRouter.sell`, `EthRouter.deposit`, and `EthRouter.change` functions again, which call such tokens' `setApprovalForAll` functions with the same pool as `_operator` and `true` as `_approved` again, will revert. In this case, the `EthRouter.sell`, `EthRouter.deposit`, and `EthRouter.change` functions are DOS'ed for such ERC721 tokens.

https://etherscan.io/address/0xf5b0a3efb8e8e4c201e2a935f110eaaf3ffecb8d#code#L672
```solidity
  function setApprovalForAll(address _operator, bool _approved) external whenNotPaused {
    require(_tokenOperator[msg.sender][_operator] != _approved);
    _tokenOperator[msg.sender][_operator] = _approved;
    ApprovalForAll(msg.sender, _operator, _approved);
  }
```

## Proof of Concept
The following steps can occur for the described scenario.
1. Alice calls the `EthRouter.sell` function to sell 1 Axie NFT to a private pool, which succeeds.
2. Alice calls the `EthRouter.sell` function again to sell another Axie NFT to the same private pool. However, this function call's execution of `ERC721(sells[i].nft).setApprovalForAll(sells[i].pool, true)` reverts because Axie's `require(_tokenOperator[msg.sender][_operator] != _approved)` reverts.
3. Bob tries to repeat Step 2 but his `EthRouter.sell` function call also reverts.
4. Hence, the `EthRouter.sell` function is DOS'ed for selling any Axie NFTs to the same private pool for any users.

## Tools Used
VSCode

## Recommended Mitigation Steps
The `EthRouter.sell`, `EthRouter.deposit`, and `EthRouter.change` functions can be respectively updated to check if the `EthRouter` contract has approved the corresponding pool to spend any of the corresponding ERC721 tokens received by itself. If not, the corresponding ERC721's `setApprovalForAll` function can be called; otherwise, the corresponding ERC721's `setApprovalForAll` function should not be called.