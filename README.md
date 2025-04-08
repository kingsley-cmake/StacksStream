# StacksStream Payment Channels Contract

![Stacks Bitcoin Integration](https://img.shields.io/badge/Blockchain-Stacks-%235768FF?logo=bitcoin&logoColor=white)  
**A Bitcoin-secured payment channel system for instant off-chain transactions**

## Overview

StacksStream implements a robust payment channel network enabling high-speed value transfers on Layer 2 while inheriting Bitcoin's unparalleled security through the Stacks blockchain. This contract facilitates trustless peer-to-peer transactions with instant finality, designed for micropayments and high-frequency transfers.

## Key Features

- **Bitcoin-Anchored Security**: Inherits Bitcoin's proof-of-work security through Stacks' block anchoring
- **Dual Closure Mechanisms**:
  - Cooperative closing with mutual signatures
  - Unilateral closure with 1008-block dispute period (~1 week Bitcoin time)
- **Multi-Fund Channels**: Add funds to existing channels dynamically
- **Dispute Resolution**: On-chain enforcement of last valid state
- **Emergency Withdraw**: Contract owner failsafe for protocol recovery
- **Non-Custodial Design**: Users maintain full control of funds

## Technical Architecture

### Core Components

- **Channel Identification**:
  - Unique 32-byte channel ID
  - Participant principal addresses (A & B)
- **Channel State**:
  ```clarity
  {
    total-deposited: uint,    // Total STX in channel
    balance-a: uint,          // Participant A's balance
    balance-b: uint,          // Participant B's balance
    is-open: bool,            // Channel status
    dispute-deadline: uint,   // Block height for dispute resolution
    nonce: uint               // State version counter
  }
  ```

## Core Functions

### 1. Channel Creation (`create-channel`)

Establishes new payment channel with initial deposit

**Parameters:**

- `channel-id`: (buff 32) Unique channel identifier
- `participant-b`: (principal) Counterparty address
- `initial-deposit`: (uint) STX amount to lock

**Requirements:**

- Minimum deposit > 0 STX
- Unique channel ID for participant pair

```clarity
(create-channel 0x1234abcd 'SP3ABC 5000)
```

### 2. Channel Funding (`fund-channel`)

Adds additional liquidity to existing channel

**Parameters:**

- `channel-id`: Existing channel identifier
- `participant-b`: Counterparty principal
- `additional-funds`: STX amount to add

**State Changes:**

- Increases `total-deposited` and caller's balance
- Maintains channel openness

```clarity
(fund-channel 0x1234abcd 'SP3ABC 2500)
```

### 3. Cooperative Closure (`close-channel-cooperative`)

Mutually-signed instant settlement

**Parameters:**

- Final balance allocations
- Dual ECDSA signatures (secp256k1)

**Execution Flow:**

1. Verify signature authenticity
2. Validate balance sum equals total
3. Distribute funds
4. Mark channel closed

```clarity
(close-channel-cooperative
  0x1234abcd
  'SP3ABC
  6000
  1500
  0xsignatureA
  0xsignatureB
)
```

### 4. Unilateral Closure (`initiate-unilateral-close`)

Force channel closure with dispute period

**Mechanics:**

- Submitter provides latest signed state
- 1008-block dispute window starts
- Counterparty can challenge with newer state
- After window: funds distributed as proposed

```clarity
(initiate-unilateral-close
  0x1234abcd
  'SP3ABC
  5500
  2000
  0xsignatureA
)
```

### 5. Dispute Resolution (`resolve-unilateral-close`)

Finalizes closure after dispute period

**Requirements:**

- Current block > dispute deadline
- No pending challenges

```clarity
(resolve-unilateral-close 0x1234abcd 'SP3ABC)
```

## Security Model

### Signature Verification

- ECDSA secp256k1 signatures required for state updates
- Message format: `channel-id || balanceA || balanceB`
- Dual verification for cooperative closure

### Dispute Safeguards

- 1008-block delay (~1 week Bitcoin time)
- Only newer state nonces accepted during disputes
- Balance consistency checks:
  ```clarity
  (asserts! (is-eq total-channel-funds (+ balance-a balance-b)))
  ```

### Emergency Protocol

**Emergency Withdraw Function:**

- Contract owner can retrieve all locked funds
- Requires principal match `CONTRACT-OWNER`
- Single-step withdrawal process

```clarity
(emergency-withdraw)
```

## Error Codes

| Code | Description              |
| ---- | ------------------------ |
| u100 | Unauthorized access      |
| u101 | Duplicate channel ID     |
| u102 | Channel not found        |
| u103 | Balance mismatch         |
| u104 | Invalid signature        |
| u105 | Channel already closed   |
| u106 | Dispute period active    |
| u107 | Invalid parameter format |

## Use Cases

1. **Streaming Payments**:  
   Real-time STX streaming for content platforms

2. **Exchange Settlement**:  
   High-volume trading with net settlement

3. **IoT Microtransactions**:  
   Machine-to-machine value transfers

4. **Gaming Economy**:  
   Instant in-game asset transactions

## Development Guide

### Requirements

- Stacks node v3.0+
- Clarinet SDK
- Bitcoin testnet access

## Audit Considerations

1. Signature verification implementation
2. STX transfer safety checks
3. Dispute deadline calculation
4. Reentrancy protections
5. Integer overflow/underflow
