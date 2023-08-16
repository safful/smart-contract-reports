## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [reordering struct fields](https://github.com/code-423n4/2021-11-nested-findings/issues/96) 

# Handle

pants


# Vulnerability details

NestedRecords line 22 - you can save storage by reordering Holding struct fields in the following way:

original:
    struct Holding {
        address token;
        uint256 amount;
        bool isActive;
    }

change to:

    struct Holding {
        uint256 amount;
        address token;
        bool isActive;
    }


