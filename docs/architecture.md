### Taiko Protocol Architecture (High-level)

```mermaid
graph TB
  %% High-level Taiko Protocol Architecture
  subgraph "Actors"
    U["User"]
    P["Proposer"]
    V["Prover(s)"]
    R["Relayer"]
    G["DAO / Timelock"]
  end

  subgraph "L1 (Ethereum)"
    L1T["TaikoL1 (Protocol Core)"]
    L1S["SignalService (L1)"]
    L1B["Bridge (L1)"]
    L1V["Token Vaults (L1): ERC20/721/1155"]
    AM["AddressManager"]
  end

  subgraph "L2 (Taiko)"
    L2T["TaikoL2 (Execution / Anchor Tx)"]
    L2S["SignalService (L2)"]
    L2B["Bridge (L2)"]
    L2V["Token Vaults (L2): ERC20/721/1155"]
    TR["Treasury (receives basefee)"]
  end

  %% L2 usage
  U -->|"send L2 txs"| L2T

  %% Proposing / Anchoring
  P -->|"propose block: metadata + txList commitment"| L1T
  L1T -->|"anchor params for first L2 tx"| L2T

  %% Proving / Contesting (contestable validity rollup)
  V -->|"submit proof / contest (tiers)"| L1T
  L1T -->|"on verification: finalize transition"| L1S

  %% State root sync (auto by protocol)
  L1S <--> |"auto-sync state roots"| L2S

  %% EIP-1559 on L2
  L1T -->|"basefee parameters (EIP-1559) in metadata"| L2T
  L2T -->|"basefee sent"| TR

  %% Bridging via SignalService proofs
  R -->|"relay messages + proofs"| L2B
  R -->|"relay messages + proofs"| L1B
  L2B <--> |"verify via SignalService proofs"| L1B
  L1B -->|"lock / release"| L1V
  L2B -->|"mint / burn / unlock"| L2V

  %% Access control / governance
  G -->|"owns / controls"| AM
  AM -.->|"grants roles"| L1B
  AM -.->|"grants roles"| L2B
  AM -.->|"grants roles"| L1V
  AM -.->|"grants roles"| L2V

  %% Optional note
  note1(["Contestable Validity Rollup: transitions can be re-proven/contested until verified on L1"]) 
  note1 --- L1T
```