### Taiko Protocol Architecture (Mid-level: Block Proposal)

```mermaid
sequenceDiagram
  autonumber

  %% Participants mapped to repo packages/contracts
  participant U as User (packages: bridge-ui, ui-lib)
  participant P as Proposer Client (package: taiko-client/proposer)
  participant L1R as L1 Resolver (contracts: protocol/contracts/shared/common/Resolvers)
  participant AM as AddressManager (contracts: protocol/contracts/shared/common/DefaultResolver.sol)
  participant L1I as Taiko Inbox / IPropose (contracts: protocol/contracts/layer1/based2/Inbox.sol, TaikoInbox.sol)
  participant BM as BondManager (contracts: protocol/contracts/layer1/based & based2 / BondManager)
  participant L1S as SignalService (L1) (contracts: protocol/contracts/shared/signal)
  participant L2C as L2 Client / Builder (package: taiko-client/driver)
  participant L2T as TaikoL2 Anchor (contracts: protocol/contracts/layer2/based/anchor)
  participant TR as Treasury (L2 system account)

  Note over P,L1I: Goal: Propose L2 block metadata + txList commitment on L1

  U->>P: Submit L2 txs to mempool
  P->>L1R: Resolve protocol contract addresses
  L1R->>AM: lookup(name, chainId)
  AM-->>L1R: resolved addresses
  L1R-->>P: TaikoInbox, BondManager, SignalService addresses

  Note over P,L1S: Gather inputs for metadata (l1Height, l1Hash, l1SignalRoot, basefee params)
  P->>L1S: fetch l1SignalRoot (via RPC proof)
  P->>P: Build txList, compute blobHash/byte range, gasLimit, mixHash

  alt proposer bond required
    P->>BM: deposit proposer bond
    BM-->>P: bond receipt
  end

  P->>L1I: propose(meta, blobHash, txListByteStart/End, gasLimit, ...)
  L1I->>L1I: validate txList bounds, metadata, proposer auth
  L1I-->>P: Proposed(event: id, parentHash, proposer)

  Note over L2C,L2T: Build L2 block with Anchor as first tx (golden-touch)
  L2C->>L2T: anchor(l1Hash, l1SignalRoot, l1Height, parentGasUsed)
  L2T-->>L2C: Anchored(event)

  Note over L1I,L2C: EIP-1559 basefee params included in metadata
  Note over L2C,TR: Basefee is sent to treasury on L2
  L2C->>TR: transfer basefee
```

#### Notes
- Proposer client modules: `packages/taiko-client/proposer`, uses `packages/taiko-client/driver` to build L2 blocks and include the anchor tx.
- L1 proposal entrypoints: `protocol/contracts/layer1/based2/IPropose.sol`, `Inbox.sol`, `TaikoInbox.sol` (network-specific variants under `layer1/mainnet`, `layer1/hekla`, etc.).
- Bonds: `protocol/contracts/layer1/based/IBondManager.sol`, `based2/IBondManager2.sol`, with state helpers in `based2/state/LibBonds.sol`.
- Resolver and address management: `protocol/contracts/shared/common/*Resolver*`, `DefaultResolver.sol`.
- Signal Service (L1/L2): `protocol/contracts/shared/signal`, provides `signalRoot` used by the anchor.
- TaikoL2 anchor and EIP-1559: `protocol/contracts/layer2/based/anchor/*`, `layer2/based/eip1559/*`.


