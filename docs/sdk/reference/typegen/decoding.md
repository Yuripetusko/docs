---
sidebar_position: 35
description: Type-safe EVM tx, log and state data
---

# Decoding

**Since `@subsquid/evm-typegen@2.0.0`**

The `squid-evm-typegen(1)` tool generates TypeScript facades for EVM transactions, logs and `eth_call` queries.

The generated facades are assumed to be used by squids indexing EVM data.

The tool takes a JSON ABIs as an input. Those can be specified in three ways:

1. as a plain JSON file(s):

   ```bash
   npx squid-evm-typegen src/abi erc20.json
   ```
   If you use this option, you can also place your JSON ABIs to the `abi` folder and run
   ```bash
   sqd typegen
   ```
   Script is available in all EVM templates.

2. as a contract address (to fetch the ABI from Etherscan API). Once can pass multiple addresses at once.

   ```bash
   npx squid-evm-typegen src/abi 0xBB9bc244D798123fDe783fCc1C72d3Bb8C189413
   ```

:::info
Please check if your contract is a proxy when using this method. If it is, consult [this page](/evm-indexing/proxy-contracts) for guidance.
:::

3. as an arbitrary URL:

   ```bash
   npx squid-evm-typegen src/abi https://example.com/erc721.json
   ```

In all cases typegen will use `basename` of the ABI as the root name for the generated files. You can change the basename of generated files using the fragment `(#)` suffix.

```bash
squid-evm-typegen src/abi 0xBB9bc244D798123fDe783fCc1C72d3Bb8C189413#my-contract-name
```

**Arguments:**

|                        |                                                           |
|------------------------|-----------------------------------------------------------|
|  `output-dir`          | output directory for generated definitions                |
|  `abi`                 | A contract address, an URL or a local path to the ABI file. Accepts multiple contracts. |


**Options:**

|                           |                                                          |
|---------------------------|----------------------------------------------------------| 
|  `--multicall`            | generate a facade for the MakerDAO multicall contract. May significantly improve the performance of contract state calls by batching RPC requests (see below)   |
|  `--etherscan-api <url>`  | etherscan API to fetch contract ABI by a known address. By default, `https://api.etherscan.io/`   |
|  `--clean`                | delete output directory before the run                   |
|  `-h, --help`             | display help for command                                 |


## Usage

### Events 

The EVM log data is provided by the `event` object with wrappers for each event defined in the ABI.

**Example**

Subscribe to two topics:

```ts
// generated by evm-typegen
import { events } from './abi/weth'

const CONTRACT_ADDRESS = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2'.toLowerCase()

const processor = new EvmBatchProcessor()
  .setDataSource({
    archive: lookupArchive('eth-mainnet'),
  })
  .addLog({
    address: [CONTRACT_ADDRESS],
    topic0: [
      events.Deposit.topic,
      events.Withdrawal.topic
    ]
  })
```

Decode event data:
```ts
processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let log of c.logs) {
      if (log.address === CONTRACT_ADDRESS && log.topics[0] == events.Deposit.topic) {
        // type-safe decoding of the Deposit event data
        const amt = events.Deposit.decode(log).wad
      }
    }
})
```

### Transactions

Similar to `events`, transaction access is provided by the `functions` object for each contract method defined in the ABI. 

### Contract state calls

The typegen creates a wrapper `Contract` class for each contract. Create a `Contract` instance using the processor context and the block height at which the state should be queried.

**Example**

```ts
// generated by evm-typegen
import { Contract } from "./abi/weth";

const contractAddress = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2'.toLowerCase()

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    // call totalSupply() at the current block
    let blockHeader = c.header
    let supply = await (new Contract(ctx, blockHeader, contractAddress).totalSupply())
  }
})
```

### Batching contract state calls using the Multicall contract

Use the `--multicall` flag to generate the `Multicall` facade class for the [MakerDAO Multicall contract](https://github.com/makerdao/multicall). 
See [Batch state queries](/evm-indexing/query-state/#batch-state-queries) for more context.

The `Multicall` facade exposes the method
```ts
tryAggregate<Args extends any[], R>(
    func: Func<Args, {}, R>,
    calls: [address: string, args: Args][],
    paging?: number
  ): Promise<MulticallResult<R>[]>
```
The arguments are as follows:
- `func`: the contract function to be called
- `calls`: an array of tuples `[contractAddress: string, args]`. Each specified contract will be called with the specified arguments.
- `paging` an (optional) argument for the maximal number of calls to be batched into a single JSON PRC request. Note that large page sizes may cause timeouts.

A typical usage is as follows:
```ts
// generated by evm-typegen
import { functions } from "./abi/mycontract";
import { Multicall } from "./abi/multicall";

const MY_CONTRACT='0xac5c7493036de60e63eb81c5e9a440b42f47ebf5'

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    // some logic
  }
  const lastBlock = ctx.blocks[ctx.blocks.length - 1];
  // Multicall address for Ethereum is 0x5ba1e12693dc8f9c48aad8770482f4739beed696
  const multicall = new Multicall(ctx, lastBlock, '0x5ba1e12693dc8f9c48aad8770482f4739beed696')
  // call MY_CONTRACT.myContractMethod('foo') and MY_CONTRACT.myContractMethod('bar')
  const args = ['foo', 'bar']
  const results = await multicall.tryAggregate(functions.myContractMethod, args.map(a => [MY_CONTRACT, a]) as [string, any[]], 100);

  results.forEach((res, i) => {
    if (res.success) {
      ctx.log.info(`Result for argument ${args[i]} is ${res.value}`);
    }
  }) 
});
```

## Migration from `evm-typegen@1.x`

* In `evm-typegen@2.0`, the `--abi`, and `--output` flags are replaced by the positional arguments. For example, the command
  ```sh
  npx squid-evm-typegen --abi src/abi/ERC721.json --output src/abi/erc721.ts
  ```
  is replaced with
  ```sh
  npx squid-evm-typegen src/abi src/abi/ERC721.json
  ```

* The `events` object generated by `evm-typegen@2.0` exposes events by name, not by the full signature. For example,
  ```ts
  events['UpdatedGravatar(uint256,address,string,string)'].decode(evmLog)
  ```
  should be replaced with a more readable
  ```ts
  events.UpdatedGravatar.decode(evmLog)
  ```
---
sidebar_position: 50
description: Type-safe Substrate data handling
---

# Substrate typegen

**Since `@subsquid/substrate-typegen@8.0.0`**

The substrate typegen tool is a part of Subsquid SDK. It generates TypeScript wrappers for interfacing Substrate events and calls.

Usage:

```bash
npx squid-substrate-typegen typegen.json
```

Within the substrate-based templates there is also an `sqd` shorthand command for executing this exact line:

```bash
sqd typegen
```

If necessary, multiple config files can be supplied:
```bash
npx squid-substrate-typegen typegen0.json typegen1.json ...
```

The structure of the `typegen.json` config file is best illustrated with an example:
```json
{
  "outDir": "src/types",
  "specVersions": "https://v2.archive.subsquid.io/metadata/kusama",
  "pallets": {
    // add one such section for each pallet
    "Balances": {
      "events": [
        // list of events to generate wrappers for
        "Transfer"
      ],
      "calls": [
        // list of calls to generate wrappers for
        "transfer_allow_death"
      ],
      "storage": [
        "Account"
      ],
      "constants": [
        "ExistentialDeposit"
      ]
    }
  }
}
```
The `specVersions` field is either
 - a metadata service endpoint URL, like
   ```
   https://v2.archive.subsquid.io/metadata/{network}
   ```
   or
 - a path to a [`jsonl`](https://jsonlines.org) file generated by [`substrate-metadata-explorer(1)`](https://github.com/subsquid/squid-sdk/tree/master/substrate/substrate-metadata-explorer).


To generate all items defined by a given pallet, set any of the `events`, `calls`, `storage` or `constants` fields to `true`, e.g.

```json
{
  "outDir": "src/types",
  "specVersions": "kusamaVersions.jsonl",
  "pallets": {
    "Balances": {
      // generate wrappers for all Balances pallet constants
      "constants": true
    }
  }
}
```

## TypeScript wrappers

Wrappers generated by the typegen command can be found in the specified `outDir` (`src/types` by convention). Assuming that this folder is imported as `types` (e.g. with `import * as types from './types'`), you'll be able to find the wrappers at:
 - for events: `types.events.${palletName}.${eventName}`
 - for calls: `types.calls.${palletName}.${callName}`
 - for storage items: `types.storage.${palletName}.${storageItemName}`
 - for constants: `types.constants.${palletName}.${constantName}`

All identifiers (pallet name, call name etc) are lowerCamelCased. E.g. the constant ```Balances.ExistentialDeposit``` becomes ```types.events.balances.existentialDeposit``` and the call ```Balances.set_balance``` becomes ```types.calls.setBalance```.

## Examples

### Events and calls

```typescript
import {events, calls} from './types'

processor.run(new TypeormDatabase(), async ctx => {
  for (let block of ctx.blocks) {
     for (let event of block.events) {
      if (event.name == events.balances.transfer.name) {
        let rec: {from: Bytes, to: Bytes, amount: bigint}
        if (events.balances.transfer.v1020.is(event)) {
          let [from, to, amount, fee] =
            events.balances.transfer.v1020.decode(event)
          rec = {from, to, amount}
        }
        // ... decode all runtime versions similarly
        // with events.balances.transfer.${ver}.is/.decode
      }
    }
    for (let call of block.calls) {
      if (call.name == calls.balances.forceTransfer.name) {
        let rec: {source: Bytes, dest: Bytes, value: bigint} | undefined
        if (calls.balances.forceTransfer.v1020.is(call)) {
          let res =
            calls.balances.forceTransfer.v1020.decode(call)
          assert(res.source.__kind === 'AccountId')
          assert(res.dest.__kind === 'AccountId')
          rec = {
            source: res.source.value,
            dest: res.dest.value,
            value: res.value
          }
        }
        // ... decode all runtime versions similarly
        // with calls.balances.forceTransfer.${ver}.is/.decode
    }
  }
})
```

### Storage

See [Storage queries](../storage-state-calls).

### Constants

```typescript
import {constants} from './types'
// ...
processor.run(new TypeormDatabase(), async ctx => {
  for (let block of ctx.blocks) {
    if (constants.balances.existentialDeposit.v1020.is(block.header)) {
      let c = new constants.balances.existentialDeposit.v1020.get(block.header)
      ctx.log.info(`Balances.ExistentialDeposit (runtime version V1020): ${c}`)
    }
  }
})
```