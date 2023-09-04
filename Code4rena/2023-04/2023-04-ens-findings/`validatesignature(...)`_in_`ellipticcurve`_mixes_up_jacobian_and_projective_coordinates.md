## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-06

# [`validateSignature(...)` in `EllipticCurve` mixes up Jacobian and projective coordinates](https://github.com/code-423n4/2023-04-ens-findings/issues/180) 

# Lines of code

https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/dnssec-oracle/algorithms/EllipticCurve.sol#L415


# Vulnerability details

## Impact
Currently not exploitable because this bug is cancelled out by another issue (see my Gas report). If the other issue is fixed `validateSignature` will return completely incorrect values.

## Details

In https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/dnssec-oracle/algorithms/EllipticCurve.sol#L415 `validateSignature` converts to affine coordinates from Jacobian coordinates, i.e. $X_a = X_j \cdot (Z_j^{-1})^2$. However, the inputs from the previous computation https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/dnssec-oracle/algorithms/EllipticCurve.sol#L408 are actually projective coordinates and the correct conversion formula is $X_a = X_p \cdot Z_p^{-1}$. This has been working so far only because the `EllipticCurve` performs a redundant chain of immediate conversions projective->affine->projective->affine and so during that last conversion $Z = 1$. Should the chain of redundant conversions be fixed, `validateSignature` will no longer work correctly.

## Tools Used

Manual review

## Recommended Mitigation Steps

To just fix this bug:

```diff
diff --git a/contracts/dnssec-oracle/algorithms/EllipticCurve.sol b/contracts/dnssec-oracle/algorithms/EllipticCurve.sol
index 6861264..ea7e865 100644
--- a/contracts/dnssec-oracle/algorithms/EllipticCurve.sol
+++ b/contracts/dnssec-oracle/algorithms/EllipticCurve.sol
@@ -412,7 +412,7 @@ contract EllipticCurve {
         }
 
         uint256 Px = inverseMod(P[2], p);
-        Px = mulmod(P[0], mulmod(Px, Px, p), p);
+        Px = mulmod(P[0], Px, p);
 
         return Px % n == rs[0];
     }

```

Or to fix this bug and optimize out the redundant conversions chain:
```diff
diff --git a/contracts/dnssec-oracle/algorithms/EllipticCurve.sol b/contracts/dnssec-oracle/algorithms/EllipticCurve.sol
index 6861264..8568be2 100644
--- a/contracts/dnssec-oracle/algorithms/EllipticCurve.sol
+++ b/contracts/dnssec-oracle/algorithms/EllipticCurve.sol
@@ -405,14 +405,13 @@ contract EllipticCurve {
         uint256 sInv = inverseMod(rs[1], n);
         (x1, y1) = multiplyScalar(gx, gy, mulmod(uint256(message), sInv, n));
         (x2, y2) = multiplyScalar(Q[0], Q[1], mulmod(rs[0], sInv, n));
-        uint256[3] memory P = addAndReturnProjectivePoint(x1, y1, x2, y2);
+        (uint256 Px,, uint256 Pz) = addProj(x1, y1, 1, x2, y2, 1);
 
-        if (P[2] == 0) {
+        if (Pz == 0) {
             return false;
         }
 
-        uint256 Px = inverseMod(P[2], p);
-        Px = mulmod(P[0], mulmod(Px, Px, p), p);
+        Px = mulmod(Px, inverseMod(Pz, p), p);
 
         return Px % n == rs[0];
     }
```