## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-01

# [ProfilePicture subprotocol is immutably linked by `subprotocolName` to the CID protocol](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/286) 

# Lines of code

https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-pfp-protocol/src/ProfilePicture.sol#L59
https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-pfp-protocol/src/ProfilePicture.sol#L99


# Vulnerability details






## Impact
Besides having to re-register the protocol, it will also have to be redeployed.

## Proof of Concept
A protocol is registered by name in the `SubprotocolRegistry`. Quoting the [Canto Identity Protocol contest details](https://code4rena.com/contests/2023-01-canto-identity-protocol-contest): "In theory, someone can front-run a call to SubprotocolRegistry.register with the same name, causing the original call to fail. There is a registration fee (100 $NOTE) and the damage is very limited (original registration call fails, user can just re-register under a different name), so this attack is not deemed feasible.". However, the `ProfilePicture` subprotocol suffers from the additional issue of also having to be redeployed, making this exploit more severe.

In `ProfilePicture`, [`subprotocolName` is set on construction](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-pfp-protocol/src/ProfilePicture.sol#L59).
```solidity
constructor(address _cidNFT, string memory _subprotocolName) ERC721("Profile Picture", "PFP") {
    cidNFT = ICidNFT(_cidNFT);
    subprotocolName = _subprotocolName;
```
`subprotocolName` is used to [retrieve the `cidNFTID` to which the subprotocols NFT is added](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-pfp-protocol/src/ProfilePicture.sol#L99).
```solidity
uint256 cidNFTID = cidNFT.getPrimaryCIDNFT(subprotocolName, _pfpID);
IAddressRegistry addressRegistry = cidNFT.addressRegistry();
if (cidNFTID == 0 || addressRegistry.getAddress(cidNFTID) != ERC721(nftContract).ownerOf(nftID)) {
    nftContract = address(0);
    nftID = 0; // Strictly not needed because nftContract has to be always checked, but reset nevertheless to 0
}
```
This means that this subprotocol *must* to be registered under this name, or it will not work. So if `ProfilePicture` is deployed but someone else manages to register it in the `SubprotocolRegistry` before the owner (recipient of fees) of `ProfilePicture` (by frontrunning or otherwise), it cannot simply be re-registered under a different name, but would have to be redeployed with a new name under which it then can be registered.

## Tools Used
Code inspection

## Recommended Mitigation Steps
Since the subprotocol is meant to be registered with the CID protocol, its deployment and registration should be atomic. Or the name can be initialized after deployment and registration, and set by a (temporary) owner to the name it then has been registered with.