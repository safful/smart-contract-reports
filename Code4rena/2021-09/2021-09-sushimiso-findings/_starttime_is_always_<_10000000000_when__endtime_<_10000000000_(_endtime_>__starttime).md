## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [_startTime is always < 10000000000 when _endTime < 10000000000 (_endTime > _startTime)](https://github.com/code-423n4/2021-09-sushimiso-findings/issues/127) 

# Handle

pauliax


# Vulnerability details

## Impact
No need to check that _startTime < 10000000000 as it is later checked against _endTime which is also < 10000000000 :
        require(_startTime < 10000000000, "Crowdsale: enter an unix timestamp in seconds, not miliseconds");
        require(_endTime < 10000000000, "Crowdsale: enter an unix timestamp in seconds, not miliseconds");
        ...
        require(_endTime > _startTime, "Crowdsale: end time must be older than start price");

## Recommended Mitigation Steps
Remove this line:
    require(_startTime < 10000000000, "Crowdsale: enter an unix timestamp in seconds, not miliseconds");

