## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- M-14

# [The `royaltyRecipient` could not be prepare to receive ether, making the `sell` to fail](https://github.com/code-423n4/2023-04-caviar-findings/issues/263) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/EthRouter.sol#L188-L191


# Vulnerability details

## Impact
The `royaltyRecipient` is an arbitrary address setup by the collection if the collection `royaltyRecipient` is a contract and this contract its not prepared to receive ether the ether transfer will always fail paying the royalties.



## Proof of Concept
Here is a foundry POC, take note that i have to write a new Milady mock collection because in the original is hardcoded to `0xbeefbeef` so its impossible to change the `royaltyRecipient`;  [Milady.sol#L31](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/test/shared/Milady.sol#L31)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "test/Fixture.sol";

contract POCRejectTest is Fixture {
    PrivatePool public privatePool;
    uint256 public totalTokens = 0;
    uint256 public minOutputAmount = 0;

    uint256 royaltyFeeRate = 0.1e18; // 10%
    address royaltyRecipient = address(new EthRejecter());

    Milady2 milady2 = new Milady2();

    function setUp() public {
        milady2.setApprovalForAll(address(ethRouter), true);

        vm.mockCall(
            address(milady2),
            abi.encodeWithSelector(ERC721.ownerOf.selector, address(privatePool)),
            abi.encode(address(this))
        );

        // lets setup a trap for the royalty        
        milady2.setRoyaltyInfo(royaltyFeeRate, royaltyRecipient);

    }

    function _addSell() internal returns (EthRouter.Sell memory, uint256) {
        uint256[] memory empty = new uint256[](0);
        privatePool = factory.create{value: 100e18}(
            address(0),
            address(milady2),
            100e18,
            10e18,
            200,
            199,
            bytes32(0),
            true,
            false,
            bytes32(address(this).balance), // random between each call to _addBuy
            empty,
            100e18
        );

        uint256[] memory tokenIds = new uint256[](2);
        for (uint256 i = 0; i < 2; i++) {
            milady2.mint(address(this), i + totalTokens);
            tokenIds[i] = i + totalTokens;
        }

        totalTokens += 2;

        bytes32[][] memory publicPoolProofs = new bytes32[][](0);
        EthRouter.Sell memory sell = EthRouter.Sell({
            pool: payable(address(privatePool)),
            nft: address(milady2),
            tokenIds: tokenIds,
            tokenWeights: new uint256[](0),
            proof: PrivatePool.MerkleMultiProof(new bytes32[](0), new bool[](0)),
            stolenNftProofs: new IStolenNftOracle.Message[](0),
            isPublicPool: false,
            publicPoolProofs: publicPoolProofs
        });

        (uint256 baseTokenAmount,,) = privatePool.sellQuote(tokenIds.length * 1e18);
        return (sell, baseTokenAmount);
    }

    function test_PaysRoyalties() public {
        // arrange
        EthRouter.Sell[] memory sells = new EthRouter.Sell[](3);
        (EthRouter.Sell memory sell1, uint256 outputAmount1) = _addSell();
        (EthRouter.Sell memory sell2, uint256 outputAmount2) = _addSell();
        minOutputAmount += outputAmount1 + outputAmount2;
        sells[0] = sell1;
        sells[1] = sell2;
        Pair pair = caviar.create(address(milady2), address(0), bytes32(0));
        deal(address(pair), 1.123e18);
        deal(address(pair), address(pair), 10e18);

        uint256[] memory tokenIds = new uint256[](2);
        for (uint256 i = 0; i < tokenIds.length; i++) {
            tokenIds[i] = i + totalTokens;
            milady2.mint(address(this), i + totalTokens);
        }
        sells[2] = EthRouter.Sell({
            pool: payable(address(pair)),
            nft: address(milady2),
            tokenIds: tokenIds,
            tokenWeights: new uint256[](0),
            proof: PrivatePool.MerkleMultiProof(new bytes32[](0), new bool[](0)),
            stolenNftProofs: new IStolenNftOracle.Message[](0),
            isPublicPool: true,
            publicPoolProofs: new bytes32[][](0)
        });

        uint256 outputAmount = pair.sellQuote(tokenIds.length * 1e18);

        uint256 royaltyFee = outputAmount / tokenIds.length * royaltyFeeRate / 1e18 * tokenIds.length;
        outputAmount -= royaltyFee;
        minOutputAmount += outputAmount;

        // act
        ethRouter.sell(sells, minOutputAmount, 0, true);

        // assert
        assertEq(address(royaltyRecipient).balance, royaltyFee, "Should have paid royalties");
        assertGt(address(royaltyRecipient).balance, 0, "Should have paid royalties");
    }
}

contract EthRejecter {
    // The contract could not have a method called "receive" or "fallback", i added here
    // to show the concept of a contract that rejects ETH
    receive() external payable {
        revert("ETH REJECTED EXAMPLE");
    }
}

contract Milady2 is ERC721, ERC2981 {
    uint256 public royaltyFeeRate = 0; // to 18 decimals
    address public royaltyRecipient = address(0);

    constructor() ERC721("Milady Maker", "MIL") {}

    function tokenURI(uint256) public view virtual override returns (string memory) {
        return "https://milady.io";
    }

    function mint(address to, uint256 id) public {
        _mint(to, id);
    }

    function setRoyaltyInfo(uint256 _royaltyFeeRate, address _royaltyRecipient) public {
        royaltyFeeRate = _royaltyFeeRate;
        royaltyRecipient = _royaltyRecipient;
    }

    function supportsInterface(bytes4 interfaceId) public view override(ERC2981, ERC721) returns (bool) {
        return super.supportsInterface(interfaceId);
    }

    function royaltyInfo(uint256, uint256 salePrice) public view override returns (address, uint256) {
        return (royaltyRecipient, salePrice * royaltyFeeRate / 1e18);
    }
}

```


## Tools Used
Manual revision


## Recommended Mitigation Steps

There are two simple ways from my point of view to force ether send and solve this issue;

You could use a simple contract that selfdestrcut an firce ether, but selfdestruct is deprecated so its not a good idea, please view [solady/SafeTransferLib.sol#L65](https://github.com/Vectorized/solady/blob/main/src/utils/SafeTransferLib.sol#L65)

The other think you could do is if a address is rejecting ether, send WETH instead, this pattern is common and well known.