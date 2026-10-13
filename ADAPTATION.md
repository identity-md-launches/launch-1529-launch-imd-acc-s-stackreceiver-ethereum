# StackReceiver mainnet launch adaptation

This adaptation changes only the receiver's two reproduced audit findings and their tests.
The existing `launch.json` is unchanged: it deploys one `StackReceiver` on Ethereum with
IMD `0xd34a99bc0f67ae1bbd63c660e6d0b0dd03e263b7`, sIMD
`0x9efa934d9fad4ae28c998a40195646b965a97247`, and LayerZero EndpointV2
`0x1a44076050125825900e736c501f859c50fE728c` (eid 30101).

## Changes and audit reproduction

### Zero-share delivery fallback — reproduced and fixed

Finding: `b9d7d08ba346a62bbe885e9f1e855e80219d3ac42b0279b900630805bec5f117`.

The supplied proof was copied into `test/scratch/` and run against the original receiver.
Its 5 IMD delivery, split 3 IMD and 2 IMD, failed with `ZeroShares()`. The revert in the
`try` success block is not caught. Independently, the permanent regression demonstrates
that the repository's Solady-based `TestSIMD` consumes assets even when its deposit returns
zero shares. Paying plain IMD after that successful deposit would spend other delivery funds.

- `src/multichain/StackReceiver.sol`: adds a receiver-local `IStakedIMDPreview` interface
  and `_tryDeposit`. Each nonzero amount is previewed before any deposit. A zero preview,
  reverting preview, or reverting deposit pays plain IMD and emits `DeliveredPlain`.
  Successful deposits still emit `Delivered`. The supplied proof passes after this change.
- `test/StackReceiver.t.sol`: adds the real Solady zero-share reproduction, plain-IMD
  fallback with unsolicited IMD left untouched, and fuzz coverage of mixed plain/staked
  batches in both orders. Separate tests cover a reverting preview, a reverting deposit,
  and a vault whose preview contradicts its deposit. They check events, recipients,
  vault balances, receiver balances, and delivery accounting.

The fixed, trusted ERC-4626 vault must honor its positive preview. A nonconforming vault
that previews positive shares but then consumes assets and returns zero still triggers an
atomic `ZeroShares` revert. This deliberately rolls back the consumed assets instead of
paying plain IMD from another trader's balance; it is not claimed to be caught. The test
checks rollback and a subsequent retry after restoring a conforming preview. The deployed
Solady vault uses the same conversion for preview and deposit, with no intervening mutable
external call introduced by this change. Its rounding-to-zero path is handled before deposit.

### Native ETH attached to compose — reproduced and fixed

Finding: `9b5ea00143388b8ec6e56aee1cfb375bd1965123a654e866d813134f62bf3ec2`.

On the original receiver, an authenticated compose with 1 wei succeeded and left that wei
in the receiver, with no withdrawal path.

- `src/multichain/StackReceiver.sol`: adds `UnexpectedValue()` and rejects `msg.value != 0`
  after authenticating the endpoint and IMD. The LayerZero-required payable ABI is preserved.
  The RemoteStacker's existing zero-value compose options require no change.
- `test/StackReceiver.t.sol`: verifies rejection preserves the bridged IMD, approval state,
  accounting and native balances, then successfully retries without value.
- `test/StackReceiverMainnetFork.t.sol`: exercises that rejection through the live IMD and
  endpoint, checks that the compose queue entry remains available, retries without value,
  and verifies the endpoint still refuses replay. It compares native balances before and
  after, since a deployment address can already be funded on a fork.

Forced or pre-deployment ETH still cannot be recovered; this change prevents ETH being
accepted through `lzCompose`, without introducing an owner or withdrawal capability.
Both imported findings reproduced; none were dismissed.

## Launch compatibility and validation

The constructor remains nonpayable, takes the same three static address arguments, and
makes no external call. There is no owner, initializer, new deployment, linked library,
or change to the endpoint/IMD authentication, amount checks, one-time vault approval,
reentrancy guard, or SafeERC20 use. `Stacker.sol`, `ProjectRegistry.sol`, `RemoteStacker.sol`,
`LayerZero.sol`, the manifest, dependencies and build configuration are unchanged.

Checks used Foundry 1.8.3 and the repository's unchanged solc 0.8.26 / Cancun / 10,000
optimizer runs / `bytecode_hash = "none"` configuration:

- `forge build`: passes.
- `forge test`: 277 passed, 0 failed, 23 skipped (300 total). The skips are the 21 optional
  fork tests without RPC environment variables and two Stacker-specific manifest checks.
  This includes deployment rehearsals, EIP-170/opcode checks, fuzz tests and multichain
  conservation invariants. Scratch reproductions are excluded from these final counts.
- `ETH_RPC_URL=https://ethereum-rpc.publicnode.com forge test --match-contract '^StackReceiverMainnetForkTest$' -vv`:
  all five live receiver fork tests pass, including the new native-value/queue regression.
  These use the provider's latest state. An attempted historical read at block 26,181,154
  returned HTTP 403 because that provider requires a token for archive requests; that
  pinned historical state was not independently rerun. Other fork suites were not rerun.
- Receiver runtime: 4,466 bytes (previously 4,154), below EIP-170's 24,576-byte limit.
  Creation bytecode is 4,830 bytes with no library link references.

Observed gas on the live receiver fork remains within the unchanged RemoteStacker budgets:

| Traders | IMD lzReceive | Compose delivery | Compose budget |
| --- | ---: | ---: | ---: |
| 1 | 98,234 | 227,668 | 280,000 |
| 10 | 110,698 | 712,968 | 1,000,000 |
| 50 | 157,616 | 3,221,322 | 4,200,000 |

The receive budget is 250,000 gas. Slither and Mythril were not run.

## Deployment handoff

No transaction was signed or broadcast. IMD's launch process supplies the confirmed
receiver address after deploying the accepted build. At that point, add it to the README's
Ethereum mainnet Deployments table and use it for the later Robinhood RemoteStacker launch.
The README is unchanged here because the receiver is not yet deployed by this assignment;
no deployment address has been invented. No workflow files were added.
