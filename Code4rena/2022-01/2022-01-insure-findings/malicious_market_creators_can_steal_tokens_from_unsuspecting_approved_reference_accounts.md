## Tags

- bug
- 3 (High Risk)
- resolved
- sponsor confirmed

# [Malicious Market Creators Can Steal Tokens From Unsuspecting Approved Reference Accounts](https://github.com/code-423n4/2022-01-insure-findings/issues/224) 

# Handle

leastwood


# Vulnerability details

## Impact

The current method of market creation involves calling `Factory.createMarket()` with a list of approved `_conditions` and `_references` accounts. If a registered template address has `templates[address(_template)].isOpen == true`, then any user is able to call `createMarket()` using this template. If the template points to `PoolTemplate.sol`, then a malicious market creator can abuse `PoolTemplate.initialize()` as it makes a vault deposit from an account that they control. The vulnerable internal function, `_depositFrom()`, makes a vault deposit from the `_references[4]` address (arbitrarily set to an approved reference address upon market creation).

Hence, if approved `_references` accounts have set an unlimited approval amount for `Vault.sol` before deploying their market, a malicious user can frontrun market creation and cause these tokens to be transferred to the incorrect market.

This issue can cause honest market creators to have their tokens transferred to an incorrectly configured market, leading to unrecoverable funds. If their approval to `Vault.sol` was set to the unlimited amount, malicious users will also be able to force honest market creators to transfer more tokens than they would normally want to allow.
## Proof of Concept

https://github.com/code-423n4/2022-01-insure/blob/main/contracts/Factory.sol#L158-L231
```
function createMarket(
    IUniversalMarket _template,
    string memory _metaData,
    uint256[] memory _conditions,
    address[] memory _references
) public override returns (address) {
    //check eligibility
    require(
        templates[address(_template)].approval == true,
        "ERROR: UNAUTHORIZED_TEMPLATE"
    );
    if (templates[address(_template)].isOpen == false) {
        require(
            ownership.owner() == msg.sender,
            "ERROR: UNAUTHORIZED_SENDER"
        );
    }
    if (_references.length > 0) {
        for (uint256 i = 0; i < _references.length; i++) {
            require(
                reflist[address(_template)][i][_references[i]] == true ||
                    reflist[address(_template)][i][address(0)] == true,
                "ERROR: UNAUTHORIZED_REFERENCE"
            );
        }
    }

    if (_conditions.length > 0) {
        for (uint256 i = 0; i < _conditions.length; i++) {
            if (conditionlist[address(_template)][i] > 0) {
                _conditions[i] = conditionlist[address(_template)][i];
            }
        }
    }

    if (
        IRegistry(registry).confirmExistence(
            address(_template),
            _references[0]
        ) == false
    ) {
        IRegistry(registry).setExistence(
            address(_template),
            _references[0]
        );
    } else {
        if (templates[address(_template)].allowDuplicate == false) {
            revert("ERROR: DUPLICATE_MARKET");
        }
    }

    //create market
    IUniversalMarket market = IUniversalMarket(
        _createClone(address(_template))
    );

    IRegistry(registry).supportMarket(address(market));
    
    markets.push(address(market));


    //initialize
    market.initialize(_metaData, _conditions, _references);

    emit MarketCreated(
        address(market),
        address(_template),
        _metaData,
        _conditions,
        _references
    );

    return address(market);
}
```

https://github.com/code-423n4/2022-01-insure/blob/main/contracts/PoolTemplate.sol#L178-L221
```
function initialize(
    string calldata _metaData,
    uint256[] calldata _conditions,
    address[] calldata _references
) external override {
    require(
        initialized == false &&
            bytes(_metaData).length > 0 &&
            _references[0] != address(0) &&
            _references[1] != address(0) &&
            _references[2] != address(0) &&
            _references[3] != address(0) &&
            _references[4] != address(0) &&
            _conditions[0] <= _conditions[1],
        "ERROR: INITIALIZATION_BAD_CONDITIONS"
    );
    initialized = true;

    string memory _name = string(
        abi.encodePacked(
            "InsureDAO-",
            IERC20Metadata(_references[1]).name(),
            "-PoolInsurance"
        )
    );
    string memory _symbol = string(
        abi.encodePacked("i-", IERC20Metadata(_references[1]).symbol())
    );
    uint8 _decimals = IERC20Metadata(_references[0]).decimals();

    initializeToken(_name, _symbol, _decimals);

    registry = IRegistry(_references[2]);
    parameters = IParameters(_references[3]);
    vault = IVault(parameters.getVault(_references[1]));

    metadata = _metaData;

    marketStatus = MarketStatus.Trading;

    if (_conditions[1] > 0) {
        _depositFrom(_conditions[1], _references[4]);
    }
}
```

## Tools Used

Manual code review.
Discussions with kohshiba.

## Recommended Mitigation Steps

After discussions with the sponsor, they have opted to parse a `_creator` address to `PoolTemplate.sol` which will act as the depositor and be set to `msg.sender` in `Factory.createMarket()`. This will prevent malicious market creators from forcing vault deposits from unsuspecting users who are approved in `Factory.sol` and have also approved `Vault.sol` to make transfers on their behalf.

