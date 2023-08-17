## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor acknowledged
- M-04

# [it's possible to swap NFT token ids without fee and also attacker can wrap unwrap all the NFT token balance of the Pair contract and steal their air drops for those token ids](https://github.com/code-423n4/2022-12-caviar-findings/issues/367) 

# Lines of code

https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L217-L243
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L248-L262


# Vulnerability details

## Impact
users can `wrap()` their NFT tokens (which id is whitelisted) and receive `1e18` fractional token or they can pay `1e18` fractional token and unwrap NFT token. there is two issue here:
1. anyone can swap their NFT token id with another NFT token id without paying any fee(both ids should be whitelisted). it's swap without fee.
2. attacker can swap his NFT token(with whitelisted id) for all the NFT balance of contract and steal those NFT tokens airdrop all in one transaction.

## Proof of Concept
This is `wrap()` and `unwrap()` code:
```
    function wrap(uint256[] calldata tokenIds, bytes32[][] calldata proofs)
        public
        returns (uint256 fractionalTokenAmount)
    {
        // *** Checks *** //

        // check that wrapping is not closed
        require(closeTimestamp == 0, "Wrap: closed");

        // check the tokens exist in the merkle root
        _validateTokenIds(tokenIds, proofs);

        // *** Effects *** //

        // mint fractional tokens to sender
        fractionalTokenAmount = tokenIds.length * ONE;
        _mint(msg.sender, fractionalTokenAmount);

        // *** Interactions *** //

        // transfer nfts from sender
        for (uint256 i = 0; i < tokenIds.length; i++) {
            ERC721(nft).safeTransferFrom(msg.sender, address(this), tokenIds[i]);
        }

        emit Wrap(tokenIds);
    }

    function unwrap(uint256[] calldata tokenIds) public returns (uint256 fractionalTokenAmount) {
        // *** Effects *** //

        // burn fractional tokens from sender
        fractionalTokenAmount = tokenIds.length * ONE;
        _burn(msg.sender, fractionalTokenAmount);

        // *** Interactions *** //

        // transfer nfts to sender
        for (uint256 i = 0; i < tokenIds.length; i++) {
            ERC721(nft).safeTransferFrom(address(this), msg.sender, tokenIds[i]);
        }

        emit Unwrap(tokenIds);
    }
```
As you can see it's possible to wrap one NFT token (which id is whitelisted and is in merkle tree) and unwrap another NFT token without paying fee. so Pair contract create NFT swap without fee for users but there is no fee generated for those who wrapped and put their fractional tokens as liquidity providers.
The other issue with this is that some NFT tokens air drop new NFT tokens for NFT holders by making NFT holders to call `getAirdrop()` function. attacker can use this swap functionality to get air drop token for all the NFT balance of the Pair contract. to steps to perform this attack:
1. if Pair contract is for NFT1 and baseToken1 and also merkle tree root hash is 0x0.
2. users deposited 100 NFT1 tokens to the Pair contract.
3. NFT1 decide to airdrop some new tokens for token holders and token holders need to call `nft.getAirDrop(id)` while they own the NFT id.
4. attacker would create a contract and buy one of the NFT1 tokens (attackerID1) and wrap it to receive `1e18` fractional tokens and perform this steps in the contract:
4.1 loop through all the NFT tokens in the Pair contract balance and:
4.2 unwrap NFT token id=i from Pair contract by paying `1e18` fractional token.
4.3 call `nft.getAirDrop(i)` and receive the new airdrop token. (the name of the function can be other thing not exactly `getAirDrop()`)
4.4 wrap NFT token id=i and receive `1e18` fractional token.
5. in the end attacker would unwrap attackerID1 token from Pair contract.
so attacker was abled to receive all the air drops of the NFT tokens that were in the contract address, there could be 100 or 1000 NFT tokens in the contract address and attacker can steal their air drops in one transaction(by writing a contract). those air drops belongs to all the fractional owners and contract shouldn't allow one user to take all the air drops for himself. as airdrops are common in NFT collections so this bug is critical and would happen.

also some of the NFT tokens allows users to stake some tokens for their NFT tokens and receive rewards(for example BAYC/MAYC). if a user stakes tokens for his NFT tokens then wrap those NFT tokens then it would be possible for attacker to unwrap those tokens and steal user staked amounts. in this scenario user made a risky move and wrapped NFT tokens while they have stake but as a lot of users wants to stake for their NFTs this would make them unable to use caviar protocol.

also any other action that attacker can perform by becoming the owner of the NFT token is possible by this attack and if that action can harm the NFT token holders then attacker can harm by doing this attack and performing that action.

## Tools Used
VIM

## Recommended Mitigation Steps
the real solution to prevent this attack (stealing air drops) can be hard. some of the things can be done is:
create functionality so admin can call `getAirDrop()` functions during the airdrops before attacker.
call `getAirDrop()` (which admin specified) function before unwrapping tokens.
make some fee for NFT token unwrapping.
create some lock time(some days) for each wrapped NFT that in that lock time only the one who supplied that token can unwrap it.
create some delay for unwrapping tokens and if user wants to unwrap token he would receive it after this delay.
