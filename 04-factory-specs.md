# Factory Specs

## Overview

The Lend Factory contract is the core orchestration layer of the Lend protocol on Soroban.

Its responsibility is to create and manage tokenized investment operations representing real-world asset financing structures. Each operation corresponds to a dedicated OpLend token contract representing investor positions in the financing.

The Factory manages the lifecycle of each operation including creation, funding state, investor participation and completion. It also emits protocol events that are indexed by the backend worker in order to reconstruct the platform state off-chain.

---

## Architecture

The protocol architecture is composed of three main components.

### Factory Contract

The Factory contract acts as the primary entry point of the protocol. It is responsible for:

- creating tokenized operations
- validating investments
- managing funding state
- emitting protocol events
- minting OpLend tokens

### OpLend Token

Each operation deploys a dedicated OpLend token contract implementing the Soroban token interface.

These tokens represent investor shares in a specific financing structure and enforce compliance rules such as capped supply and transfer restrictions.

### Backend Worker

A backend worker continuously reads events emitted by the Factory and OpLend contracts through Soroban RPC.

These events are used to reconstruct the protocol state including:

- funding progress
- investor allocations
- operation lifecycle

---

## Operation Lifecycle

Each investment opportunity follows the lifecycle below.

### 1. Operation Creation

The protocol administrator creates a new tokenized operation through the Factory.

A new OpLend token contract is deployed and linked to the operation.

### 2. Funding Start

Once the operation is ready, funding is opened.

Investors can begin participating in the operation.

### 3. Investment Phase

Users can subscribe to the operation by purchasing shares.

Funds are transferred to the protocol and OpLend tokens are minted to represent the investor position.

### 4. Funding Completion

Once the total number of shares is reached, the operation is marked as completed.

### 5. Token Claim / Distribution

In certain cases such as predeposits, users may claim their OpLend tokens once the operation has started.

---

## Core Features

### Create Operation

Creates a new tokenized financing operation.

Parameters:

- operation name
- total shares
- price per share

Behavior:

- deploy a new OpLend token contract
- register the operation in Factory storage
- emit an `OperationCreated` event

---

### Start Operation

Marks an operation as open for funding.

Once started, investors can participate in the funding round.

---

### Invest

Allows a user to participate in an operation by purchasing shares.

The Factory performs several checks before accepting the investment:

- operation must exist
- funding must be active
- operation must not be canceled
- share limit must not be exceeded
- user must not be blacklisted

The contract then:

- verifies the backend signature authorization
- calculates the stablecoin amount required
- transfers funds to the protocol
- updates funding progress
- mints OpLend tokens

---

### Predeposit

Allows investors to reserve shares before an operation officially starts.

Funds are transferred to the protocol but tokens are only minted once the operation is started.

---

### Claim Tokens

Users who participated through predeposit or gifted allocations can claim their OpLend tokens once the operation begins.

---

### Cancel Operation

The protocol administrator may cancel an operation before completion.

In such cases, users may later be refunded.

---

### Withdraw Funds

Once an operation is successfully funded, the protocol administrator can withdraw the raised funds to the designated destination.

---

## Compliance and Security

The protocol integrates several compliance mechanisms.

### Backend Authorization

Investments require a backend-generated signature authorizing the transaction.

This mechanism allows the protocol to enforce off-chain compliance rules such as KYC validation.

### Transfer Restrictions

OpLend tokens enforce restricted transfers through whitelisting logic.

### Blacklist

The protocol can prevent specific addresses from interacting with the system.

---

## Pricing Mechanism

Investment pricing is based on:

- the share price defined in EUR
- the EUR/USD oracle price

The contract calculates the required stablecoin amount dynamically using the oracle.

This mechanism ensures that the share price remains stable in EUR terms while accepting stablecoin payments.

---

## Oracle Integration

Note: Work in progress as the design will vary from our original codebase

The Factory relies on an external price oracle adapter to convert the operation share price denominated in EUR into the subscription asset amount required on Stellar.

We plan to leverage Reflector as our primary price oracle, combined with additional safety mechanisms to ensure that pricing and sales cannot be manipulated.

---

## Operation lifecycle

```
+----------------------+
|   Operation Created  |
| Factory deploys      |
| OpLend token         |
+----------+-----------+
           |
           v
+----------------------+
|   Operation Ready    |
| Metadata registered  |
| Waiting for funding  |
+----------+-----------+
           |
           v
+----------------------+
|   Funding Started    |
| Operation opened     |
| for subscriptions    |
+----------+-----------+
           |
           v
+----------------------+
|   Investor Action    |
| Invest / Predeposit  |
| / Gift allocation    |
+----------+-----------+
           |
           v
+-------------------------------+
| Funding state updated         |
| - funding progress increases  |
| - investor allocation stored  |
| - funds transferred           |
+----------+--------------------+
           |
           +-------------------------------+
           |                               |
           v                               v
+----------------------+        +----------------------+
| Immediate mint       |        | Deferred claim path  |
| OpLend minted        |        | For predeposits /    |
| to investor          |        | gifted allocations   |
+----------+-----------+        +----------+-----------+
           |                               |
           |                               v
           |                    +----------------------+
           |                    | Claim OpLend tokens  |
           |                    | once operation is    |
           |                    | started              |
           |                    +----------+-----------+
           |                               |
           +---------------+---------------+
                           |
                           v
+----------------------+
| Funding Completed ?  |
| totalShares reached  |
+----------+-----------+
           |
      +----+----+
      |         |
     No        Yes
      |         |
      |         v
      |   +----------------------+
      |   | Operation Finished   |
      |   | Funding closed       |
      |   +----------+-----------+
      |              |
      |              v
      |   +----------------------+
      |   | Funds Withdrawn      |
      |   | Capital sent to      |
      |   | destination wallet   |
      |   +----------------------+
      |
      v
+----------------------+
| Funding continues    |
| until full / paused  |
| / canceled           |
+----------------------+
```

---

## Events

The Factory emits events that allow the backend infrastructure to reconstruct protocol state.

Key events include:

- `OperationCreated`
- `OperationStarted`
- `OperationPaused`
- `OperationCanceled`
- `Invested`
- `Predeposit`
- `Gifted`
- `ClaimedOpToken`
- `Refunded`
- `OperationFinished`

These events are consumed by the backend worker through Soroban RPC.

---

## Backend Worker

The backend worker is responsible for indexing protocol activity.

It continuously reads events emitted by the Factory and OpLend contracts and reconstructs the platform state.

Indexed data includes:

- operations
- funding progress
- investor allocations
- claimable balances
- operation status

This indexed state is stored in a database used by the frontend and analytics systems.

The worker interacts with the Stellar network using:

- Soroban RPC for contract events
- Horizon for network and account data
