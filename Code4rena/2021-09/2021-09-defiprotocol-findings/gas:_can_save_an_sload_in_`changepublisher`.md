## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Can save an sload in `changePublisher`](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/227) 

# Handle

cmichel


# Vulnerability details

The `if`-branch of `Basket.changePublisher` function ensures that `pendingPublisher.publisher == newPublisher` which means setting `publisher = newPublisher` is equivalent to `publisher = pendingPublisher.publisher` but the former saves an expensive storage load operation.

