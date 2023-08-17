## Tags

- bug
- G (Gas Optimization)
- grade-a
- selected for report
- G-33

# [Gas Optimizations](https://github.com/code-423n4/2022-10-inverse-findings/issues/368) 

##### Gas Optimizations

Gas savings are estimated using the gas report of existing `forge test --gas-report` tests (the sum of all deployment costs and the sum of the costs of calling methods) and may vary depending on the implementation of the fix.

|        | Issue                                                                                                        | Instances | Estimated gas(deployments) | Estimated gas(min method call) | Estimated gas(avg method call) | Estimated gas(max method call) |
| :----: | :----------------------------------------------------------------------------------------------------------- | :-------: | :------------------------: | :----------------------------: | :----------------------------: | :----------------------------: |
| **1**  | State variables only set in the constructor should be declared immutable                                     |     2     |          117 275           |              104               |              110               |              110               |
| **2**  | Use function instead of modifiers                                                                            |     4     |          115 926           |              162               |              -264              |              -481              |
| **3**  | Duplicated require()/revert() checks should be refactored to a modifier or function                          |    11     |          114 932           |              -59               |              -284              |              -398              |
| **4**  | Multiple address mappings can be combined into a single mapping of an address to a struct, where appropriate |     5     |           24 227           |              254               |              533               |             -6 726             |
| **5**  | Expression can be unchecked when overflow is not possible                                                    |     6     |           20 220           |              410               |             4 630              |              1354              |
| **6**  | State variables can be packed into fewer storage slots                                                       |     1     |           -5 008           |             1 911              |             15 525             |             20 972             |
| **7**  | refactoring similar statements                                                                               |     1     |           18 422           |              -18               |              -11               |               6                |
| **8**  | Better algorithm for underflow check                                                                         |     3     |           12 613           |              656               |             8 332              |             3 741              |
| **9**  | `x = x + y` is cheaper than `x += y`                                                                         |    12     |           11 214           |              180               |              468               |              616               |
| **10** | `internal` functions only called once can be inlined to save gas                                             |     1     |           5 207            |               67               |               47               |               24               |
| **11** | State variables should be cached in stack variables rather than re-reading them from storage                 |     2     |           5 007            |              478               |             1 117              |             1 423              |
|        | **Overall gas savings**                                                                                      |  **48**   |    **416 802 (6,58%)**     |       **3 423 (0,34%)**        |       **15 773 (0,82%)**       |       **18 283 (0,72%)**       |

**Total: 48 instances over 11 issues**

---

#### 1. **State variables only set in the constructor should be declared immutable (2 instances)**

Deployment. Gas Saved: **117 275**

Minimum Method Call. Gas Saved: **104**

Average Method Call. Gas Saved: **110**

Maximum Method Call. Gas Saved: **110**

Overall gas change: **-678 (-0.723%)**

Avoids a Gsset (20000 gas) in the constructor, and replaces each Gwarmacces (100 gas) with a PUSH32 (3 gas).

##### - src/DBR.sol:[11](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L11), [12](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L12)

**NOTE:** name and symbol must be within 32 bytes

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..013960f 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -8,8 +8,8 @@ pragma solidity ^0.8.13;
    8,   8: */
    9,   9: contract DolaBorrowingRights {
   10,  10: 
-  11     :-    string public name;
-  12     :-    string public symbol;
+       11:+    bytes32 public immutable name;
+       12:+    bytes32 public immutable symbol;
   13,  13:     uint8 public constant decimals = 18;
   14,  14:     uint256 public _totalSupply;
   15,  15:     address public operator;
@@ -34,8 +34,8 @@ contract DolaBorrowingRights {
   34,  34:         address _operator
   35,  35:     ) {
   36,  36:         replenishmentPriceBps = _replenishmentPriceBps;
-  37     :-        name = _name;
-  38     :-        symbol = _symbol;
+       37:+        name = bytes32(bytes(_name));
+       38:+        symbol = bytes32(bytes(_symbol));
   39,  39:         operator = _operator;
   40,  40:         INITIAL_CHAIN_ID = block.chainid;
   41,  41:         INITIAL_DOMAIN_SEPARATOR = computeDomainSeparator();
@@ -268,7 +268,7 @@ contract DolaBorrowingRights {
  268, 268:             keccak256(
  269, 269:                 abi.encode(
  270, 270:                     keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
- 271     :-                    keccak256(bytes(name)),
+      271:+                    keccak256(bytes.concat(name)),
  272, 272:                     keccak256("1"),
  273, 273:                     block.chainid,
  274, 274:                     address(this)
```

---

#### 2. **Use function instead of modifiers (4 instances)**

Deployment. Gas Saved: **115 926**

Minimum Method Call. Gas Saved: **162**

Average Method Call. Gas Saved: **-264**

Maximum Method Call. Gas Saved: **-481**

Overall gas change: **734 (2.459%)**

##### - src/BorrowController.sol:[17](https://github.com/code-423n4/2022-10-inverse/blob/main/src/BorrowController.sol#L17)

```diff
diff --git a/src/BorrowController.sol b/src/BorrowController.sol
index 6decad1..080a4e3 100644
--- a/src/BorrowController.sol
+++ b/src/BorrowController.sol
@@ -14,28 +14,36 @@ contract BorrowController {
   14,  14:         operator = _operator;
   15,  15:     }
   16,  16: 
-  17     :-    modifier onlyOperator {
+       17:+    function onlyOperator() private view {
   18,  18:         require(msg.sender == operator, "Only operator");
-  19     :-        _;
   20,  19:     }
   21,  20:     
   22,  21:     /**
   23,  22:     @notice Sets the operator of the borrow controller. Only callable by the operator.
   24,  23:     @param _operator The address of the new operator.
   25,  24:     */
-  26     :-    function setOperator(address _operator) public onlyOperator { operator = _operator; }
+       25:+    function setOperator(address _operator) public { 
+       26:+        onlyOperator();
+       27:+        operator = _operator; 
+       28:+    }
   27,  29: 
   28,  30:     /**
   29,  31:     @notice Allows a contract to use the associated market.
   30,  32:     @param allowedContract The address of the allowed contract
   31,  33:     */
-  32     :-    function allow(address allowedContract) public onlyOperator { contractAllowlist[allowedContract] = true; }
+       34:+    function allow(address allowedContract) public { 
+       35:+        onlyOperator();
+       36:+        contractAllowlist[allowedContract] = true; 
+       37:+    }
   33,  38: 
   34,  39:     /**
   35,  40:     @notice Denies a contract to use the associated market
   36,  41:     @param deniedContract The addres of the denied contract
   37,  42:     */
-  38     :-    function deny(address deniedContract) public onlyOperator { contractAllowlist[deniedContract] = false; }
+       43:+    function deny(address deniedContract) public { 
+       44:+        onlyOperator();
+       45:+        contractAllowlist[deniedContract] = false; 
+       46:+    }
   39,  47: 
   40,  48:     /**
   41,  49:     @notice Checks if a borrow is allowed
```

##### - src/DBR.sol:[44](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L44)

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..50428cd 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -41,16 +41,16 @@ contract DolaBorrowingRights {
   41,  41:         INITIAL_DOMAIN_SEPARATOR = computeDomainSeparator();
   42,  42:     }
   43,  43: 
-  44     :-    modifier onlyOperator {
+       44:+    function onlyOperator() private view {
   45,  45:         require(msg.sender == operator, "ONLY OPERATOR");
-  46     :-        _;
   47,  46:     }
   48,  47:     
   49,  48:     /**
   50,  49:     @notice Sets pending operator of the contract. Operator role must be claimed by the new oprator. Only callable by Operator.
   51,  50:     @param newOperator_ The address of the newOperator
   52,  51:     */
-  53     :-    function setPendingOperator(address newOperator_) public onlyOperator {
+       52:+    function setPendingOperator(address newOperator_) public {
+       53:+        onlyOperator();
   54,  54:         pendingOperator = newOperator_;
   55,  55:     }
   56,  56: 
@@ -59,7 +59,8 @@ contract DolaBorrowingRights {
   59,  59:      At 10000, the cost of replenishing 1 DBR is 1 DOLA in debt. Only callable by Operator.
   60,  60:     @param newReplenishmentPriceBps_ The new replen
   61,  61:     */
-  62     :-    function setReplenishmentPriceBps(uint newReplenishmentPriceBps_) public onlyOperator {
+       62:+    function setReplenishmentPriceBps(uint newReplenishmentPriceBps_) public {
+       63:+        onlyOperator();
   63,  64:         require(newReplenishmentPriceBps_ > 0, "replenishment price must be over 0");
   64,  65:         replenishmentPriceBps = newReplenishmentPriceBps_;
   65,  66:     }
@@ -78,7 +79,8 @@ contract DolaBorrowingRights {
   78,  79:     @notice Add a minter to the set of addresses allowed to mint DBR tokens. Only callable by Operator.
   79,  80:     @param minter_ The address of the new minter.
   80,  81:     */
-  81     :-    function addMinter(address minter_) public onlyOperator {
+       82:+    function addMinter(address minter_) public {
+       83:+        onlyOperator();
   82,  84:         minters[minter_] = true;
   83,  85:         emit AddMinter(minter_);
   84,  86:     }
@@ -87,7 +89,8 @@ contract DolaBorrowingRights {
   87,  89:     @notice Removes a minter from the set of addresses allowe to mint DBR tokens. Only callable by Operator.
   88,  90:     @param minter_ The address to be removed from the minter set.
   89,  91:     */
-  90     :-    function removeMinter(address minter_) public onlyOperator {
+       92:+    function removeMinter(address minter_) public {
+       93:+        onlyOperator();
   91,  94:         minters[minter_] = false;
   92,  95:         emit RemoveMinter(minter_);
   93,  96:     }
@@ -96,7 +99,8 @@ contract DolaBorrowingRights {
   96,  99:     @dev markets can be added but cannot be removed. A removed market would result in unrepayable debt for some users.
   97, 100:     @param market_ The address of the new market contract to be added.
   98, 101:     */
-  99     :-    function addMarket(address market_) public onlyOperator {
+      102:+    function addMarket(address market_) public {
+      103:+        onlyOperator();
  100, 104:         markets[market_] = true;
  101, 105:         emit AddMarket(market_);
  102, 106:     }
```

##### - src/Market.sol:[92](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L92)

```diff
diff --git a/src/Market.sol b/src/Market.sol
index 9585b85..796d0d0 100644
--- a/src/Market.sol
+++ b/src/Market.sol
@@ -89,9 +89,8 @@ contract Market {
   89,  89:         INITIAL_DOMAIN_SEPARATOR = computeDomainSeparator();
   90,  90:     }
   91,  91:     
-  92     :-    modifier onlyGov {
+       92:+    function onlyGov() private view {
   93,  93:         require(msg.sender == gov, "Only gov can call this function");
-  94     :-        _;
   95,  94:     }
   96,  95: 
   97,  96:     function DOMAIN_SEPARATOR() public view virtual returns (bytes32) {
@@ -115,38 +114,54 @@ contract Market {
  115, 114:     @notice sets the oracle to a new oracle. Only callable by governance.
  116, 115:     @param _oracle The new oracle conforming to the IOracle interface.
  117, 116:     */
- 118     :-    function setOracle(IOracle _oracle) public onlyGov { oracle = _oracle; }
+      117:+    function setOracle(IOracle _oracle) public { 
+      118:+        onlyGov();
+      119:+        oracle = _oracle; 
+      120:+    }
  119, 121: 
  120, 122:     /**
  121, 123:     @notice sets the borrow controller to a new borrow controller. Only callable by governance.
  122, 124:     @param _borrowController The new borrow controller conforming to the IBorrowController interface.
  123, 125:     */
- 124     :-    function setBorrowController(IBorrowController _borrowController) public onlyGov { borrowController = _borrowController; }
+      126:+    function setBorrowController(IBorrowController _borrowController) public { 
+      127:+        onlyGov();
+      128:+        borrowController = _borrowController; 
+      129:+    }
  125, 130: 
  126, 131:     /**
  127, 132:     @notice sets the address of governance. Only callable by governance.
  128, 133:     @param _gov Address of the new governance.
  129, 134:     */
- 130     :-    function setGov(address _gov) public onlyGov { gov = _gov; }
+      135:+    function setGov(address _gov) public { 
+      136:+        onlyGov();
+      137:+        gov = _gov; 
+      138:+    }
  131, 139: 
  132, 140:     /**
  133, 141:     @notice sets the lender to a new lender. The lender is allowed to recall dola from the contract. Only callable by governance.
  134, 142:     @param _lender Address of the new lender.
  135, 143:     */
- 136     :-    function setLender(address _lender) public onlyGov { lender = _lender; }
+      144:+    function setLender(address _lender) public { 
+      145:+        onlyGov();
+      146:+        lender = _lender; 
+      147:+    }
  137, 148: 
  138, 149:     /**
  139, 150:     @notice sets the pause guardian. The pause guardian can pause borrowing. Only callable by governance.
  140, 151:     @param _pauseGuardian Address of the new pauseGuardian.
  141, 152:     */
- 142     :-    function setPauseGuardian(address _pauseGuardian) public onlyGov { pauseGuardian = _pauseGuardian; }
+      153:+    function setPauseGuardian(address _pauseGuardian) public { 
+      154:+        onlyGov();
+      155:+        pauseGuardian = _pauseGuardian; 
+      156:+    }
  143, 157:     
  144, 158:     /**
  145, 159:     @notice sets the Collateral Factor requirement of the market as measured in basis points. 1 = 0.01%. Only callable by governance.
  146, 160:     @dev Collateral factor mus be set below 100%
  147, 161:     @param _collateralFactorBps The new collateral factor as measured in basis points. 
  148, 162:     */
- 149     :-    function setCollateralFactorBps(uint _collateralFactorBps) public onlyGov {
+      163:+    function setCollateralFactorBps(uint _collateralFactorBps) public  {
+      164:+        onlyGov();
  150, 165:         require(_collateralFactorBps < 10000, "Invalid collateral factor");
  151, 166:         collateralFactorBps = _collateralFactorBps;
  152, 167:     }
@@ -158,7 +173,8 @@ contract Market {
  158, 173:     @dev Must be set between 1 and 10000.
  159, 174:     @param _liquidationFactorBps The new liquidation factor in basis points. 1 = 0.01%/
  160, 175:     */
- 161     :-    function setLiquidationFactorBps(uint _liquidationFactorBps) public onlyGov {
+      176:+    function setLiquidationFactorBps(uint _liquidationFactorBps) public  {
+      177:+        onlyGov();
  162, 178:         require(_liquidationFactorBps > 0 && _liquidationFactorBps <= 10000, "Invalid liquidation factor");
  163, 179:         liquidationFactorBps = _liquidationFactorBps;
  164, 180:     }
@@ -169,7 +185,8 @@ contract Market {
  169, 185:     @dev Must be set between 1 and 10000.
  170, 186:     @param _replenishmentIncentiveBps The new replenishment incentive set in basis points. 1 = 0.01%
  171, 187:     */
- 172     :-    function setReplenismentIncentiveBps(uint _replenishmentIncentiveBps) public onlyGov {
+      188:+    function setReplenismentIncentiveBps(uint _replenishmentIncentiveBps) public {
+      189:+        onlyGov();
  173, 190:         require(_replenishmentIncentiveBps > 0 && _replenishmentIncentiveBps < 10000, "Invalid replenishment incentive");
  174, 191:         replenishmentIncentiveBps = _replenishmentIncentiveBps;
  175, 192:     }
@@ -180,7 +197,8 @@ contract Market {
  180, 197:     @dev Must be set between 0 and 10000 - liquidation fee.
  181, 198:     @param _liquidationIncentiveBps The new liqudation incentive set in basis points. 1 = 0.01% 
  182, 199:     */
- 183     :-    function setLiquidationIncentiveBps(uint _liquidationIncentiveBps) public onlyGov {
+      200:+    function setLiquidationIncentiveBps(uint _liquidationIncentiveBps) public {
+      201:+        onlyGov();
  184, 202:         require(_liquidationIncentiveBps > 0 && _liquidationIncentiveBps + liquidationFeeBps < 10000, "Invalid liquidation incentive");
  185, 203:         liquidationIncentiveBps = _liquidationIncentiveBps;
  186, 204:     }
@@ -191,7 +209,8 @@ contract Market {
  191, 209:     @dev Must be set between 0 and 10000 - liquidation factor.
  192, 210:     @param _liquidationFeeBps The new liquidation fee set in basis points. 1 = 0.01%
  193, 211:     */
- 194     :-    function setLiquidationFeeBps(uint _liquidationFeeBps) public onlyGov {
+      212:+    function setLiquidationFeeBps(uint _liquidationFeeBps) public {
+      213:+        onlyGov();
  195, 214:         require(_liquidationFeeBps > 0 && _liquidationFeeBps + liquidationIncentiveBps < 10000, "Invalid liquidation fee");
  196, 215:         liquidationFeeBps = _liquidationFeeBps;
  197, 216:     }
```

##### - src/Oracle.sol:[35](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Oracle.sol#L35)

```diff
diff --git a/src/Oracle.sol b/src/Oracle.sol
index 14338ed..3e7c608 100644
--- a/src/Oracle.sol
+++ b/src/Oracle.sol
@@ -32,16 +32,18 @@ contract Oracle {
   32,  32:         operator = _operator;
   33,  33:     }
   34,  34: 
-  35     :-    modifier onlyOperator {
+       35:+    function onlyOperator() private view {
   36,  36:         require(msg.sender == operator, "ONLY OPERATOR");
-  37     :-        _;
   38,  37:     }
   39,  38:     
   40,  39:     /**
   41,  40:     @notice Sets the pending operator of the oracle. Only callable by operator.
   42,  41:     @param newOperator_ The address of the pending operator.
   43,  42:     */
-  44     :-    function setPendingOperator(address newOperator_) public onlyOperator { pendingOperator = newOperator_; }
+       43:+    function setPendingOperator(address newOperator_) public { 
+       44:+        onlyOperator();
+       45:+        pendingOperator = newOperator_; 
+       46:+    }
   45,  47: 
   46,  48:     /**
   47,  49:     @notice Sets the price feed of a specific token address.
@@ -50,7 +52,10 @@ contract Oracle {
   50,  52:     @param feed The chainlink feed of the ERC20 token.
   51,  53:     @param tokenDecimals uint8 representing the decimal precision of the token
   52,  54:     */
-  53     :-    function setFeed(address token, IChainlinkFeed feed, uint8 tokenDecimals) public onlyOperator { feeds[token] = FeedData(feed, tokenDecimals); }
+       55:+    function setFeed(address token, IChainlinkFeed feed, uint8 tokenDecimals) public { 
+       56:+        onlyOperator();
+       57:+        feeds[token] = FeedData(feed, tokenDecimals); 
+       58:+    }
   54,  59: 
   55,  60:     /**
   56,  61:     @notice Sets a fixed price for a token
@@ -58,7 +63,10 @@ contract Oracle {
   58,  63:     @param token The address of the fixed price token
   59,  64:     @param price The fixed price of the token. Remember to account for decimal precision when setting this.
   60,  65:     */
-  61     :-    function setFixedPrice(address token, uint price) public onlyOperator { fixedPrices[token] = price; }
+       66:+    function setFixedPrice(address token, uint price) public { 
+       67:+        onlyOperator();
+       68:+        fixedPrices[token] = price; 
+       69:+    }
   62,  70: 
   63,  71:     /**
   64,  72:     @notice Claims the operator role. Only successfully callable by the pending operator.
```

---

#### 3. **Duplicated require()/revert() checks should be refactored to a modifier or function ( instances)**

Deployment. Gas Saved: **114 932**

Minimum Method Call. Gas Saved: **-59**

Average Method Call. Gas Saved: **-284**

Maximum Method Call. Gas Saved: **-398**

Overall gas change: **-2 665 (-12.599%)**

##### - src/Fed.sol:[49](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L49), [58](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L58), [67](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L67), [76](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L76), [87](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L87), [88](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L88), [104](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L104), [105](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L105)

```diff
diff --git a/src/Fed.sol b/src/Fed.sol
index 1e819bb..8b54676 100644
--- a/src/Fed.sol
+++ b/src/Fed.sol
@@ -41,12 +41,24 @@ contract Fed {
   41,  41:         supplyCeiling = _supplyCeiling;
   42,  42:     }
   43,  43: 
+       44:+    function is_gov() private view {
+       45:+        require(msg.sender == gov, "ONLY GOV");
+       46:+    }
+       47:+
+       48:+    function is_chair() private view {
+       49:+        require(msg.sender == chair, "ONLY CHAIR");
+       50:+    }
+       51:+
+       52:+    function is_supported_market(IMarket _market) private view {
+       53:+        require(dbr.markets(address(_market)), "UNSUPPORTED MARKET");
+       54:+    }
+       55:+
   44,  56:     /**
   45,  57:     @notice Change the governance of the Fed contact. Only callable by governance.
   46,  58:     @param _gov The address of the new governance contract
   47,  59:     */
   48,  60:     function changeGov(address _gov) public {
-  49     :-        require(msg.sender == gov, "ONLY GOV");
+       61:+        is_gov();
   50,  62:         gov = _gov;
   51,  63:     }
   52,  64: 
@@ -55,7 +67,7 @@ contract Fed {
   55,  67:     @param _supplyCeiling Amount to set the supply ceiling to
   56,  68:     */
   57,  69:     function changeSupplyCeiling(uint _supplyCeiling) public {
-  58     :-        require(msg.sender == gov, "ONLY GOV");
+       70:+        is_gov();
   59,  71:         supplyCeiling = _supplyCeiling;
   60,  72:     }
   61,  73: 
@@ -64,7 +76,7 @@ contract Fed {
   64,  76:     @param _chair Address of the new chair.
   65,  77:     */
   66,  78:     function changeChair(address _chair) public {
-  67     :-        require(msg.sender == gov, "ONLY GOV");
+       79:+        is_gov();
   68,  80:         chair = _chair;
   69,  81:     }
   70,  82: 
@@ -73,7 +85,7 @@ contract Fed {
   73,  85:     @dev Useful for immediately removing chair powers in case of a wallet compromise.
   74,  86:     */
   75,  87:     function resign() public {
-  76     :-        require(msg.sender == chair, "ONLY CHAIR");
+       88:+        is_chair();
   77,  89:         chair = address(0);
   78,  90:     }
   79,  91: 
@@ -84,8 +96,8 @@ contract Fed {
   84,  96:     @param amount The amount of DOLA to mint and supply to the market.
   85,  97:     */
   86,  98:     function expansion(IMarket market, uint amount) public {
-  87     :-        require(msg.sender == chair, "ONLY CHAIR");
-  88     :-        require(dbr.markets(address(market)), "UNSUPPORTED MARKET");
+       99:+        is_chair();
+      100:+        is_supported_market(market);
   89, 101:         require(market.borrowPaused() != true, "CANNOT EXPAND PAUSED MARKETS");
   90, 102:         dola.mint(address(market), amount);
   91, 103:         supplies[market] += amount;
@@ -101,8 +113,8 @@ contract Fed {
  101, 113:     @param amount The amount of DOLA to withdraw and burn.
  102, 114:     */
  103, 115:     function contraction(IMarket market, uint amount) public {
- 104     :-        require(msg.sender == chair, "ONLY CHAIR");
- 105     :-        require(dbr.markets(address(market)), "UNSUPPORTED MARKET");
+      116:+        is_chair();
+      117:+        is_supported_market(market);
  106, 118:         uint supply = supplies[market];
  107, 119:         require(amount <= supply, "AMOUNT TOO BIG"); // can't burn profits
  108, 120:         market.recall(amount);
```

##### - src/DBR.sol:[171](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L171), [195](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L195), [373](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L373)

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..625c422 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -46,6 +46,10 @@ contract DolaBorrowingRights {
   46,  46:         _;
   47,  47:     }
   48,  48:     
+       49:+    function is_balance_sufficient(address _user, uint256 amount) private view {
+       50:+        require(balanceOf(_user) >= amount, "Insufficient balance");
+       51:+    }
+       52:+
   49,  53:     /**
   50,  54:     @notice Sets pending operator of the contract. Operator role must be claimed by the new oprator. Only callable by Operator.
   51,  55:     @param newOperator_ The address of the newOperator
@@ -168,7 +172,7 @@ contract DolaBorrowingRights {
  168, 172:     @return Always returns true, will revert if not successful.
  169, 173:     */
  170, 174:     function transfer(address to, uint256 amount) public virtual returns (bool) {
- 171     :-        require(balanceOf(msg.sender) >= amount, "Insufficient balance");
+      175:+        is_balance_sufficient(msg.sender, amount);
  172, 176:         balances[msg.sender] -= amount;
  173, 177:         unchecked {
  174, 178:             balances[to] += amount;
@@ -192,7 +196,7 @@ contract DolaBorrowingRights {
  192, 196:     ) public virtual returns (bool) {
  193, 197:         uint256 allowed = allowance[from][msg.sender];
  194, 198:         if (allowed != type(uint256).max) allowance[from][msg.sender] = allowed - amount;
- 195     :-        require(balanceOf(from) >= amount, "Insufficient balance");
+      199:+        is_balance_sufficient(from, amount);
  196, 200:         balances[from] -= amount;
  197, 201:         unchecked {
  198, 202:             balances[to] += amount;
@@ -370,7 +374,7 @@ contract DolaBorrowingRights {
  370, 374:     @param amount Amount of DBR to be burned.
  371, 375:     */
  372, 376:     function _burn(address from, uint256 amount) internal virtual {
- 373     :-        require(balanceOf(from) >= amount, "Insufficient balance");
+      377:+        is_balance_sufficient(from, amount);
  374, 378:         balances[from] -= amount;
  375, 379:         unchecked {
  376, 380:             _totalSupply -= amount;
```

---

#### 4. **Multiple address mappings can be combined into a single mapping of an address to a struct, where appropriate (5 instances)**

Deployment. Gas Saved: **24 227**

Minimum Method Call. Gas Saved: **254**

Average Method Call. Gas Saved: **533**

Maximum Method Call. Gas Saved: **-6 726**

Overall gas change: **-1 371 (20.741%)**

Saves a storage slot for the mapping. Depending on the circumstances and sizes of types, can avoid a Gsset (20000 gas) per mapping combined. Reads and subsequent writes can also be cheaper when a function requires both values and they both fit in the same storage slot. Finally, if both fields are accessed in the same function, can save ~42 gas per access due to not having to recalculate the key's keccak256 hash (Gkeccak256 - 30 gas) and that calculation's associated stack operations.

##### - src/DBR.sol:[19](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L19), [23](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L23), [26](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L26), [27](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L27), [28](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L28)

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..43db0aa 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -8,6 +8,17 @@ pragma solidity ^0.8.13;
    8,   8: */
    9,   9: contract DolaBorrowingRights {
   10,  10: 
+       11:+    struct UserInfo {
+       12:+        uint256 balances;
+       13:+        
+       14:+        uint256 nonce;
+       15:+        uint256 debts;  // user => debt across all tracked markets
+       16:+        uint256 dueTokensAccrued; // user => amount of due tokens accrued
+       17:+        uint256 lastUpdated; // user => last update timestamp
+       18:+    }
+       19:+    
+       20:+    mapping(address => mapping(address => uint256)) public allowance;
+       21:+
   11,  22:     string public name;
   12,  23:     string public symbol;
   13,  24:     uint8 public constant decimals = 18;
@@ -16,16 +27,11 @@ contract DolaBorrowingRights {
   16,  27:     address public pendingOperator;
   17,  28:     uint public totalDueTokensAccrued;
   18,  29:     uint public replenishmentPriceBps;
-  19     :-    mapping(address => uint256) public balances;
-  20     :-    mapping(address => mapping(address => uint256)) public allowance;
+       30:+    mapping(address => UserInfo) public userInfo;
   21,  31:     uint256 internal immutable INITIAL_CHAIN_ID;
   22,  32:     bytes32 internal immutable INITIAL_DOMAIN_SEPARATOR;
-  23     :-    mapping(address => uint256) public nonces;
   24,  33:     mapping (address => bool) public minters;
   25,  34:     mapping (address => bool) public markets;
-  26     :-    mapping (address => uint) public debts; // user => debt across all tracked markets
-  27     :-    mapping (address => uint) public dueTokensAccrued; // user => amount of due tokens accrued
-  28     :-    mapping (address => uint) public lastUpdated; // user => last update timestamp
   29,  35: 
   30,  36:     constructor(
   31,  37:         uint _replenishmentPriceBps,
@@ -118,10 +124,10 @@ contract DolaBorrowingRights {
  118, 124:     @return uint representing the balance of the user.
  119, 125:     */
  120, 126:     function balanceOf(address user) public view returns (uint) {
- 121     :-        uint debt = debts[user];
- 122     :-        uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
- 123     :-        if(dueTokensAccrued[user] + accrued > balances[user]) return 0;
- 124     :-        return balances[user] - dueTokensAccrued[user] - accrued;
+      127:+        uint debt = userInfo[user].debts;
+      128:+        uint accrued = (block.timestamp - userInfo[user].lastUpdated) * debt / 365 days;
+      129:+        if(userInfo[user].dueTokensAccrued + accrued > userInfo[user].balances) return 0;
+      130:+        return userInfo[user].balances - userInfo[user].dueTokensAccrued - accrued;
  125, 131:     }
  126, 132: 
  127, 133:     /**
@@ -131,10 +137,10 @@ contract DolaBorrowingRights {
  131, 137:     @return uint representing the deficit of the user.
  132, 138:     */
  133, 139:     function deficitOf(address user) public view returns (uint) {
- 134     :-        uint debt = debts[user];
- 135     :-        uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
- 136     :-        if(dueTokensAccrued[user] + accrued < balances[user]) return 0;
- 137     :-        return dueTokensAccrued[user] + accrued - balances[user];
+      140:+        uint debt = userInfo[user].debts;
+      141:+        uint accrued = (block.timestamp - userInfo[user].lastUpdated) * debt / 365 days;
+      142:+        if(userInfo[user].dueTokensAccrued + accrued < userInfo[user].balances) return 0;
+      143:+        return userInfo[user].dueTokensAccrued + accrued - userInfo[user].balances;
  138, 144:     }
  139, 145:     
  140, 146:     /**
@@ -144,9 +150,9 @@ contract DolaBorrowingRights {
  144, 150:     @return Returns a signed int of the user's balance
  145, 151:     */
  146, 152:     function signedBalanceOf(address user) public view returns (int) {
- 147     :-        uint debt = debts[user];
- 148     :-        uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
- 149     :-        return int(balances[user]) - int(dueTokensAccrued[user]) - int(accrued);
+      153:+        uint debt = userInfo[user].debts;
+      154:+        uint accrued = (block.timestamp - userInfo[user].lastUpdated) * debt / 365 days;
+      155:+        return int(userInfo[user].balances) - int(userInfo[user].dueTokensAccrued) - int(accrued);
  150, 156:     }
  151, 157: 
  152, 158:     /**
@@ -169,9 +175,9 @@ contract DolaBorrowingRights {
  169, 175:     */
  170, 176:     function transfer(address to, uint256 amount) public virtual returns (bool) {
  171, 177:         require(balanceOf(msg.sender) >= amount, "Insufficient balance");
- 172     :-        balances[msg.sender] -= amount;
+      178:+        userInfo[msg.sender].balances -= amount;
  173, 179:         unchecked {
- 174     :-            balances[to] += amount;
+      180:+            userInfo[to].balances += amount;
  175, 181:         }
  176, 182:         emit Transfer(msg.sender, to, amount);
  177, 183:         return true;
@@ -193,9 +199,9 @@ contract DolaBorrowingRights {
  193, 199:         uint256 allowed = allowance[from][msg.sender];
  194, 200:         if (allowed != type(uint256).max) allowance[from][msg.sender] = allowed - amount;
  195, 201:         require(balanceOf(from) >= amount, "Insufficient balance");
- 196     :-        balances[from] -= amount;
+      202:+        userInfo[from].balances -= amount;
  197, 203:         unchecked {
- 198     :-            balances[to] += amount;
+      204:+            userInfo[to].balances += amount;
  199, 205:         }
  200, 206:         emit Transfer(from, to, amount);
  201, 207:         return true;
@@ -236,7 +242,7 @@ contract DolaBorrowingRights {
  236, 242:                                 owner,
  237, 243:                                 spender,
  238, 244:                                 value,
- 239     :-                                nonces[owner]++,
+      245:+                                userInfo[owner].nonce++,
  240, 246:                                 deadline
  241, 247:                             )
  242, 248:                         )
@@ -256,7 +262,7 @@ contract DolaBorrowingRights {
  256, 262:     @notice Function for invalidating the nonce of a signed message.
  257, 263:     */
  258, 264:     function invalidateNonce() public {
- 259     :-        nonces[msg.sender]++;
+      265:+        userInfo[msg.sender].nonce++;
  260, 266:     }
  261, 267: 
  262, 268:     function DOMAIN_SEPARATOR() public view virtual returns (bytes32) {
@@ -282,12 +288,12 @@ contract DolaBorrowingRights {
  282, 288:     @param user The address of the user to accrue DBR debt to.
  283, 289:     */
  284, 290:     function accrueDueTokens(address user) public {
- 285     :-        uint debt = debts[user];
- 286     :-        if(lastUpdated[user] == block.timestamp) return;
- 287     :-        uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
- 288     :-        dueTokensAccrued[user] += accrued;
+      291:+        uint debt = userInfo[user].debts;
+      292:+        if(userInfo[user].lastUpdated == block.timestamp) return;
+      293:+        uint accrued = (block.timestamp - userInfo[user].lastUpdated) * debt / 365 days;
+      294:+        userInfo[user].dueTokensAccrued += accrued;
  289, 295:         totalDueTokensAccrued += accrued;
- 290     :-        lastUpdated[user] = block.timestamp;
+      296:+        userInfo[user].lastUpdated = block.timestamp;
  291, 297:         emit Transfer(user, address(0), accrued);
  292, 298:     }
  293, 299: 
@@ -301,7 +307,7 @@ contract DolaBorrowingRights {
  301, 307:         require(markets[msg.sender], "Only markets can call onBorrow");
  302, 308:         accrueDueTokens(user);
  303, 309:         require(deficitOf(user) == 0, "DBR Deficit");
- 304     :-        debts[user] += additionalDebt;
+      310:+        userInfo[user].debts += additionalDebt;
  305, 311:     }
  306, 312: 
  307, 313:     /**
@@ -313,7 +319,7 @@ contract DolaBorrowingRights {
  313, 319:     function onRepay(address user, uint repaidDebt) public {
  314, 320:         require(markets[msg.sender], "Only markets can call onRepay");
  315, 321:         accrueDueTokens(user);
- 316     :-        debts[user] -= repaidDebt;
+      322:+        userInfo[user].debts -= repaidDebt;
  317, 323:     }
  318, 324: 
  319, 325:     /**
@@ -329,7 +335,7 @@ contract DolaBorrowingRights {
  329, 335:         require(deficit >= amount, "Amount > deficit");
  330, 336:         uint replenishmentCost = amount * replenishmentPriceBps / 10000;
  331, 337:         accrueDueTokens(user);
- 332     :-        debts[user] += replenishmentCost;
+      338:+        userInfo[user].debts += replenishmentCost;
  333, 339:         _mint(user, amount);
  334, 340:     }
  335, 341: 
@@ -359,7 +365,7 @@ contract DolaBorrowingRights {
  359, 365:     function _mint(address to, uint256 amount) internal virtual {
  360, 366:         _totalSupply += amount;
  361, 367:         unchecked {
- 362     :-            balances[to] += amount;
+      368:+            userInfo[to].balances += amount;
  363, 369:         }
  364, 370:         emit Transfer(address(0), to, amount);
  365, 371:     }
@@ -371,7 +377,7 @@ contract DolaBorrowingRights {
  371, 377:     */
  372, 378:     function _burn(address from, uint256 amount) internal virtual {
  373, 379:         require(balanceOf(from) >= amount, "Insufficient balance");
- 374     :-        balances[from] -= amount;
+      380:+        userInfo[from].balances -= amount;
  375, 381:         unchecked {
  376, 382:             _totalSupply -= amount;
  377, 383:         }
diff --git a/src/test/DBR.t.sol b/src/test/DBR.t.sol
index 3988cf7..754bf7f 100644
--- a/src/test/DBR.t.sol
+++ b/src/test/DBR.t.sol
@@ -145,17 +145,19 @@ contract DBRTest is FiRMTest {
  145, 145:     }
  146, 146: 
  147, 147:     function test_invalidateNonce() public {
- 148     :-        assertEq(dbr.nonces(user), 0, "User nonce should be uninitialized");
+      148:+        (, uint256 nonce,,,) = dbr.userInfo(user);
+      149:+        assertEq(nonce, 0, "User nonce should be uninitialized");
  149, 150: 
  150, 151:         vm.startPrank(user);
  151, 152:         dbr.invalidateNonce();
  152, 153: 
- 153     :-        assertEq(dbr.nonces(user), 1, "User nonce was not invalidated");
+      154:+        (,nonce,,,) = dbr.userInfo(user);
+      155:+        assertEq(nonce, 1, "User nonce was not invalidated");
  154, 156:     }
  155, 157: 
  156, 158:     function test_approve_increasesAllowanceByAmount() public {
  157, 159:         uint amount = 100e18;
- 158     :-
+      160:+        
  159, 161:         assertEq(dbr.allowance(user, gov), 0, "Allowance should not be set yet");
  160, 162: 
  161, 163:         vm.startPrank(user);
```

---

#### 5. **Expression can be unchecked when overflow is not possible (6 instances)**

Deployment. Gas Saved: **20 220**

Minimum Method Call. Gas Saved: **410**

Average Method Call. Gas Saved: **4 630**

Maximum Method Call. Gas Saved: **1 354**

Overall gas change: **-6 233 (-5.326%)**

##### - src/DBR.sol:[110](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L110), [124](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L124), [137](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L137), [259](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L259)

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..0781c97 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -107,8 +107,10 @@ contract DolaBorrowingRights {
  107, 107:     @return uint representing the total supply of DBR.
  108, 108:     */
  109, 109:     function totalSupply() public view returns (uint) {
- 110     :-        if(totalDueTokensAccrued > _totalSupply) return 0;
- 111     :-        return _totalSupply - totalDueTokensAccrued;
+      110:+        unchecked {
+      111:+            if(totalDueTokensAccrued > _totalSupply) return 0;
+      112:+            return _totalSupply - totalDueTokensAccrued;
+      113:+        }
  112, 114:     }
  113, 115: 
  114, 116:     /**
@@ -121,7 +123,7 @@ contract DolaBorrowingRights {
  121, 123:         uint debt = debts[user];
  122, 124:         uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
  123, 125:         if(dueTokensAccrued[user] + accrued > balances[user]) return 0;
- 124     :-        return balances[user] - dueTokensAccrued[user] - accrued;
+      126:+        unchecked { return balances[user] - dueTokensAccrued[user] - accrued; }
  125, 127:     }
  126, 128: 
  127, 129:     /**
@@ -134,7 +136,7 @@ contract DolaBorrowingRights {
  134, 136:         uint debt = debts[user];
  135, 137:         uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
  136, 138:         if(dueTokensAccrued[user] + accrued < balances[user]) return 0;
- 137     :-        return dueTokensAccrued[user] + accrued - balances[user];
+      139:+        unchecked { return dueTokensAccrued[user] + accrued - balances[user]; }
  138, 140:     }
  139, 141:     
  140, 142:     /**
@@ -256,7 +258,7 @@ contract DolaBorrowingRights {
  256, 258:     @notice Function for invalidating the nonce of a signed message.
  257, 259:     */
  258, 260:     function invalidateNonce() public {
- 259     :-        nonces[msg.sender]++;
+      261:+        unchecked { nonces[msg.sender]++; }
  260, 262:     }
  261, 263: 
  262, 264:     function DOMAIN_SEPARATOR() public view virtual returns (bytes32) {
```

##### - src/Fed.sol:[124](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Fed.sol#L124)

```diff
diff --git a/src/Fed.sol b/src/Fed.sol
index 1e819bb..b57b444 100644
--- a/src/Fed.sol
+++ b/src/Fed.sol
@@ -121,7 +121,7 @@ contract Fed {
  121, 121:         uint marketValue = dola.balanceOf(address(market)) + market.totalDebt();
  122, 122:         uint supply = supplies[market];
  123, 123:         if(supply >= marketValue) return 0;
- 124     :-        return marketValue - supply;
+      124:+        unchecked { return marketValue - supply; }
  125, 125:     }
  126, 126: 
  127, 127:     /**
```

##### - src/Market.sol:[521](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L521)

```diff
diff --git a/src/Market.sol b/src/Market.sol
index 9585b85..293bbb6 100644
--- a/src/Market.sol
+++ b/src/Market.sol
@@ -518,7 +518,7 @@ contract Market {
  518, 518:     @notice Function for incrementing the nonce of the msg.sender, making their latest signed message unusable.
  519, 519:     */
  520, 520:     function invalidateNonce() public {
- 521     :-        nonces[msg.sender]++;
+      521:+        unchecked { nonces[msg.sender]++; }
  522, 522:     }
  523, 523:     
  524, 524:     /**
```

---

#### 6. **State variables can be packed into fewer storage slots (1 instance)**

Deployment. Gas Saved: **-5 008**

Minimum Method Call. Gas Saved: **1 911**

Average Method Call. Gas Saved: **15 525**

Maximum Method Call. Gas Saved: **20 972**

Overall gas change: **-62 419 (-69.524%)**

If variables occupying the same slot are both written the same function or by the constructor, avoids a separate Gsset (20000 gas). Reads of the variables can also be cheaper

uint256(32), mapping(32), address(20), bool(1)

##### - src/Market.sol:[53](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L53)

```diff
diff --git a/src/Market.sol b/src/Market.sol
index 9585b85..6141e5c 100644
--- a/src/Market.sol
+++ b/src/Market.sol
@@ -36,6 +36,7 @@ interface IBorrowController {
   36,  36: contract Market {
   37,  37: 
   38,  38:     address public gov;
+       39:+    bool public borrowPaused;
   39,  40:     address public lender;
   40,  41:     address public pauseGuardian;
   41,  42:     address public immutable escrowImplementation;
@@ -50,7 +51,6 @@ contract Market {
   50,  51:     uint public liquidationFeeBps;
   51,  52:     uint public liquidationFactorBps = 5000; // 50% by default
   52,  53:     bool immutable callOnDepositCallback;
-  53     :-    bool public borrowPaused;
   54,  54:     uint public totalDebt;
   55,  55:     uint256 internal immutable INITIAL_CHAIN_ID;
   56,  56:     bytes32 internal immutable INITIAL_DOMAIN_SEPARATOR;
```

---

#### 7. **refactoring similar statements (1 instance)**

Deployment. Gas Saved: **18 422**

Minimum Method Call. Gas Saved: **-18**

Average Method Call. Gas Saved: **-11**

Maximum Method Call. Gas Saved: **6**

Overall gas change: **4 876 (7.739%)**

##### - src/Market.sol:[213](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L213)

```diff
diff --git a/src/Market.sol b/src/Market.sol
index 9585b85..da295e5 100644
--- a/src/Market.sol
+++ b/src/Market.sol
@@ -210,11 +210,9 @@ contract Market {
  210, 210:     @param _value Boolean representing the state pause state of borrows. true = paused, false = unpaused.
  211, 211:     */
  212, 212:     function pauseBorrows(bool _value) public {
- 213     :-        if(_value) {
- 214     :-            require(msg.sender == pauseGuardian || msg.sender == gov, "Only pause guardian or governance can pause");
- 215     :-        } else {
- 216     :-            require(msg.sender == gov, "Only governance can unpause");
- 217     :-        }
+      213:+        require(
+      214:+            ( _value && msg.sender == pauseGuardian) || msg.sender == gov,
+      215:+            "Only pause guardian or governance can pause");
  218, 216:         borrowPaused = _value;
  219, 217:     }
  220, 218: 
diff --git a/src/test/Market.t.sol b/src/test/Market.t.sol
index 8992ab9..86af449 100644
--- a/src/test/Market.t.sol
+++ b/src/test/Market.t.sol
@@ -16,7 +16,7 @@ import "./mocks/BorrowContract.sol";
   16,  16: import {EthFeed} from "./mocks/EthFeed.sol";
   17,  17: 
   18,  18: contract MarketTest is FiRMTest {
-  19     :-    bytes onlyGovUnpause = "Only governance can unpause";
+       19:+    bytes onlyGovUnpause = "Only pause guardian or governance can pause";
   20,  20:     bytes onlyPauseGuardianOrGov = "Only pause guardian or governance can pause";
   21,  21: 
   22,  22:     BorrowContract borrowContract;
```

---

#### 8. **Better algorithm for underflow check (3 instances)**

Deployment. Gas Saved: **12 613**

Minimum Method Call. Gas Saved: **656**

Average Method Call. Gas Saved: **8 332**

Maximum Method Call. Gas Saved: **3 741**

Overall gas change: **-18 048 (-15.981%)**

##### - src/DBR.sol:[110](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L110), [123](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L123), [136](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L136)

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..bff9fef 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -104,37 +104,39 @@ contract DolaBorrowingRights {
  104, 104:     /**
  105, 105:     @notice Get the total supply of DBR tokens.
  106, 106:     @dev The total supply is calculated as the difference between total DBR minted and total DBR accrued.
- 107     :-    @return uint representing the total supply of DBR.
+      107:+    @return ret uint representing the total supply of DBR.
  108, 108:     */
- 109     :-    function totalSupply() public view returns (uint) {
- 110     :-        if(totalDueTokensAccrued > _totalSupply) return 0;
- 111     :-        return _totalSupply - totalDueTokensAccrued;
+      109:+    function totalSupply() public view returns (uint ret) {
+      110:+        unchecked { ret = _totalSupply - totalDueTokensAccrued; }
+      111:+        if(ret > _totalSupply) return 0;
  112, 112:     }
  113, 113: 
  114, 114:     /**
  115, 115:     @notice Get the DBR balance of an address. Will return 0 if the user has zero DBR or a deficit.
  116, 116:     @dev The balance of a user is calculated as the difference between the user's balance and the user's accrued DBR debt + due DBR debt.
  117, 117:     @param user Address of the user.
- 118     :-    @return uint representing the balance of the user.
+      118:+    @return ret uint representing the balance of the user.
  119, 119:     */
- 120     :-    function balanceOf(address user) public view returns (uint) {
+      120:+    function balanceOf(address user) public view returns (uint ret) {
  121, 121:         uint debt = debts[user];
  122, 122:         uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
- 123     :-        if(dueTokensAccrued[user] + accrued > balances[user]) return 0;
- 124     :-        return balances[user] - dueTokensAccrued[user] - accrued;
+      123:+        uint mid = dueTokensAccrued[user] + accrued;
+      124:+        unchecked { ret = balances[user] - mid; }
+      125:+        if(ret > balances[user]) return 0;
  125, 126:     }
  126, 127: 
  127, 128:     /**
  128, 129:     @notice Get the DBR deficit of an address. Will return 0 if th user has zero DBR or more.
  129, 130:     @dev The deficit of a user is calculated as the difference between the user's accrued DBR deb + due DBR debt and their balance.
  130, 131:     @param user Address of the user.
- 131     :-    @return uint representing the deficit of the user.
+      132:+    @return ret uint representing the deficit of the user.
  132, 133:     */
- 133     :-    function deficitOf(address user) public view returns (uint) {
+      134:+    function deficitOf(address user) public view returns (uint ret) {
  134, 135:         uint debt = debts[user];
  135, 136:         uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
- 136     :-        if(dueTokensAccrued[user] + accrued < balances[user]) return 0;
- 137     :-        return dueTokensAccrued[user] + accrued - balances[user];
+      137:+        uint mid = dueTokensAccrued[user] + accrued;
+      138:+        unchecked { ret = mid - balances[user]; }
+      139:+        if(mid < ret) return 0;
  138, 140:     }
  139, 141:     
  140, 142:     /**
```

---

#### 9. **`x = x + y` is cheaper than `x += y` (12 instances)**

Deployment. Gas Saved: **11 214**

Minimum Method Call. Gas Saved: **180**

Average Method Call. Gas Saved: **468**

Maximum Method Call. Gas Saved: **616**

Overall gas change: **-5 325 (-1.318%)**

##### - src/DBR.sol:[174](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L174), [196](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L196), [289](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L289), [360](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L360), [362](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L362), [376](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L376)

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..c02b782 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -171,7 +171,7 @@ contract DolaBorrowingRights {
  171, 171:         require(balanceOf(msg.sender) >= amount, "Insufficient balance");
  172, 172:         balances[msg.sender] -= amount;
  173, 173:         unchecked {
- 174     :-            balances[to] += amount;
+      174:+            balances[to] = balances[to] + amount;
  175, 175:         }
  176, 176:         emit Transfer(msg.sender, to, amount);
  177, 177:         return true;
@@ -193,7 +193,7 @@ contract DolaBorrowingRights {
  193, 193:         uint256 allowed = allowance[from][msg.sender];
  194, 194:         if (allowed != type(uint256).max) allowance[from][msg.sender] = allowed - amount;
  195, 195:         require(balanceOf(from) >= amount, "Insufficient balance");
- 196     :-        balances[from] -= amount;
+      196:+        balances[from] = balances[from] - amount;
  197, 197:         unchecked {
  198, 198:             balances[to] += amount;
  199, 199:         }
@@ -286,7 +286,7 @@ contract DolaBorrowingRights {
  286, 286:         if(lastUpdated[user] == block.timestamp) return;
  287, 287:         uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
  288, 288:         dueTokensAccrued[user] += accrued;
- 289     :-        totalDueTokensAccrued += accrued;
+      289:+        totalDueTokensAccrued = totalDueTokensAccrued + accrued;
  290, 290:         lastUpdated[user] = block.timestamp;
  291, 291:         emit Transfer(user, address(0), accrued);
  292, 292:     }
@@ -357,9 +357,9 @@ contract DolaBorrowingRights {
  357, 357:     @param amount Amount of DBR to mint.
  358, 358:     */
  359, 359:     function _mint(address to, uint256 amount) internal virtual {
- 360     :-        _totalSupply += amount;
+      360:+        _totalSupply = _totalSupply + amount;
  361, 361:         unchecked {
- 362     :-            balances[to] += amount;
+      362:+            balances[to] = balances[to] + amount;
  363, 363:         }
  364, 364:         emit Transfer(address(0), to, amount);
  365, 365:     }
@@ -373,7 +373,7 @@ contract DolaBorrowingRights {
  373, 373:         require(balanceOf(from) >= amount, "Insufficient balance");
  374, 374:         balances[from] -= amount;
  375, 375:         unchecked {
- 376     :-            _totalSupply -= amount;
+      376:+            _totalSupply = _totalSupply - amount;
  377, 377:         }
  378, 378:         emit Transfer(from, address(0), amount);
  379, 379:     }
```

##### - src/Market.sol:[395](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L395), [397](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L397), [535](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L535), [568](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L568), [598](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L598), [600](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L600)

```diff
diff --git a/src/Market.sol b/src/Market.sol
index 9585b85..bc0ff93 100644
--- a/src/Market.sol
+++ b/src/Market.sol
@@ -392,9 +392,9 @@ contract Market {
  392, 392:             require(borrowController.borrowAllowed(msg.sender, borrower, amount), "Denied by borrow controller");
  393, 393:         }
  394, 394:         uint credit = getCreditLimitInternal(borrower);
- 395     :-        debts[borrower] += amount;
+      395:+        debts[borrower] = debts[borrower] + amount;
  396, 396:         require(credit >= debts[borrower], "Exceeded credit limit");
- 397     :-        totalDebt += amount;
+      397:+        totalDebt = totalDebt + amount;
  398, 398:         dbr.onBorrow(borrower, amount);
  399, 399:         dola.transfer(to, amount);
  400, 400:         emit Borrow(borrower, amount);
@@ -532,7 +532,7 @@ contract Market {
  532, 532:         uint debt = debts[user];
  533, 533:         require(debt >= amount, "Insufficient debt");
  534, 534:         debts[user] -= amount;
- 535     :-        totalDebt -= amount;
+      535:+        totalDebt = totalDebt - amount;
  536, 536:         dbr.onRepay(user, amount);
  537, 537:         dola.transferFrom(msg.sender, address(this), amount);
  538, 538:         emit Repay(user, msg.sender, amount);
@@ -565,7 +565,7 @@ contract Market {
  565, 565:         debts[user] += replenishmentCost;
  566, 566:         uint collateralValue = getCollateralValueInternal(user);
  567, 567:         require(collateralValue >= debts[user], "Exceeded collateral value");
- 568     :-        totalDebt += replenishmentCost;
+      568:+        totalDebt = totalDebt + replenishmentCost;
  569, 569:         dbr.onForceReplenish(user, amount);
  570, 570:         dola.transfer(msg.sender, replenisherReward);
  571, 571:         emit ForceReplenish(user, msg.sender, amount, replenishmentCost, replenisherReward);
@@ -595,9 +595,9 @@ contract Market {
  595, 595:         require(repaidDebt <= debt * liquidationFactorBps / 10000, "Exceeded liquidation factor");
  596, 596:         uint price = oracle.getPrice(address(collateral), collateralFactorBps);
  597, 597:         uint liquidatorReward = repaidDebt * 1 ether / price;
- 598     :-        liquidatorReward += liquidatorReward * liquidationIncentiveBps / 10000;
+      598:+        liquidatorReward = liquidatorReward + liquidatorReward * liquidationIncentiveBps / 10000;
  599, 599:         debts[user] -= repaidDebt;
- 600     :-        totalDebt -= repaidDebt;
+      600:+        totalDebt = totalDebt - repaidDebt;
  601, 601:         dbr.onRepay(user, repaidDebt);
  602, 602:         dola.transferFrom(msg.sender, address(this), repaidDebt);
  603, 603:         IEscrow escrow = predictEscrow(user);
```

---

#### 10. **`internal` functions only called once can be inlined to save gas (1 instance)**

Deployment. Gas Saved: **5 207**

Minimum Method Call. Gas Saved: **67**

Average Method Call. Gas Saved: **47**

Maximum Method Call. Gas Saved: **24**

Overall gas change: -137 (-0.154%)

##### - src/DBR.sol:[341](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L341)

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..a357f92 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -338,7 +338,12 @@ contract DolaBorrowingRights {
  338, 338:     @param amount Amount to be burned
  339, 339:     */
  340, 340:     function burn(uint amount) public {
- 341     :-        _burn(msg.sender, amount);
+      341:+        require(balanceOf(msg.sender) >= amount, "Insufficient balance");
+      342:+        balances[msg.sender] -= amount;
+      343:+        unchecked {
+      344:+            _totalSupply -= amount;
+      345:+        }
+      346:+        emit Transfer(msg.sender, address(0), amount);
  342, 347:     }
  343, 348: 
  344, 349:     /**
@@ -364,20 +369,6 @@ contract DolaBorrowingRights {
  364, 369:         emit Transfer(address(0), to, amount);
  365, 370:     }
  366, 371: 
- 367     :-    /**
- 368     :-    @notice Internal function for burning DBR.
- 369     :-    @param from Address to burn DBR from.
- 370     :-    @param amount Amount of DBR to be burned.
- 371     :-    */
- 372     :-    function _burn(address from, uint256 amount) internal virtual {
- 373     :-        require(balanceOf(from) >= amount, "Insufficient balance");
- 374     :-        balances[from] -= amount;
- 375     :-        unchecked {
- 376     :-            _totalSupply -= amount;
- 377     :-        }
- 378     :-        emit Transfer(from, address(0), amount);
- 379     :-    }
- 380     :-
  381, 372:     event Transfer(address indexed from, address indexed to, uint256 amount);
  382, 373:     event Approval(address indexed owner, address indexed spender, uint256 amount);
  383, 374:     event AddMinter(address indexed minter);
```

---

#### 11. **State variables should be cached in stack variables rather than re-reading them from storage (2 instances)**

Deployment. Gas Saved: **5 007**

Minimum Method Call. Gas Saved: **478**

Average Method Call. Gas Saved: **1 117**

Maximum Method Call. Gas Saved: **1 423**

Overall gas change: **-6 231 (-1.618%)**

##### - src/DBR.sol:[286](https://github.com/code-423n4/2022-10-inverse/blob/main/src/DBR.sol#L286)

```diff
diff --git a/src/DBR.sol b/src/DBR.sol
index aab6daf..c70fcd7 100644
--- a/src/DBR.sol
+++ b/src/DBR.sol
@@ -283,8 +283,9 @@ contract DolaBorrowingRights {
  283, 283:     */
  284, 284:     function accrueDueTokens(address user) public {
  285, 285:         uint debt = debts[user];
- 286     :-        if(lastUpdated[user] == block.timestamp) return;
- 287     :-        uint accrued = (block.timestamp - lastUpdated[user]) * debt / 365 days;
+      286:+        uint _lastUpdated = lastUpdated[user];
+      287:+        if(_lastUpdated == block.timestamp) return;
+      288:+        uint accrued = (block.timestamp - _lastUpdated) * debt / 365 days;
  288, 289:         dueTokensAccrued[user] += accrued;
  289, 290:         totalDueTokensAccrued += accrued;
  290, 291:         lastUpdated[user] = block.timestamp;
```

##### - src/Market.sol:[391](https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L391)

```diff
diff --git a/src/Market.sol b/src/Market.sol
index 9585b85..5f3264d 100644
--- a/src/Market.sol
+++ b/src/Market.sol
@@ -388,8 +388,9 @@ contract Market {
  388, 388:     */
  389, 389:     function borrowInternal(address borrower, address to, uint amount) internal {
  390, 390:         require(!borrowPaused, "Borrowing is paused");
- 391     :-        if(borrowController != IBorrowController(address(0))) {
- 392     :-            require(borrowController.borrowAllowed(msg.sender, borrower, amount), "Denied by borrow controller");
+      391:+        IBorrowController _borrowController = borrowController;
+      392:+        if(_borrowController != IBorrowController(address(0))) {
+      393:+            require(_borrowController.borrowAllowed(msg.sender, borrower, amount), "Denied by borrow controller");
  393, 394:         }
  394, 395:         uint credit = getCreditLimitInternal(borrower);
  395, 396:         debts[borrower] += amount;
```

---

#### 99. **Overall gas savings**

Deployment. Gas Saved: **416 802**

Minimum Method Call. Gas Saved: **3 423**

Average Method Call. Gas Saved: **15 773**

Maximum Method Call. Gas Saved: **18 283**

Overall gas change: **-84 866 (-67.204%)**

```diff
diff --git a/original.txt b/fixed.txt
index 15fe5a1..d8a52d4 100644
--- a/original.txt
+++ b/fixed.txt
 ╭────────────────────────────────────────────────────┬─────────────────┬───────┬────────┬───────┬─────────╮
 │ src/BorrowController.sol:BorrowController contract ┆                 ┆       ┆        ┆       ┆         │
 ╞════════════════════════════════════════════════════╪═════════════════╪═══════╪════════╪═══════╪═════════╡
 │ Deployment Cost                                    ┆ Deployment Size ┆       ┆        ┆       ┆         │
 ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
-│ 180951                                             ┆ 971             ┆       ┆        ┆       ┆         │
+│ 166939                                             ┆ 901             ┆       ┆        ┆       ┆         │
 ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
 │ Function Name                                      ┆ min             ┆ avg   ┆ median ┆ max   ┆ # calls │
 ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
-│ allow                                              ┆ 664             ┆ 19132 ┆ 22750  ┆ 24750 ┆ 5       │
+│ allow                                              ┆ 646             ┆ 19148 ┆ 22774  ┆ 24774 ┆ 5       │
 ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
 │ borrowAllowed                                      ┆ 624             ┆ 1761  ┆ 1807   ┆ 2807  ┆ 4       │
 ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
 │ contractAllowlist                                  ┆ 553             ┆ 1353  ┆ 553    ┆ 2553  ┆ 5       │
 ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
-│ deny                                               ┆ 576             ┆ 1972  ┆ 596    ┆ 4744  ┆ 3       │
+│ deny                                               ┆ 558             ┆ 1980  ┆ 615    ┆ 4768  ┆ 3       │
 ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
... See the rest this report [here](https://github.com/code-423n4/2022-10-inverse-findings/blob/main/data/pfapostol-G.md)