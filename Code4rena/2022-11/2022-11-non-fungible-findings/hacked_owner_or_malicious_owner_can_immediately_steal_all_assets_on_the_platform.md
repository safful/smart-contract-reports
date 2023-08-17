## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-02

# [Hacked owner or malicious owner can immediately steal all assets on the platform](https://github.com/code-423n4/2022-11-non-fungible-findings/issues/179) 

# Lines of code

https://github.com/code-423n4/2022-11-non-fungible/blob/323b7cbf607425dd81da96c0777c8b12e800305d/contracts/Exchange.sol#L639
https://github.com/code-423n4/2022-11-non-fungible/blob/323b7cbf607425dd81da96c0777c8b12e800305d/contracts/Exchange.sol#L30


# Vulnerability details

## Description

In Non-Fungible's security model, users approve their ERC20 / ERC721 / ERC1155 tokens to the ExecutionDelegate contract, which accepts transfer requests from Exchange. 

The requests are made here:
```
function _transferTo(
    address paymentToken,
    address from,
    address to,
    uint256 amount
) internal {
    if (amount == 0) {
        return;
    }
    if (paymentToken == address(0)) {
        /* Transfer funds in ETH. */
        require(to != address(0), "Transfer to zero address");
        (bool success,) = payable(to).call{value: amount}("");
        require(success, "ETH transfer failed");
    } else if (paymentToken == POOL) {
        /* Transfer Pool funds. */
        bool success = IPool(POOL).transferFrom(from, to, amount);
        require(success, "Pool transfer failed");
    } else if (paymentToken == WETH) {
        /* Transfer funds in WETH. */
        executionDelegate.transferERC20(WETH, from, to, amount);
    } else {
        revert("Invalid payment token");
    }
}
```

```
function _executeTokenTransfer(
    address collection,
    address from,
    address to,
    uint256 tokenId,
    uint256 amount,
    AssetType assetType
) internal {
    /* Call execution delegate. */
    if (assetType == AssetType.ERC721) {
        executionDelegate.transferERC721(collection, from, to, tokenId);
    } else if (assetType == AssetType.ERC1155) {
        executionDelegate.transferERC1155(collection, from, to, tokenId, amount);
    }
}
```

The issue is that there is a significant centralization risk trusting Exchange.sol contract to behave well, because it is an immediately upgradeable ERC1967Proxy. All it takes for a malicious owner or hacked owner to upgrade to the following contract:

```
function _stealTokens(
    address token,
    address from,
    address to,
    uint256 tokenId,
    uint256 amount,
    AssetType assetType
) external onlyOwner {
    /* Call execution delegate. */
    if (assetType == AssetType.ERC721) {
        executionDelegate.transferERC721(token, from, to, tokenId);
    } else if (assetType == AssetType.ERC1155) {
        executionDelegate.transferERC1155(token, from, to, tokenId, amount);
    } else if (assetType == AssetType.ERC20) {
        executionDelegate.transferERC20(token, from, to, amount);
}
```

At this point hacker or owner can steal all the assets approved to Non-Fungible. 

## Impact

Hacked owner or malicious owner can immediately steal all assets on the platform

## Tools Used

Manual audit

## Recommended Mitigation Steps

Exchange contract proxy should implement a timelock, to give users enough time to withdraw their approvals before some malicious action becomes possible.

## Judging note

The status quo regarding significant centralization vectors has always been to award M severity, in order to warn users of the protocol of this category of risks. See [here](https://gist.github.com/GalloDaSballo/881e7a45ac14481519fb88f34fdb8837) for list of centralization issues previously judged.
