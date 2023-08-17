## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-09

# [Unable to adjust position in some cases](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/454) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Position.sol#L132-L152


# Vulnerability details

## Impact
The `adjust` function in `Position.sol` is designed to adjust the outstanding amount of ZCHF, the collateral amount, and the price in a single transaction. 
However, there are certain cases where this function always reverts.
Assuming the new price is greater than the current price, if the value of `newCollateral` is less than `colbal` or the value of `newMinted` is greater than `minted`, the `adjust` function will always revert with a customized error message reading `Hot`.
## Proof of Concept
If the value of `newPrice` is great than value of `price`, the `restrictMinting` function is triggered.
In this case, if the value of `cooldown` exceeds `block.timestamp` + 3 days, `cooldown` will be update to `block.timestamp + 3 days`, therefore, both the `withdrawCollateral` and `mint` functions will be reverted with custom error because of `noCooldown` modifier.
https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Position.sol#L132-L152
https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Position.sol#L159-L167
I will share the test code
```
it("Will revert adjest tx when newPrice is greater than prevPrice", async () => {
    let collateral = mockVOL.address;
    let fliqPrice = floatToDec18(1000);
    let minCollateral = floatToDec18(1);
    let fInitialCollateral = floatToDec18(initialCollateral);
    let duration = BN.from(14*86_400);
    let fFees = BN.from(fee * 1000_000);
    let fReserve = BN.from(reserve * 1000_000);
    let openingFeeZCHF = await mintingHubContract.OPENING_FEE();
    let challengePeriod = BN.from(7 * 86400); // 7 days
    await mockVOL.connect(accounts[0]).approve(mintingHubContract.address, fInitialCollateral);
    let balBefore = await ZCHFContract.balanceOf(owner);
    let balBeforeVOL = await mockVOL.balanceOf(owner);
    let tx = await mintingHubContract["openPosition(address,uint256,uint256,uint256,uint256,uint256,uint32,uint256,uint32)"]
        (collateral, minCollateral, fInitialCollateral, initialLimit, duration, challengePeriod, fFees, fliqPrice, fReserve);
    let rc = await tx.wait();
    const topic = '0x591ede549d7e337ac63249acd2d7849532b0a686377bbf0b0cca6c8abd9552f2'; // PositionOpened
    const log = rc.logs.find(x => x.topics.indexOf(topic) >= 0);
    positionAddr = log.address;
    let balAfter = await ZCHFContract.balanceOf(owner);
    let balAfterVOL = await mockVOL.balanceOf(owner);
    let dZCHF = dec18ToFloat(balAfter.sub(balBefore));
    let dVOL = dec18ToFloat(balAfterVOL.sub(balBeforeVOL));
    expect(dVOL).to.be.equal(-initialCollateral);
    expect(dZCHF).to.be.equal(-dec18ToFloat(openingFeeZCHF));
    positionContract = await ethers.getContractAt('Position', positionAddr, accounts[0]);

    console.log("price:",await positionContract.price());
    console.log("minted:",await positionContract.minted());
    await ethers.provider.send('evm_increaseTime', [7 * 86_400 + 60]); 
    await ethers.provider.send("evm_mine");

    let erx = positionContract.adjust(1, floatToDec18(8), floatToDec18(1200))
    await expect(erx).to.be.revertedWithCustomError(positionContract, "Hot");

    console.log("price:",await positionContract.price());
    console.log("minted:",await positionContract.minted());
});
```


## Tools Used
VS Code
## Recommended Mitigation Steps

To solve this problem, we can modify the "adjust" function like this;
```
function adjust(uint256 newMinted, uint256 newCollateral, uint256 newPrice) public onlyOwner {
    uint256 colbal = collateralBalance();
    if (newCollateral > colbal){
        collateral.transferFrom(msg.sender, address(this), newCollateral - colbal);
    }
    // Must be called after collateral deposit, but before withdrawal
    if (newMinted < minted){
        zchf.burnFrom(msg.sender, minted - newMinted, reserveContribution);
        minted = newMinted;
    }
    if (newCollateral < colbal){
        withdrawCollateral(msg.sender, colbal - newCollateral);
    }
    // Must be called after collateral withdrawal
    if (newMinted > minted){
        mint(msg.sender, newMinted - minted);
    }

    if (newPrice != price){
        adjustPrice(newPrice);
    }
}
```