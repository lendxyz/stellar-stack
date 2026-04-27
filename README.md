# Technical Architecture — Lend Protocol on Stellar

Lend is a compliant real estate tokenization protocol deployed on Stellar using Soroban smart contracts. The protocol enables investors to deploy stablecoins into professionally structured real estate operations, with programmable yield distribution and secondary market liquidity.

This document describes the full technical architecture of the Stellar integration.

---

## System Overview

```mermaid
graph TB
    subgraph Frontend["Frontend"]
        SWK["Stellar Wallet Kit"]
        ABS["Allbridge SDK"]
        KYC["KYC Flow"]
    end

    subgraph Worker["Backend Worker"]
        IDX["Event Indexer"]
        API["REST API"]
        SIG["KYC Signature Generator"]
    end

    subgraph Stellar["Stellar Network"]
        FC["Factory Contract<br/>(Soroban)"]
        OPL["OpLend Tokens<br/>(per operation)"]
        REF["Reflector Oracle<br/>(EUR/USDC)"]
        FC --> OPL
        FC --> REF
    end

    subgraph Wallets["Wallet Connection Layer"]
        SW["Stellar Wallet<br/>Lobster / Freighter / xBull"]
        EW["EVM Wallet<br/>Rabby / MetaMask"]
    end

    subgraph Bridge["Cross-Chain Bridge (Allbridge)"]
        EVM["EVM USDC<br/>(Ethereum, Polygon, BSC, Arbitrum)"]
        ALB["Allbridge Core"]
        SUSDC["Stellar USDC"]
        EVM --> ALB --> SUSDC
    end

    Frontend <--> Worker
    Worker <--> Stellar
    Frontend --> Wallets
    Wallets --> Bridge
    SUSDC --> FC

    classDef stellar fill:#d6eaf8,stroke:#1a3a5c,color:#1a3a5c
    classDef frontend fill:#d5f5e3,stroke:#1e8449,color:#1e8449
    classDef worker fill:#fef9e7,stroke:#f39c12,color:#b9770e
    classDef bridge fill:#f5eef8,stroke:#6c3483,color:#6c3483

    class FC,OPL,REF stellar
    class SWK,ABS,KYC frontend
    class IDX,API,SIG worker
    class EVM,ALB,SUSDC bridge
```

---

## 1. Smart Contracts (Soroban)

### 1.1 Factory Contract

The Factory is the primary entry point of the protocol. It orchestrates the full lifecycle of tokenized investment operations.

**Responsibilities:**

- Create tokenized operations (deploy a new OpLend token contract per operation)
- Manage funding state (open, pause, cancel, complete)
- Process investor subscriptions with backend signature verification
- Handle pre-deposits and token claims
- Execute cancellation with automated investor refunds
- Enable fund withdrawal by the operation issuer
- Emit protocol events for off-chain indexing

**Key functions:**

| Function | Description | Access |
|----------|-------------|--------|
| `create_operation` | Deploy new OpLend token, register operation | Admin |
| `start_operation` | Open funding to investors | Admin |
| `invest` | Subscribe to operation (requires backend signature) | Investor (whitelisted) |
| `predeposit` | Reserve shares before operation starts | Investor (whitelisted) |
| `claim_tokens` | Claim OpLend tokens from predeposit | Investor |
| `cancel_operation` | Cancel operation, enable refunds | Admin |
| `refund` | Refund investor after cancellation | Investor |
| `withdraw_funds` | Withdraw raised capital to destination | Admin |

**Investment flow — signature verification:**

```mermaid
sequenceDiagram
    participant I as Investor
    participant B as Backend
    participant F as Factory Contract

    I->>B: KYC submission
    B->>B: KYC/AML check
    B->>I: Signature (authorization)
    I->>F: invest(shares, signature)
    F->>F: Verify signature
    F->>F: Check whitelist
    F->>F: Query oracle (EUR/USDC)
    F->>F: Transfer USDC
    F->>F: Mint OpLend tokens
    F->>F: Emit Invested event
    F->>I: OpLend tokens
```

### 1.2 OpLend Token Contract (SEP-41)

Each real estate operation deploys a dedicated OpLend token representing investor shares in the financing structure.

**Interface:** Implements the Soroban Token Interface ([SEP-41](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md)) with 9 standard functions + 6 admin functions.

**Compliance features:**

- **Whitelist-based transfers:** Only whitelisted addresses can send and receive tokens
- **Blacklist enforcement:** Blocked addresses cannot interact with the token
- **Capped supply:** Total supply is hard-capped to the operation's funding target
- **Transfer restrictions:** All transfers validated against compliance rules before execution

**Storage model:**

| Data | Storage Type | Rationale |
|------|-------------|-----------|
| Admin, config | Instance | Shared across all calls, high access frequency |
| Investor balances | Persistent | Must survive archival, per-user data |
| Allowances | Temporary | Short-lived approvals, auto-cleanup acceptable |

### 1.3 Oracle Integration (Reflector)

Operations are priced in EUR while investors settle in USDC. The Factory integrates [Reflector](https://reflector.network/), Stellar's native oracle network, to dynamically convert share prices.

**Flow:**

1. Operation share price is defined in EUR at creation
2. When an investor subscribes, the Factory queries the Reflector oracle adapter for the current EUR/USDC rate
3. The required USDC amount is calculated: `shares × price_per_share_eur × eur_usdc_rate`
4. The investor transfers the calculated USDC amount

**Safety mechanisms:**

- Price staleness check: reject if oracle data is older than a configurable threshold
- Price deviation bounds: reject if price moves beyond acceptable range between simulation and execution
- Fallback: operation can be paused if oracle is unavailable

**Implementation:**

```mermaid
graph LR
    FC["Factory Contract<br/>EUR amount requested"] -->|query price| OA["Oracle Adapter<br/>(Soroban)"]
    OA -->|fetch rate| RF["Reflector Network<br/>EUR/USDC live feed"]
    RF -->|EUR/USDC rate| OA
    OA -->|USDC amount| FC

    classDef stellar fill:#d6eaf8,stroke:#1a3a5c,color:#1a3a5c
    class FC,OA,RF stellar
```

---

## 2. Dual-Wallet Architecture

A key differentiator of the Lend protocol is the ability for investors to connect both a Stellar wallet and an EVM wallet under a single platform account. This enables cross-chain capital onboarding while keeping investment settlement on Stellar.

### 2.1 Wallet Connection

**Stellar wallet:**
- Connected via [Stellar Wallet Kit](https://stellarwalletskit.dev/)
- Supported wallets: Lobster, Freighter, xBull
- Used for: investing in operations, receiving OpLend tokens, claiming yields

**EVM wallet:**
- Connected via standard Web3 provider (Rabby, MetaMask)
- Used for: bridging USDC from EVM chains to Stellar via Allbridge

**Account linking:**

```mermaid
graph LR
    subgraph Account["Lend Platform Account — KYC verified"]
        direction LR
        SW["🔵 Stellar Wallet<br/>Lobster / Freighter / xBull<br/><br/>• Invest<br/>• Receive OpLend tokens<br/>• Claim yields<br/>• Sign transactions"]
        EW["🟣 EVM Wallet (optional)<br/>Rabby / MetaMask<br/><br/>• Bridge USDC to Stellar<br/>• Approve bridge transactions"]
    end

    SW -.-|"linked under<br/>same account"| EW

    classDef stellar fill:#d6eaf8,stroke:#1a3a5c,color:#1a3a5c
    classDef evm fill:#f5eef8,stroke:#6c3483,color:#6c3483
    classDef account fill:#f8f9fa,stroke:#2c3e50,color:#2c3e50

    class SW stellar
    class EW evm
```

Both wallets are linked to the same KYC-verified identity. The Stellar wallet is the primary wallet that receives OpLend tokens and weekly yield distributions. The EVM wallet is optional and used exclusively for cross-chain capital bridging.

### 2.2 User Flows

**Flow A — Stellar-native investor:**

1. Connect Stellar wallet via Stellar Wallet Kit
2. Complete KYC verification on platform
3. Browse available real estate operations
4. Invest USDC directly from Stellar wallet
5. Receive OpLend tokens representing investment position
6. Receive weekly yield distributions to Stellar wallet

**Flow B — Cross-chain investor (EVM → Stellar):**

1. Connect Stellar wallet via Stellar Wallet Kit
2. Connect EVM wallet via Web3 provider (Rabby, MetaMask)
3. Complete KYC verification on platform
4. Browse available real estate operations
5. Select investment amount → platform calculates USDC needed
6. Approve USDC on EVM chain
7. Allbridge bridges USDC from EVM to investor's Stellar wallet
8. Factory contract processes investment from Stellar wallet
9. Receive OpLend tokens on Stellar wallet
10. Receive weekly yield distributions to Stellar wallet

**Key principle:** The bridge is a one-way capital onboarding mechanism. Once USDC arrives on Stellar, all subsequent operations (investment, yield distribution, token transfers) happen natively on Stellar.

---

## 3. Cross-Chain Bridge (Allbridge Core)

### 3.1 Architecture

Allbridge Core is integrated to enable USDC transfers from EVM chains into Stellar. The bridge flow is embedded directly in the Lend frontend, presenting a seamless single-session experience.

**Supported source chains:** Ethereum, Polygon, BSC, Arbitrum

**Bridge flow:**

```mermaid
graph LR
    EVM["Investor EVM Wallet<br/>(USDC)"] -->|"1. approve"| ALB["Allbridge<br/>Core Protocol"]
    EVM -->|"2. send USDC"| ALB
    ALB -->|"3. mint USDC"| SW["Investor<br/>Stellar Wallet"]
    SW -->|"4. invest"| FC["Factory<br/>Contract"]

    classDef evm fill:#f5eef8,stroke:#6c3483,color:#6c3483
    classDef bridge fill:#fef9e7,stroke:#f39c12,color:#b9770e
    classDef stellar fill:#d6eaf8,stroke:#1a3a5c,color:#1a3a5c

    class EVM evm
    class ALB bridge
    class SW,FC stellar
```

### 3.2 Frontend Integration

The Allbridge SDK is integrated into the Lend frontend. The user experience is:

1. User selects an operation and investment amount
2. If paying from EVM: connect EVM wallet, approve USDC, initiate bridge
3. Frontend displays bridge progress with status updates
4. Once USDC arrives on Stellar, the investment transaction is prepared
5. User signs the Stellar transaction via Stellar Wallet Kit
6. Investment is processed by the Factory contract

The bridge and investment are presented as a single guided process, but they are technically two separate transactions (bridge + invest) to maintain clean separation of concerns.

---

## 4. Compliance Layer

### 4.1 Regulatory Framework

Lend operates under French financial regulation. Each tokenized operation requires a formal investment document (DIS — Document d'Information Synthétique) submitted to the Autorité des Marchés Financiers (AMF).

### 4.2 On-Chain Compliance

Compliance is **structural**, not declarative. It is enforced at the smart contract level:

**Whitelist/Blacklist (Factory + OpLend):**
- Only whitelisted addresses can invest through the Factory
- Only whitelisted addresses can receive OpLend token transfers
- Blacklisted addresses are blocked from all protocol interactions
- Whitelist/blacklist managed by protocol admin

**Backend Signature Authorization:**
- Every investment requires a cryptographic signature generated by the backend
- The backend only generates signatures after successful KYC/AML verification
- The Factory contract verifies the signature on-chain before processing the investment
- This creates a two-layer compliance gate: off-chain verification + on-chain enforcement

**Identity linking:**
- Each wallet address is associated with a declared identity (legal name, residential address)
- Required for regulatory compliance: each tokenized bond position must be linked to a real-world investor identity
- Sanctions and AML screening performed against relevant databases before signature generation

### 4.3 Compliance Flow

```mermaid
sequenceDiagram
    participant I as Investor
    participant B as Lend Backend
    participant C as Compliance Provider
    participant F as Factory Contract

    I->>B: Submit identity
    B->>C: KYC/AML screening
    C->>B: Result (pass/fail)
    B->>I: Signature (if passed)
    I->>F: invest(shares, signature)
    F->>F: Verify signature
    F->>F: Check whitelist
    F->>F: Process investment
    F->>I: Mint OpLend tokens
```

---

## 5. Backend Worker & Event Indexing

### 5.1 Role

The backend worker maintains an accurate off-chain representation of the protocol state by continuously indexing events emitted by the Factory and OpLend contracts.

### 5.2 Data Sources

| Source | Used For |
|--------|----------|
| **Soroban RPC** (`getEvents`) | Contract events, transaction simulation, contract invocation |
| **Horizon** | Account balances, transaction history, asset metadata, network data |

### 5.3 Indexed Events

| Event | Source | Data |
|-------|--------|------|
| `OperationCreated` | Factory | Operation ID, name, total shares, price, OpLend token address |
| `OperationStarted` | Factory | Operation ID, timestamp |
| `OperationPaused` | Factory | Operation ID |
| `OperationCanceled` | Factory | Operation ID |
| `OperationFinished` | Factory | Operation ID, total funded |
| `Invested` | Factory | Operation ID, investor address, shares, USDC amount |
| `Predeposit` | Factory | Operation ID, investor address, shares |
| `ClaimedOpToken` | Factory | Operation ID, investor address, token amount |
| `Refunded` | Factory | Operation ID, investor address, USDC amount |

### 5.4 Reconstructed State

The worker reconstructs and maintains:

- **Operations:** list, status, funding progress, metadata
- **Investor positions:** allocations per operation, token balances, claimable amounts
- **Protocol metrics:** TVL, total capital deployed, unique investors, operation count


### 5.5 Architecture

```mermaid
graph LR
    RPC["Soroban RPC<br/>(events)"] --> W["Backend Worker<br/>Indexer + Database<br/>API + KYC Signatures"]
    HOR["Horizon<br/>(accounts)"] --> W
    W --> FE["Frontend<br/>(React)<br/>REST API consumer"]

    classDef stellar fill:#d6eaf8,stroke:#1a3a5c,color:#1a3a5c
    classDef worker fill:#fef9e7,stroke:#f39c12,color:#b9770e
    classDef frontend fill:#d5f5e3,stroke:#1e8449,color:#1e8449

    class RPC,HOR stellar
    class W worker
    class FE frontend
```

---

## 6. Yield Distribution

Lend distributes yields to investors on a weekly basis, entirely on-chain. This is a key reason for choosing Stellar: the low transaction costs (~0.00001 XLM per operation) make weekly distribution to thousands of investors economically viable.

### 6.1 Distribution Mechanism

1. Real estate operation generates revenue (rent, interest)
2. Lend's asset management team processes the revenue off-chain
3. Revenue is converted to USDC and deposited to the distribution wallet on Stellar
4. Distribution is executed proportionally to each investor's OpLend token holdings
5. USDC is sent directly to each investor's Stellar wallet

### 6.2 Transparency

All yield distributions are visible on-chain through Stellar Explorer. Investors can verify:
- Distribution frequency and amounts
- Proportional allocation relative to their holdings
- Historical distribution record

---

## 7. Incentive Mechanism

To encourage adoption and anchor capital on Stellar, Lend introduces a **1.25x multiplier on Lend Points** for investments executed on the Stellar instance during the first year.

This incentive is designed to:
- Position Stellar as the preferred chain for Lend investors
- Drive early adoption and long-term user anchoring
- Create a concrete mechanism for TVL growth on Stellar

At this stage, incentives are limited to points-based rewards. This grant is strictly scoped to development and does not include capital allocation for yield subsidies.

---

## 8. Design Decisions

### Why Soroban smart contracts over Stellar Classic Assets?

Stellar supports native asset issuance through Classic Assets, but Lend chose Soroban for the following reasons:

| Requirement | Classic Assets | Soroban | Choice |
|------------|---------------|---------|--------|
| Transfer restrictions (whitelist) | Limited (authorization flags) | Full programmability | Soroban |
| Compliance hooks per transaction | Not possible | Custom logic in `invest()` | Soroban |
| Supply cap enforcement | Manual | Built into contract | Soroban |
| Operation-specific token logic | Not possible | Per-operation contract | Soroban |
| Protocol event emission | Not available | Full event system | Soroban |
| Backend signature verification | Not possible | Custom verification | Soroban |

### Why Stellar over other networks?

The primary strategic reason for choosing Stellar is its **anchor network (SEP-6/24)**.

Lend targets European and international expansion with the ambition of onboarding investors who are **not crypto-native** — institutional LPs, family offices, traditional real estate investors. These investors need regulated fiat on/off-ramps: deposit euros, invest in tokenized real estate, withdraw yields in fiat.

Stellar's anchor network provides exactly this. Multiple regulated anchors are already active across Europe and internationally, offering compliant fiat ramps integrated at the protocol level. This is a structural advantage that aligns naturally with Lend's positioning and target market.

| Factor | Relevance for Lend |
|--------|-------------------|
| **Anchor network (SEP-6/24)** | **Primary reason.** Regulated fiat on/off-ramps for non-crypto-native investors. Enables European and international expansion. No equivalent on EVM. |
| **Reflector oracle** | Native EUR/USDC oracle for EUR-denominated real estate pricing. |
| **Soroban** | Programmable compliance for regulated securities (whitelist, KYC signatures, capped supply). |
| **RWA ecosystem alignment** | Stellar's strategic focus on real-world assets (SDF 2026 roadmap: $1B in tokenized assets). |
| **Transaction costs** | Economical weekly yield distribution to large investor bases. |
| **Settlement speed** | 5-second finality for investment confirmation. |

---

## 9. Operation Lifecycle

```mermaid
graph TD
    A["Operation Created<br/>Factory deploys OpLend token"] --> B["Funding Started<br/>Open for subscriptions"]
    B --> C["Investor Action<br/>Invest / Predeposit"]
    C --> D["Immediate mint<br/>OpLend tokens"]
    C --> E["Deferred claim path<br/>(predeposits)"]
    D --> F{"Funding Complete?"}
    E --> F
    F -->|No| G["Continue / Pause / Cancel"]
    F -->|Yes| H["Operation Finished"]
    H --> I["Funds Withdrawn"]
    I --> J["Weekly Yield Distribution"]

    classDef active fill:#d6eaf8,stroke:#1a3a5c,color:#1a3a5c
    classDef decision fill:#fef9e7,stroke:#f39c12,color:#b9770e
    classDef final fill:#d5f5e3,stroke:#1e8449,color:#1e8449

    class A,B,C,D,E active
    class F decision
    class H,I,J final
    class G active
```

---

## 10. Deployment Architecture

### Testnet (current)

- Factory contract deployed on Soroban testnet
- OpLend token contract deployed on Soroban testnet
- Testnet address: [GAIOQM6QINN427MWFQUHJZGG6T6KOE2ZGLRS2DVYIUGUOBSREDHJNTQM](https://testnet.stellarchain.io/address/GAIOQM6QINN427MWFQUHJZGG6T6KOE2ZGLRS2DVYIUGUOBSREDHJNTQM)
- Source code: [github.com/lendxyz/lend-contracts-soroban](https://github.com/lendxyz/lend-contracts-soroban)

### Mainnet (planned)

- Production deployment with hardened configuration
- Backend worker indexing mainnet events
- Reflector oracle on mainnet feeds
- Allbridge configured for mainnet USDC
- Security review of all contract parameters completed before deployment
