## Tags

- bug
- 2 (Med Risk)
- selected for report

# [Contract Owner Possesses Too Many Privileges](https://github.com/code-423n4/2022-10-blur-findings/issues/235) 

# Lines of code

https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/ExecutionDelegate.sol#L119


# Vulnerability details

### Protocol can be easily rug-pulled by the owner
To use the protocol (buy/sell NFTs), users must approve the `ExecutionDelegate` to handle transfers for their `ERC721`, `ERC1155`, or `ERC20` tokens. 

The safety mechanisms mentioned by the protocol do not protect users at all if the project's owner decides to rugpull.

From the contest page, Safety Features:
-   The calling contract must be approved on the `ExecutionDelegate`
-   Users have the ability to revoke approval from the `ExecutionDelegate` without having to individually calling every token contract.

#### POC
```sol
function transferERC20(address token, address from, address to, uint256 amount)
        approvedContract
        external
        returns (bool)
    {
        require(revokedApproval[from] == false, "User has revoked approval");
        return IERC20(token).transferFrom(from, to, amount);
    }
```
The owner can set `approvedContract`  to any address at any time with `approveContract(address _contract)`, and `revokeApproval()` can be frontrun. As a result, all user funds approved to the `ExecutionDelegate` contract can be lost via rugpull.

#### Justification
While rug-pulling may not be the project's intention, I find that this is still an inherently dangerous design.

I am unsure about the validity of centralization risk findings on C4, but I argue this is a valid High risk issue as:
- It is too easy to steal all of user funds as a project owner. A single Bored Ape NFT traded on the exchange would mean roughly \$200,000 can be stolen based on current floor price (75.6 ETH as of writing, Source: https://nftpricefloor.com/bored-ape-yacht-club). \$200k because 75.6ETH for NFT seller and at least 75.6ETH approved by buyer.
- web3 security should not be based on "trust".
- Assuming the project owner is not malicious and will never rug-pull:
	- 1 successful phishing attack (private key compromise) against the project's owner is all it takes to wipe the protocol out.
	- the protocol is still affected as user's will not want to trade on a platfrom knowing such an attack is possible.

#### Recommendations
This is due to an insecure design of the protocol. So as far as recommendations go, the team should reconsider the protocol's design. 

I do not think `ExecutionDelegate` should be used. It would be better if `BlurExchange.sol` is approved by users instead. The exchange should require that the buyer has received their NFT and the seller has received their ETH/WETH or revert.
