## Tags

- bug
- disagree with severity
- 3 (High Risk)
- sponsor confirmed
- filed

# [Vault rewards last claim time not always initialized](https://github.com/code-423n4/2021-04-vader-findings/issues/223) 

# Handle

@cmichelio


# Vulnerability details


## Vulnerability Details

The `harvest` calls `calcCurrentReward` which computes `_secondsSinceClaim = block.timestamp - mapMemberSynth_lastTime[member][synth];`.
As one can claim different synths than the synths that they deposited, `mapMemberSynth_lastTime[member][synth]` might still be uninitialized and the `_secondsSinceClaim` becomes the current block timestamp.

## Impact

The larger the `_secondsSinceClaim` the larger the rewards.
This bug allows claiming a huge chunk of the rewards.

## Recommended Mitigation Steps

Let users only harvest synths that they deposited.


