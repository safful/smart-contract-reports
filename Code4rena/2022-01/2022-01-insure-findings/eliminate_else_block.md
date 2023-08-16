## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Eliminate else block](https://github.com/code-423n4/2022-01-insure-findings/issues/303) 

# Handle

pauliax


# Vulnerability details

## Impact
You dont need this else block, code can be refactored from this:
```solidity
  if (address(controller) != address(0)) {
      controller.migrate(address(_controller));
      controller = IController(_controller);
  } else {
      controller = IController(_controller);
  }
```
to this:
```solidity
  if (address(controller) != address(0)) {
      controller.migrate(address(_controller));
  }
  controller = IController(_controller);
```

