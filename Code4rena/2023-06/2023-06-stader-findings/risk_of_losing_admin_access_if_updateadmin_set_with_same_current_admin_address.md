## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-01

# [Risk of losing admin access if updateAdmin set with same current admin address](https://github.com/code-423n4/2023-06-stader-findings/issues/390) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L176-L183


# Vulnerability details

> N.B : This bug is different that the other one titled "The admin address used in initialize function, can behave maliciously". Both issues are related to access control, but the impact, root cause and bug fix are different, so DO NOT mark it as dupliate of the other one.


Current admin will lose DEFAULT_ADMIN_ROLE role if updateAdmin issued with same address.


## Impact
The is a possibility of loss of protocol admin access to this critical StaderConfig.sol contract, if updateAdmin() is set with same current admin address by mistake.


## Proof of Concept
Contract : StaderConfig.sol
Function : function updateAdmin(address _admin)

Using Brownie python automation framework commands in below examples.


* Step#1 After initialization, admin-A is the admin which has the DEFAULT_ADMIN_ROLE

* Step#2 update new Admin
  StaderConfig.updateAdmin(admin-B, {'from':admin-A})
  The value of StaderConfig.getAdmin() is admin-B

* Step#3 admin-B updates admin to itself again
  StaderConfig.updateAdmin(admin-B, {'from':admin-B})
  The value of StaderConfig.getAdmin() is admin-B, but the DEFAULT_ADMIN_ROLE is revoked due to _revokeRole(DEFAULT_ADMIN_ROLE, oldAdmin);
  Now the protocol admin control is lost for StaderConfig contract


## Recommended Mitigation Steps
Ref : https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderConfig.sol#L177
In the updateAdmin() function, add a check for oldAdmin != _admin , like below
```
    address oldAdmin = accountsMap[ADMIN];
+   require(oldAdmin != _admin, "Already set to admin");


## Assessed type

Access Control