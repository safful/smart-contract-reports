## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor acknowledged
- H-07

# [Anyone who uses same adapter have ability to pause it](https://github.com/code-423n4/2023-01-popcorn-findings/issues/292) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L605-L615
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L575


# Vulnerability details

## Impact
Anyone who uses same adapter have ability to pause it. As result you have ability to pause any vault by creating your vault with same adapter.

When user creates vault he has ability to deploy new adapter or [reuse already created adapter](https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L103-L104).

VaultController gives ability to pause adapter.
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L605-L615
```solidity
  function pauseAdapters(address[] calldata vaults) external {
    uint8 len = uint8(vaults.length);
    for (uint256 i = 0; i < len; i++) {
      _verifyCreatorOrOwner(vaults[i]);
      (bool success, bytes memory returnData) = adminProxy.execute(
        IVault(vaults[i]).adapter(),
        abi.encodeWithSelector(IPausable.pause.selector)
      );
      if (!success) revert UnderlyingError(returnData);
    }
  }
```

As you can see `_verifyCreatorOrOwner` is used to determine if msg.sender can pause adapter.
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L667-L670
```solidity
  function _verifyCreatorOrOwner(address vault) internal returns (VaultMetadata memory metadata) {
    metadata = vaultRegistry.getVault(vault);
    if (msg.sender != metadata.creator || msg.sender != owner) revert NotSubmitterNorOwner(msg.sender);
  }
```
So in case if you are creator of vault that uses adaptor that you want to pause, then you are able to pause it.

This is how it can be used in order to stop the vault, i don't like.
1.Someone created vault that uses adapterA.
2.Attacker creates own vault and set adapterA as well.
3.Now attacker is able to pause adapterA and as result it's not possible to deposit anymore. Also vault is not earning fees now, as pausing [withdraws all from strategy](https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L575).
4.And it can pause it as many times as he wants(in case if someone else will try to unpause it).

So this attack allows to stop all vaults that use same adapter from earning yields.
## Tools Used
VsCode
## Recommended Mitigation Steps
I think that it's better to create a clone of adapter for the vault, so each vault has separate adaptor.