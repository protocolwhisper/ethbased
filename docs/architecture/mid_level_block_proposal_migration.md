### Migration Guide: Mid-level Block Proposal (zk Rollup → Based Rollup)

Legend: Reuse = keep as-is; Adopt = deploy/use Taiko module; Configure = params/addresses; Modify = code changes/adapters.

```mermaid
sequenceDiagram
  autonumber

  %% Participants with migration guidance and repo mapping
  participant U as User<br/>Reuse<br/>packages: bridge-ui, ui-lib
  participant P as Proposer Client<br/>Modify<br/>package: taiko-client/proposer
  participant L1R as L1 Resolver<br/>Reuse<br/>contracts: protocol/shared/common/Resolvers
  participant AM as AddressManager<br/>Reuse<br/>contracts: protocol/shared/common/DefaultResolver.sol
  participant L1I as Taiko Inbox / IPropose<br/>Adopt<br/>contracts: protocol/layer1/based2/Inbox.sol and TaikoInbox.sol
  participant BM as BondManager<br/>Adopt/Configure<br/>contracts: protocol/layer1/based and based2
  participant L1S as SignalService<br/>Reuse<br/>contracts: protocol/shared/signal
  participant L2C as L2 Client / Builder<br/>Modify<br/>package: taiko-client/driver
  participant L2T as TaikoL2 Anchor<br/>Adopt/Implement<br/>contracts: protocol/layer2/based/anchor
  participant TR as Treasury<br/>Configure<br/>L2 system account

  Note over P,L1I: Propose L2 block metadata + txList commitment on L1

  U->>P: Submit L2 txs to mempool
  P->>L1R: Resolve protocol contract addresses
  L1R->>AM: lookup(name, chainId)
  AM-->>L1R: resolved addresses
  L1R-->>P: Inbox, BondManager, SignalService addresses

  Note over P,L1S: Gather inputs for metadata (l1Height, l1Hash, l1SignalRoot, basefee params)
  P->>L1S: fetch l1SignalRoot (via RPC proof)
  P->>P: Build txList
  P->>P: Compute blobHash and txList byte range
  P->>P: Determine gasLimit and mixHash

  alt proposer bond required
    P->>BM: deposit proposer bond
    BM-->>P: bond receipt
  end

  P->>L1I: propose metadata and commitments
  L1I->>L1I: validate metadata, bounds, proposer auth
  L1I-->>P: Proposed(event: id, parentHash, proposer)

  Note over L2C,L2T: Build L2 block with Anchor as 1st tx (golden-touch)
  L2C->>L2T: anchor(l1Hash, l1SignalRoot, l1Height, parentGasUsed)
  L2T-->>L2C: Anchored(event)

  Note over L1I,L2C: EIP-1559 basefee params included in metadata
  Note over L2C,TR: Basefee is sent to treasury on L2
  L2C->>TR: transfer basefee
```

#### What to Modify vs Reuse
- Modify
  - Proposer client: `packages/taiko-client/proposer` (adapter to your L2 execution client, txList building, config for L1 addresses and chains).
  - L2 client/builder: `packages/taiko-client/driver` (ensure anchor as first tx; integrate basefee path).
  - L2 chain integration: implement or adapt `protocol/contracts/layer2/based/anchor/*` to your chain specifics.

- Adopt/Configure
  - L1 Inbox/IPropose: `protocol/contracts/layer1/based2/{Inbox.sol, TaikoInbox.sol}` (deploy; wire into resolver; set limits).
  - Bonding/policy: `protocol/contracts/layer1/based{,2}` (BondManager, LibBonds) according to your economics.
  - Treasury and EIP-1559 params: configure destination address and curve parameters.

- Reuse (minimal/no changes)
  - SignalService (L1/L2): `protocol/contracts/shared/signal`.
  - Address resolution: `protocol/contracts/shared/common/*Resolver*`, `DefaultResolver.sol`.
  - UI packages (where applicable): `packages/bridge-ui`, `packages/ui-lib`.

Notes:
- If you replace proof systems later, do so in verifier contracts and prover client; it does not affect the proposal flow shown here.
- Keep interfaces stable to maximize reuse; add adapters at client boundaries rather than changing shared contracts.


