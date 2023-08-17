## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-05

# [If the underlying NFT is burned, getPFP may returns incorrect results](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/209) 

# Lines of code

https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-pfp-protocol/src/ProfilePicture.sol#L94-L105


# Vulnerability details

## Impact
ProfilePicture.getPFP will return information about the underlying NFT and when `addressRegistry.getAddress(cidNFTID) ! = ERC721(nftContract).ownerOf(nftID)`, it will return 0. 
But if the underlying NFT is burned, getPFP may return incorrect information
```solidity
    function getPFP(uint256 _pfpID) public view returns (address nftContract, uint256 nftID) {
        if (_ownerOf[_pfpID] == address(0)) revert TokenNotMinted(_pfpID);
        ProfilePictureData storage pictureData = pfp[_pfpID];
        nftContract = pictureData.nftContract;
        nftID = pictureData.nftID;
        uint256 cidNFTID = cidNFT.getPrimaryCIDNFT(subprotocolName, _pfpID);
        IAddressRegistry addressRegistry = cidNFT.addressRegistry();
        if (cidNFTID == 0 || addressRegistry.getAddress(cidNFTID) != ERC721(nftContract).ownerOf(nftID)) {
            nftContract = address(0);
            nftID = 0; // Strictly not needed because nftContract has to be always checked, but reset nevertheless to 0
        }
    }
```
Consider the following scenario.
1. alice mint a ProfilePicture NFT (pfp NFT) with the underlying NFT, and later mint a CidNFT by transferring the pfp NFT to the CidNFT contract. 
**noting that alice does not call AddressRegistry.register to register the CidNFT.** 
Since getAddress returns 0, ProfilePicture.getPFP will also return 0

```solidity
    function getAddress(uint256 _cidNFTID) external view returns (address user) {
        user = ERC721(cidNFT).ownerOf(_cidNFTID);
        if (_cidNFTID != cidNFTs[user]) {
            // User owns CID NFT, but has not registered it
            user = address(0);
        }
    }
```
2. alice sends the underlying NFT to bob, who does what alice did before

3. bob sold the underlying NFT to charlie and charlie burned it for some reason, now if alice or bob calls ProfilePicture.getPFP(), it will return the contract address and id of the underlying NFT since getAddress() == ownerOf() == address(0).

When integrating with other projects, there may be bugs because getPFP returns an incorrect result
 
## Proof of Concept
https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-pfp-protocol/src/ProfilePicture.sol#L94-L105
https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-identity-protocol/src/AddressRegistry.sol#L86-L92
## Tools Used
None
## Recommended Mitigation Steps
Change to
```diff
    function getPFP(uint256 _pfpID) public view returns (address nftContract, uint256 nftID) {
        if (_ownerOf[_pfpID] == address(0)) revert TokenNotMinted(_pfpID);
        ProfilePictureData storage pictureData = pfp[_pfpID];
        nftContract = pictureData.nftContract;
        nftID = pictureData.nftID;
        uint256 cidNFTID = cidNFT.getPrimaryCIDNFT(subprotocolName, _pfpID);
        IAddressRegistry addressRegistry = cidNFT.addressRegistry();
-       if (cidNFTID == 0 || addressRegistry.getAddress(cidNFTID) != ERC721(nftContract).ownerOf(nftID)) {
+       if (cidNFTID == 0 || ERC721(nftContract).ownerOf(nftID) == 0 || addressRegistry.getAddress(cidNFTID) != ERC721(nftContract).ownerOf(nftID)) {
            nftContract = address(0);
            nftID = 0; // Strictly not needed because nftContract has to be always checked, but reset nevertheless to 0
        }
    }
```