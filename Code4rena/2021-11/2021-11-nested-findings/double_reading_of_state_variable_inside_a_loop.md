## Tags

- bug
- question
- G (Gas Optimization)
- sponsor confirmed

# [double reading of state variable inside a loop](https://github.com/code-423n4/2021-11-nested-findings/issues/98) 

# Handle

pants


# Vulnerability details

MixinOperatorResolver.rebuildCache (addressCache[name]), isResolverCached (addressCache[name])

You can cache the value after the first read into a local variable to save the other SLOAD and also the "out of bounds" check.

