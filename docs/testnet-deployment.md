# D9 Testnet Deployment Guide

This guide provides step-by-step instructions for deploying a D9 test network with multiple validator nodes.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Network Architecture](#network-architecture)
- [Network Ports & RPC/WebSocket Configuration](#network-ports--rpcwebsocket-configuration)
- [Deployment Steps](#deployment-steps)
  - [1. Prepare Chain Specification](#1-prepare-chain-specification)
  - [2. Deploy Bootnode](#2-deploy-bootnode)
  - [3. Deploy Validator Nodes](#3-deploy-validator-nodes)
  - [4. Generate and Insert Session Keys](#4-generate-and-insert-session-keys)
  - [5. Configure Validators](#5-configure-validators)
  - [6. Start the Network](#6-start-the-network)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### System Requirements (Per Node)

- **Operating System**: Ubuntu 22.04 LTS or Debian 12
- **Architecture**: x86_64 or ARM64
- **RAM**: Minimum 8GB (16GB recommended)
- **Storage**: Minimum 60GB free space (SSD recommended)
- **Network**: Stable internet connection with open ports
- **Ports Required**:
  - 40100 (P2P communication)
  - 40200 (RPC - optional, for monitoring)

### Software Requirements

- D9 node binary (built from source or downloaded)
- jq (for JSON processing)
- Basic knowledge of SSH and Linux command line

## Network Architecture

### Understanding Node Types and Requirements

A D9 network consists of different node types, each with specific purposes and requirements:

#### 1. **Bootnode** (Bootstrap Node)
- **Purpose**: Entry point for new nodes to discover the network
- **Keys Required**: ❌ **NONE** - Bootnodes do NOT need validator keys
- **Function**: Helps nodes find each other (peer discovery)
- **Can be**: A dedicated lightweight node or one of your validators can act as bootnode
- **Recommendation**: Deploy a dedicated bootnode for cleaner architecture

#### 2. **Validator Nodes**
- **Purpose**: Produce and finalize blocks, maintain consensus
- **Keys Required**: ✅ **YES** - Validators MUST have all session keys:
  - **Aura** (Sr25519) - Block production authority
  - **Grandpa** (Ed25519) - Block finality
  - **ImOnline** (Sr25519) - Heartbeat/liveness proof
- **Minimum Required**: **3 validators** for a functional network
- **Recommendation**: Start with 3-4 validators for testing

#### 3. **Full Nodes** (Non-Validating)
- **Purpose**: Sync and store blockchain data, can serve RPC requests
- **Keys Required**: ❌ **NONE** - Full nodes do NOT need validator keys
- **Function**: Provide redundancy, serve as RPC endpoints
- **Use Case**: Public RPC/WSS access point

#### 4. **RPC Node** (Full Node with Public Access)
- **Purpose**: Provide public RPC/WebSocket access for users and applications
- **Keys Required**: ❌ **NONE** - RPC nodes do NOT need validator keys
- **Function**: Handles user transactions and queries
- **Security**: Should run `--rpc-methods Safe` and use Nginx reverse proxy

### Minimum Viable Network Configurations

#### Option 1: Absolute Minimum (3 Nodes)
```
Node 1: Validator + Bootnode (requires keys: Aura, Grandpa, ImOnline)
Node 2: Validator (requires keys: Aura, Grandpa, ImOnline)
Node 3: Validator (requires keys: Aura, Grandpa, ImOnline)
```
**Total**: 3 validators (one doubles as bootnode)
- ✅ Network will function
- ⚠️ No dedicated RPC endpoint
- ⚠️ Validators serving RPC is not recommended for production

#### Option 2: Recommended Minimum (4 Nodes)
```
Node 1: Bootnode (NO keys needed)
Node 2: Validator (requires keys: Aura, Grandpa, ImOnline)
Node 3: Validator (requires keys: Aura, Grandpa, ImOnline)
Node 4: Validator (requires keys: Aura, Grandpa, ImOnline)
```
**Total**: 1 bootnode + 3 validators
- ✅ Cleaner separation of concerns
- ✅ Network will function properly
- ⚠️ Still no dedicated RPC endpoint

#### Option 3: Functional Deployment (5 Nodes) ⭐ RECOMMENDED
```
Node 1: Bootnode (NO keys needed)
Node 2: Validator (requires keys: Aura, Grandpa, ImOnline)
Node 3: Validator (requires keys: Aura, Grandpa, ImOnline)
Node 4: Validator (requires keys: Aura, Grandpa, ImOnline)
Node 5: RPC/WSS Node (NO keys needed)
```
**Total**: 1 bootnode + 3 validators + 1 RPC node
- ✅ Proper network architecture
- ✅ Dedicated public access point
- ✅ Validators are protected
- ✅ Users can interact via RPC node
- ✅ Production-ready setup

### Key Requirements Summary

| Node Type | Needs Aura Key? | Needs Grandpa Key? | Needs ImOnline Key? | Total Keys |
|-----------|----------------|-------------------|-------------------|------------|
| **Bootnode** | ❌ No | ❌ No | ❌ No | 0 |
| **Validator** | ✅ Yes (Sr25519) | ✅ Yes (Ed25519) | ✅ Yes (Sr25519) | 3 |
| **Full Node** | ❌ No | ❌ No | ❌ No | 0 |
| **RPC Node** | ❌ No | ❌ No | ❌ No | 0 |

**Important Notes:**
- Only validators need keys - they must have ALL THREE session keys
- Each validator needs a UNIQUE set of keys (different seed phrase)
- Keys are generated from a seed phrase and inserted into the node's keystore
- Without proper keys, validators cannot participate in consensus
- Bootnodes and RPC nodes never need keys

### Why 3 Validators Minimum?

D9 uses a consensus mechanism that requires:
- **2/3 + 1** validators to agree for finality (Grandpa)
- With 3 validators: Need 3 × 2/3 = 2 validators + 1 = **3 validators** minimum
- With 2 validators: Cannot tolerate any failures (2 × 2/3 = 1.33, rounded up = 2, but need +1)
- **Less than 3 validators = network cannot finalize blocks**

### Typical Production Network (Recommended)

For a production testnet, consider this architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    INTERNET / USERS                         │
│                (Wallets, dApps, Explorers)                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ WSS (443) via Nginx
                         │
                ┌────────▼────────┐
                │   RPC Node      │  (NO keys needed)
                │  (Public WSS)   │  --rpc-methods Safe
                └────────┬────────┘  --rpc-external
                         │           --ws-external
                         │
                         │ P2P (40100)
                         │
        ┌────────────────┴────────────────┐
        │                                 │
┌───────▼────────┐              ┌────────▼────────┐
│   Bootnode     │◄────────────►│   Validator 1   │
│  (NO keys)     │              │ (Aura, Grandpa, │
│                │              │   ImOnline)     │
└───────┬────────┘              └────────┬────────┘
        │                                │
        │         P2P Network            │
        │        (40100)                 │
        │                                │
┌───────▼────────┐              ┌────────▼────────┐
│  Validator 2   │◄────────────►│   Validator 3   │
│ (Aura, Grandpa,│              │ (Aura, Grandpa, │
│   ImOnline)    │              │   ImOnline)     │
└────────────────┘              └─────────────────┘
```

**This setup provides:**
- ✅ Network can reach consensus (3 validators)
- ✅ Easy peer discovery (dedicated bootnode)
- ✅ Public access for users (RPC node)
- ✅ Validators are protected (no public RPC)
- ✅ Scalable (can add more validators and RPC nodes)

## Network Ports & RPC/WebSocket Configuration

### Port Types Explained

D9 nodes use three main types of ports for different communication purposes:

#### 1. **P2P Port (Peer-to-Peer)** - Default: `40100`
- **Purpose**: Inter-node communication for blockchain consensus
- **Protocol**: TCP
- **Usage**: Validators and full nodes discovering each other, syncing blocks, exchanging transactions
- **Must be open to**: Internet (public) for node discovery
- **Security**: No sensitive data, encrypted peer communication
- **Flag**: `--port 40100`

#### 2. **RPC Port (HTTP RPC)** - Default: `9933`
- **Purpose**: HTTP JSON-RPC API for blockchain queries and transactions
- **Protocol**: HTTP over TCP
- **Usage**: Applications, scripts, monitoring tools making one-off requests
- **Should be open to**: Localhost only (unless you need external access)
- **Security**: Can expose sensitive operations - use `--rpc-methods Safe` for public exposure
- **Flag**: `--rpc-port 9933`

#### 3. **WebSocket Port (WS RPC)** - Default: `9944`
- **Purpose**: WebSocket JSON-RPC API for real-time blockchain subscriptions
- **Protocol**: WebSocket (WS) or Secure WebSocket (WSS) over TCP
- **Usage**: Wallets, dApps, block explorers needing real-time updates (block notifications, event subscriptions)
- **Should be open to**: Depends on use case (localhost for development, public with reverse proxy for production)
- **Security**: Same as RPC - restrict methods for public access
- **Flag**: `--ws-port 9944`

### Port Configuration Matrix

| Node Type | P2P (40100) | RPC (9933) | WS (9944) | External Access |
|-----------|-------------|------------|-----------|-----------------|
| **Validator** | Open (Public) | Closed | Closed | None (security) |
| **Full Node (Private)** | Open (Public) | Localhost only | Localhost only | None |
| **RPC Node (Public)** | Open (Public) | Open (Restricted) | Open (Restricted) | Yes (with CORS) |
| **Development** | Localhost | Localhost | Localhost | None |

### Configuration Flags Reference

#### Basic Port Configuration
```bash
--port 40100          # P2P port for node-to-node communication
--rpc-port 9933       # HTTP RPC port
--ws-port 9944        # WebSocket port
```

#### Exposing Ports Externally
```bash
--rpc-external        # Allow external RPC connections (all interfaces: 0.0.0.0)
--ws-external         # Allow external WebSocket connections (all interfaces: 0.0.0.0)
--unsafe-rpc-external # Same as --rpc-external but suppresses security warning
--unsafe-ws-external  # Same as --ws-external but suppresses security warning
```

**Warning**: Using `--rpc-external` or `--ws-external` exposes your node to the internet. Always combine with CORS and method restrictions!

#### CORS Configuration (Cross-Origin Resource Sharing)
```bash
--rpc-cors all                    # Allow all origins (DANGEROUS - development only!)
--rpc-cors "https://app.d9.network"  # Allow specific domain
--rpc-cors "https://app.d9.network,https://wallet.d9.network"  # Multiple domains
--rpc-cors "null"                 # Allow file:// protocol (local HTML files)
```

**Default**: If not specified, only `localhost` and `https://polkadot.js.org` are allowed.

#### Method Restrictions
```bash
--rpc-methods Safe    # Only allow safe, read-only methods (recommended for public nodes)
--rpc-methods Unsafe  # Allow all methods including state-changing operations (default)
```

**Safe methods include**: Reading chain state, querying blocks, subscribing to events
**Unsafe methods include**: Submitting keys, inserting data, developer APIs

#### Connection Limits
```bash
--ws-max-connections 1000  # Maximum concurrent WebSocket connections (default: 100)
--in-peers 25              # Maximum incoming P2P connections (default: 25)
--out-peers 25             # Maximum outgoing P2P connections (default: 25)
```

### Configuration Examples

#### Example 1: Validator Node (Secure - No External Access)
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --name "D9-Validator-1" \
  --validator \
  --port 40100
  # No RPC/WS exposed - only accessible via localhost
```

**Use case**: Production validator - maximum security, no external RPC access

#### Example 2: Full Node with Local RPC Access
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --name "D9-FullNode" \
  --port 40100 \
  --rpc-port 9933 \
  --ws-port 9944
  # RPC/WS available only on localhost (127.0.0.1)
```

**Use case**: Running applications locally that need to query the blockchain

#### Example 3: Public RPC Node (Development Testnet)
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --name "D9-RPC-Public" \
  --port 40100 \
  --rpc-port 9933 \
  --ws-port 9944 \
  --rpc-external \
  --ws-external \
  --rpc-cors all \
  --ws-max-connections 1000
```

**Use case**: Testnet public RPC for developers (NOT recommended for mainnet)

#### Example 4: Public RPC Node (Production - Restricted)
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/mainnet-spec.json \
  --name "D9-RPC-Prod" \
  --port 40100 \
  --rpc-port 9933 \
  --ws-port 9944 \
  --rpc-external \
  --ws-external \
  --rpc-methods Safe \
  --rpc-cors "https://app.d9.network,https://wallet.d9.network" \
  --ws-max-connections 500
```

**Use case**: Production RPC node serving your dApps/wallets with restricted access

### Setting Up a Reverse Proxy (Nginx) for WSS (Secure WebSocket)

For production, you should use **WSS (Secure WebSocket)** with SSL/TLS encryption via a reverse proxy.

#### Step 1: Install Nginx
```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

#### Step 2: Create Nginx Configuration
Create `/etc/nginx/sites-available/d9-rpc`:

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    server_name rpc.yourdomain.com;

    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS server with WebSocket proxy
server {
    listen 443 ssl http2;
    server_name rpc.yourdomain.com;

    # SSL certificates (will be added by certbot)
    ssl_certificate /etc/letsencrypt/live/rpc.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rpc.yourdomain.com/privkey.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # WebSocket proxy to D9 node
    location / {
        proxy_pass http://127.0.0.1:9944;
        proxy_http_version 1.1;

        # WebSocket upgrade headers
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts for long-lived connections
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    # Optional: HTTP RPC endpoint
    location /rpc {
        proxy_pass http://127.0.0.1:9933;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

#### Step 3: Enable Site and Get SSL Certificate
```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/d9-rpc /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Get SSL certificate
sudo certbot --nginx -d rpc.yourdomain.com

# Reload Nginx
sudo systemctl reload nginx
```

#### Step 4: Update Firewall
```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow 40100/tcp  # P2P port
```

#### Step 5: Configure D9 Node (No External Flags Needed!)
```bash
./target/release/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/mainnet-spec.json \
  --name "D9-RPC-Prod" \
  --port 40100 \
  --rpc-port 9933 \
  --ws-port 9944 \
  --rpc-methods Safe
  # No --rpc-external or --ws-external needed!
  # Nginx handles external access
```

**Connection URL**: `wss://rpc.yourdomain.com`

### Connecting to Your Node

#### From Polkadot.js Apps
1. Open [https://polkadot.js.org/apps](https://polkadot.js.org/apps)
2. Click top-left network dropdown
3. Select "Development" → "Custom endpoint"
4. Enter your endpoint:
   - Local: `ws://127.0.0.1:9944`
   - Remote (with Nginx): `wss://rpc.yourdomain.com`
   - Remote (direct, insecure): `ws://YOUR_IP:9944`

#### From JavaScript/TypeScript Application
```javascript
import { ApiPromise, WsProvider } from '@polkadot/api';

// Connect to local node
const wsProvider = new WsProvider('ws://127.0.0.1:9944');

// Connect to remote node with WSS (production)
// const wsProvider = new WsProvider('wss://rpc.yourdomain.com');

const api = await ApiPromise.create({ provider: wsProvider });

// Subscribe to new blocks
await api.rpc.chain.subscribeNewHeads((header) => {
  console.log(`Chain is at block: #${header.number}`);
});
```

#### Using Curl (HTTP RPC)
```bash
# Local node
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  http://localhost:9933

# Remote node
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  https://rpc.yourdomain.com/rpc
```

#### Using Websocat (WebSocket CLI Tool)
```bash
# Install websocat
sudo wget -qO /usr/local/bin/websocat https://github.com/vi/websocat/releases/latest/download/websocat.x86_64-unknown-linux-musl
sudo chmod +x /usr/local/bin/websocat

# Connect to local node
echo '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' | websocat ws://127.0.0.1:9944

# Connect to remote node with WSS
echo '{"id":1, "jsonrpc":"2.0", "method": "chain_getBlock"}' | websocat wss://rpc.yourdomain.com
```

## Understanding Blockchain Interaction & Transaction Submission

### Why You Need RPC/WebSocket Access

Understanding the architecture of blockchain interaction is crucial for anyone building applications, wallets, or services on D9.

#### The Two Communication Layers

**P2P Port (40100) - Validator Communication ONLY**
- Used exclusively for validator-to-validator communication
- Handles block propagation, consensus messages, and peer discovery
- **NOT accessible for user applications**
- **Cannot be used to submit transactions or query blockchain state**
- Only validators and full nodes communicate via this port

**RPC/WebSocket Ports (9933/9944) - User Application Layer**
- **REQUIRED for ALL user interactions with the blockchain**
- Handles transaction submission, state queries, and event subscriptions
- This is how wallets, dApps, and services interact with D9
- Both read (queries) and write (transactions) operations use these ports

#### Why This Separation Exists

```
┌─────────────────────────────────────────────────────────────┐
│                    USER APPLICATIONS                        │
│  (Wallets, dApps, Block Explorers, Your Application)       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ RPC/WebSocket (9933/9944)
                         │ • Submit transactions
                         │ • Query blockchain state
                         │ • Subscribe to events
                         │
┌────────────────────────▼────────────────────────────────────┐
│                    D9 BLOCKCHAIN NODE                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              RPC/WebSocket Interface                 │  │
│  │  • Accepts user transactions                        │  │
│  │  • Processes queries                                │  │
│  │  • Manages subscriptions                            │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                Runtime (Blockchain Logic)            │  │
│  │  • D9 Balances Pallet                               │  │
│  │  • D9 Referral System                               │  │
│  │  • Node Voting Pallet                               │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              P2P Network Interface                   │  │
│  │  • Block propagation                                │  │
│  │  • Consensus messages                               │  │
│  │  • Peer discovery                                   │  │
│  └─────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ P2P (40100)
                         │ VALIDATORS ONLY
                         │
┌────────────────────────▼────────────────────────────────────┐
│              OTHER D9 VALIDATOR NODES                       │
└─────────────────────────────────────────────────────────────┘
```

**Key Takeaway**: If you want to interact with D9 blockchain (check balances, send tokens, vote for validators), you **MUST** connect via RPC/WebSocket. The P2P port is not accessible for this purpose.

### What is a Connector/Client Library?

To interact with D9, you need a **client library** (also called a connector) that:

1. **Manages the connection** to your node via WebSocket or HTTP
2. **Handles cryptography** for signing transactions
3. **Encodes/decodes data** in the correct format for the blockchain
4. **Provides a developer-friendly API** instead of raw JSON-RPC calls

#### Popular Client Libraries for Substrate-based Chains

**For JavaScript/TypeScript (Most Common):**
- `@polkadot/api` - Official Polkadot.js API library
- Full-featured, well-documented, actively maintained
- Works in Node.js and browsers
- **This is what you'll use for D9**

**For Other Languages:**
- Python: `substrate-interface`
- Rust: `subxt`
- Go: `gsrpc`
- Java: `polkaj`

### Read vs Write Operations

Understanding the difference between reading data and writing data is crucial:

#### Read Operations (Queries)

**What they are:**
- Retrieving blockchain state (balances, storage, etc.)
- Subscribing to events (new blocks, transactions)
- Getting chain metadata and information

**Requirements:**
- ✅ RPC/WebSocket connection
- ❌ No account/private key needed
- ❌ No transaction fees
- ❌ No signing required

**Examples:**
- Check account balance
- Get current block number
- Query validator votes
- Subscribe to new blocks
- Read referral relationships

#### Write Operations (Transactions/Extrinsics)

**What they are:**
- Any operation that changes blockchain state
- Also called "extrinsics" in Substrate terminology
- Must be included in a block by validators

**Requirements:**
- ✅ RPC/WebSocket connection
- ✅ Account with private key (for signing)
- ✅ Sufficient balance for transaction fees
- ✅ Valid signature

**Examples:**
- Transfer tokens
- Vote for a validator
- Register as validator candidate
- Update validator commission

**Transaction Lifecycle:**
```
1. Create transaction   → Your application
2. Sign transaction     → Using your private key
3. Submit to node       → Via RPC/WebSocket
4. Enter transaction    → Node's transaction pool
   pool
5. Validator picks      → Next block author
   transaction
6. Include in block     → Block is created
7. Block finalized      → Transaction is permanent
8. Event emitted        → You can subscribe to this
```

### Transaction Submission Requirements

To successfully submit a transaction to D9, you need:

#### 1. WebSocket/RPC Connection
```javascript
import { ApiPromise, WsProvider } from '@polkadot/api';

// Connect to your D9 node
const wsProvider = new WsProvider('ws://127.0.0.1:9944');
const api = await ApiPromise.create({ provider: wsProvider });
```

#### 2. Account with Private Key
```javascript
import { Keyring } from '@polkadot/keyring';

// Create keyring (for managing accounts)
const keyring = new Keyring({ type: 'sr25519' });

// Import existing account from seed phrase
const account = keyring.addFromUri('your twelve word seed phrase here');

// Or create new account
const newAccount = keyring.addFromUri('//Alice'); // Dev account
```

#### 3. Sufficient Balance for Fees
```javascript
// Check balance before transaction
const { data: { free } } = await api.query.system.account(account.address);
console.log(`Free balance: ${free.toHuman()}`);

// Typical transaction fee: 0.01 - 0.1 D9 tokens
// Make sure you have enough!
```

#### 4. Proper Transaction Construction
```javascript
// Create transaction
const transfer = api.tx.d9Balances.transfer(recipientAddress, amount);

// Sign and send
const hash = await transfer.signAndSend(account);
console.log(`Transaction submitted with hash: ${hash}`);
```

### D9-Specific Transaction Examples

D9 has custom pallets with unique functionality. Here are the operations you can perform:

#### D9 Balance Operations

**Transfer Tokens**
```javascript
import { ApiPromise, WsProvider } from '@polkadot/api';
import { Keyring } from '@polkadot/keyring';

// Setup
const wsProvider = new WsProvider('ws://127.0.0.1:9944');
const api = await ApiPromise.create({ provider: wsProvider });
const keyring = new Keyring({ type: 'sr25519' });
const sender = keyring.addFromUri('//Alice');

// Transfer 100 tokens (adjust decimals based on your chain config)
const recipient = 'Dn5FHneW52S7C8VmMJgMU34zG8hh9MAYWn1Cv8GfPLscYmMzS';
const amount = 100 * 10**12; // Assuming 12 decimals

// Create and send transaction
const transfer = api.tx.d9Balances.transfer(recipient, amount);

// Subscribe to transaction status
const unsub = await transfer.signAndSend(sender, ({ status, events }) => {
  if (status.isInBlock) {
    console.log(`Transaction included in block: ${status.asInBlock}`);

    // Check for successful transfer event
    events.forEach(({ event }) => {
      if (api.events.d9Balances.Transfer.is(event)) {
        const [from, to, amount] = event.data;
        console.log(`Transfer: ${from} → ${to}: ${amount}`);
      }
    });
  } else if (status.isFinalized) {
    console.log(`Transaction finalized in block: ${status.asFinalized}`);
    unsub(); // Unsubscribe
  }
});
```

#### Node Voting Operations

**Register as Validator Candidate**
```javascript
// Register as validator with 10% commission
const commission = 10; // 10%

const register = api.tx.d9NodeVoting.registerAsCandidate(commission);

await register.signAndSend(account, ({ status }) => {
  if (status.isFinalized) {
    console.log('Successfully registered as validator candidate!');
  }
});
```

**Vote for a Validator**
```javascript
// Vote for validator with your tokens
const validatorAddress = 'Dn5FHneW52S7C8VmMJgMU34zG8hh9MAYWn1Cv8GfPLscYmMzS';
const voteAmount = 1000 * 10**12; // 1000 tokens

const vote = api.tx.d9NodeVoting.voteForNode(validatorAddress, voteAmount);

await vote.signAndSend(account, ({ status, events }) => {
  if (status.isFinalized) {
    console.log(`Voted ${voteAmount} for validator ${validatorAddress}`);

    // Check for VoteCast event
    events.forEach(({ event }) => {
      if (api.events.d9NodeVoting.VoteCast.is(event)) {
        console.log('Vote successfully recorded!');
      }
    });
  }
});
```

**Remove Vote from Validator**
```javascript
// Remove your vote from a validator
const removeVote = api.tx.d9NodeVoting.removeVote(validatorAddress);

await removeVote.signAndSend(account, ({ status }) => {
  if (status.isFinalized) {
    console.log('Vote removed successfully');
  }
});
```

**Update Validator Commission**
```javascript
// Update your commission rate as a validator
const newCommission = 15; // 15%

const updateCommission = api.tx.d9NodeVoting.updateCommission(newCommission);

await updateCommission.signAndSend(account, ({ status }) => {
  if (status.isFinalized) {
    console.log(`Commission updated to ${newCommission}%`);
  }
});
```

#### Query Operations (Read-Only)

These operations don't require signing or fees:

**Query Account Balance**
```javascript
// Get account balance
const accountInfo = await api.query.system.account(address);
console.log(`Free balance: ${accountInfo.data.free.toHuman()}`);
console.log(`Reserved: ${accountInfo.data.reserved.toHuman()}`);
console.log(`Nonce: ${accountInfo.nonce}`);
```

**Query Referral Relationships**
```javascript
// Get parent (referrer) of an account
const parent = await api.query.d9Referral.referralRelationships(address);

if (parent.isSome) {
  console.log(`Referred by: ${parent.unwrap()}`);
} else {
  console.log('No referrer (root account)');
}

// Get referral statistics
const directReferrals = await api.query.d9Referral.directReferralCount(address);
console.log(`Direct referrals: ${directReferrals}`);
```

**Query Validator Votes**
```javascript
// Get total votes for a validator
const votes = await api.query.d9NodeVoting.candidateVotes(validatorAddress);
console.log(`Total votes: ${votes.toHuman()}`);

// Get validator commission rate
const commission = await api.query.d9NodeVoting.validatorCommission(validatorAddress);
console.log(`Commission: ${commission}%`);

// Get your voting interests
const myVotes = await api.query.d9NodeVoting.votingInterests(myAddress);
console.log('My votes:', myVotes.toHuman());
```

**Subscribe to New Blocks**
```javascript
// Subscribe to new block headers
const unsubscribe = await api.rpc.chain.subscribeNewHeads((header) => {
  console.log(`New block #${header.number}: ${header.hash}`);
});

// Later: unsubscribe()
```

**Subscribe to Events**
```javascript
// Subscribe to all system events
api.query.system.events((events) => {
  events.forEach((record) => {
    const { event } = record;

    // Filter for specific events
    if (api.events.d9Balances.Transfer.is(event)) {
      const [from, to, amount] = event.data;
      console.log(`Transfer: ${from} → ${to}: ${amount.toHuman()}`);
    }

    if (api.events.d9NodeVoting.VoteCast.is(event)) {
      const [voter, candidate, amount] = event.data;
      console.log(`Vote cast: ${voter} → ${candidate}: ${amount.toHuman()}`);
    }
  });
});
```

### Complete Working Example: Transfer with Error Handling

Here's a production-ready example with proper error handling:

```javascript
import { ApiPromise, WsProvider } from '@polkadot/api';
import { Keyring } from '@polkadot/keyring';

async function transferTokens() {
  let api;

  try {
    // 1. Connect to node
    console.log('Connecting to D9 node...');
    const wsProvider = new WsProvider('ws://127.0.0.1:9944');
    api = await ApiPromise.create({ provider: wsProvider });
    console.log('Connected!');

    // 2. Setup account
    const keyring = new Keyring({ type: 'sr25519' });
    const sender = keyring.addFromUri('your seed phrase here');

    // 3. Check balance
    const { data: { free } } = await api.query.system.account(sender.address);
    console.log(`Sender balance: ${free.toHuman()}`);

    const amount = 10 * 10**12; // 10 tokens

    if (free.lt(amount)) {
      throw new Error('Insufficient balance for transfer + fees');
    }

    // 4. Create transaction
    const recipient = 'Dn5FHneW52S7C8VmMJgMU34zG8hh9MAYWn1Cv8GfPLscYmMzS';
    const transfer = api.tx.d9Balances.transfer(recipient, amount);

    // 5. Estimate fees (optional)
    const info = await transfer.paymentInfo(sender);
    console.log(`Estimated fee: ${info.partialFee.toHuman()}`);

    // 6. Sign and send
    console.log('Sending transaction...');

    await new Promise((resolve, reject) => {
      transfer.signAndSend(sender, ({ status, events, dispatchError }) => {
        // Check for errors
        if (dispatchError) {
          if (dispatchError.isModule) {
            const decoded = api.registry.findMetaError(dispatchError.asModule);
            const { docs, name, section } = decoded;
            reject(new Error(`${section}.${name}: ${docs.join(' ')}`));
          } else {
            reject(new Error(dispatchError.toString()));
          }
          return;
        }

        // Transaction status updates
        if (status.isInBlock) {
          console.log(`Transaction included in block: ${status.asInBlock}`);
        } else if (status.isFinalized) {
          console.log(`Transaction finalized in block: ${status.asFinalized}`);

          // Parse events
          events.forEach(({ event }) => {
            if (api.events.d9Balances.Transfer.is(event)) {
              const [from, to, value] = event.data;
              console.log(`✅ Transfer successful: ${value.toHuman()}`);
            }
          });

          resolve(status.asFinalized);
        }
      });
    });

    console.log('Transfer completed successfully!');

  } catch (error) {
    console.error('Error:', error.message);
    throw error;
  } finally {
    // Always disconnect
    if (api) {
      await api.disconnect();
    }
  }
}

// Run the function
transferTokens().catch(console.error);
```

### For Wallet & dApp Developers

If you're building a wallet or dApp on D9, here's what you need to know:

#### Chain Configuration

```javascript
// D9 Chain configuration
const D9_CONFIG = {
  // Connection
  wsEndpoint: 'wss://rpc.d9.network', // Mainnet
  // wsEndpoint: 'ws://localhost:9944', // Local

  // Chain properties
  ss58Format: 9, // D9's SS58 address format (starts with 'Dn')
  tokenDecimals: 12, // Adjust based on your chain
  tokenSymbol: 'D9',

  // Network
  chainName: 'D9 Network',
  chainType: 'Live', // or 'Development', 'TestNet'
};

// Create API with custom config
const api = await ApiPromise.create({
  provider: new WsProvider(D9_CONFIG.wsEndpoint),
  types: {}, // Add custom types if needed
});
```

#### Custom Types (if needed)

If D9 uses custom types not in the standard Substrate types, you'll need to provide them:

```javascript
const customTypes = {
  // Example custom types
  VotingInterest: {
    candidate: 'AccountId',
    amount: 'Balance',
    lastUpdate: 'BlockNumber',
  },
  // Add other custom types as needed
};

const api = await ApiPromise.create({
  provider: new WsProvider(endpoint),
  types: customTypes,
});
```

#### Address Formatting

D9 uses a custom SS58 format (addresses start with "Dn"):

```javascript
import { encodeAddress, decodeAddress } from '@polkadot/util-crypto';

// Convert to D9 format
function toD9Address(address) {
  const publicKey = decodeAddress(address);
  return encodeAddress(publicKey, 9); // 9 is D9's SS58 format
}

// Example
const genericAddress = '5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY';
const d9Address = toD9Address(genericAddress);
console.log(d9Address); // Dn...
```

#### Metadata and Chain Info

```javascript
// Get chain metadata
const metadata = await api.rpc.state.getMetadata();
console.log('Chain metadata version:', metadata.version);

// Get chain info
const chain = await api.rpc.system.chain();
const version = await api.rpc.system.version();
const properties = await api.rpc.system.properties();

console.log(`Connected to: ${chain}`);
console.log(`Version: ${version}`);
console.log(`Properties:`, properties.toHuman());
```

### Common Use Cases: What Requires RPC/WS?

Here's a comprehensive table showing what operations require RPC/WebSocket access:

| Operation | Requires RPC/WS | Requires Signing | Requires Fees | Can Use P2P? |
|-----------|----------------|------------------|---------------|--------------|
| **Check account balance** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Transfer tokens** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **Query referral parent** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Register as validator** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **Vote for validator** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **Remove vote** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **Update commission** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **Query validator votes** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Subscribe to new blocks** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Subscribe to events** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Get chain metadata** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Check transaction status** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Validator consensus** | ❌ No | N/A | N/A | ✅ Yes (P2P only) |

**Key Insights:**
- **100% of user-facing operations** require RPC/WS
- **0% of user operations** can use P2P port
- P2P port is exclusively for validator-to-validator communication
- Read operations don't require signing or fees, but still need RPC/WS
- Write operations always require signing, fees, and RPC/WS

### Quick Start for Developers

**Install Dependencies:**
```bash
npm install @polkadot/api @polkadot/keyring @polkadot/util-crypto
```

**Minimal Working Example:**
```javascript
import { ApiPromise, WsProvider } from '@polkadot/api';
import { Keyring } from '@polkadot/keyring';

async function main() {
  // Connect
  const api = await ApiPromise.create({
    provider: new WsProvider('ws://127.0.0.1:9944')
  });

  // Get chain info
  const chain = await api.rpc.system.chain();
  console.log(`Connected to ${chain}`);

  // Query balance (read)
  const address = 'Dn5FHneW52S7C8VmMJgMU34zG8hh9MAYWn1Cv8GfPLscYmMzS';
  const { data: balance } = await api.query.system.account(address);
  console.log(`Balance: ${balance.free.toHuman()}`);

  // Send transaction (write)
  const keyring = new Keyring({ type: 'sr25519' });
  const sender = keyring.addFromUri('//Alice');

  const tx = api.tx.d9Balances.transfer(address, 1000000);
  await tx.signAndSend(sender);

  await api.disconnect();
}

main().catch(console.error);
```

### Troubleshooting Connection Issues

**Cannot connect to WebSocket:**
```bash
# Test if WebSocket port is accessible
telnet localhost 9944

# Check if node is listening
sudo netstat -tlnp | grep 9944
```

**Connection refused:**
- Ensure node is running with `--ws-port 9944`
- For external access, add `--ws-external`
- Check firewall allows port 9944

**CORS errors in browser:**
- Add `--rpc-cors all` for development
- For production, specify allowed origins: `--rpc-cors "https://yourdomain.com"`

**Transaction fails:**
- Check account has sufficient balance
- Verify transaction parameters are correct
- Check for dispatch errors in event logs

### Summary

**Remember these key points:**

1. ✅ **RPC/WebSocket is REQUIRED** for all user interactions with D9
2. ❌ **P2P port cannot** be used for transactions or queries
3. 🔐 **Transactions require** signing with private key + fees
4. 📖 **Queries are free** but still need RPC/WS connection
5. 📚 **Use @polkadot/api** for JavaScript/TypeScript applications
6. 🔒 **Keep private keys secure** - never expose them in client code
7. 🌐 **Use WSS (secure WebSocket)** for production deployments

**When deploying your testnet, ensure:**
- Validators have P2P port (40100) open
- RPC nodes have WS port (9944) open
- Use Nginx reverse proxy for WSS in production
- Apply `--rpc-methods Safe` for public nodes
- Properly configure CORS for your domains

### Security Best Practices

#### For Validators
- **NEVER** expose RPC/WS ports (`--rpc-external` / `--ws-external`)
- Use firewall to block ports 9933, 9944 from internet
- Only allow P2P port (40100)
- Use SSH tunneling for remote management

#### For Public RPC Nodes
- Always use `--rpc-methods Safe`
- Use reverse proxy (Nginx) with SSL/TLS (WSS)
- Implement rate limiting at proxy level
- Restrict CORS to known domains (not `all`)
- Monitor for abuse
- Use separate server from validators

#### For Development Nodes
- Use `--rpc-cors all` only on testnet/development
- Never use on mainnet
- Consider using Docker with network isolation

### Firewall Configuration Examples

#### Validator Node
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 40100/tcp  # P2P only
sudo ufw enable
```

#### Public RPC Node (with Nginx)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 40100/tcp      # P2P
sudo ufw allow 'Nginx Full'   # HTTP/HTTPS (80/443)
sudo ufw enable
```

#### Public RPC Node (direct access - NOT recommended)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 40100/tcp  # P2P
sudo ufw allow 9933/tcp   # HTTP RPC (consider limiting to specific IPs)
sudo ufw allow 9944/tcp   # WebSocket (consider limiting to specific IPs)
sudo ufw enable
```

### Troubleshooting Connection Issues

#### Cannot connect to remote WebSocket
```bash
# Check if port is open
telnet YOUR_IP 9944

# Check if node is listening on correct interface
sudo netstat -tlnp | grep 9944

# Should show 0.0.0.0:9944 (all interfaces) if using --ws-external
# Shows 127.0.0.1:9944 (localhost only) if not
```

#### CORS errors in browser console
```
Access to XMLHttpRequest has been blocked by CORS policy
```

**Solution**: Add your domain to `--rpc-cors`:
```bash
--rpc-cors "https://yourdomain.com"
```

#### Connection refused
- Ensure node is running: `sudo systemctl status d9-node`
- Check firewall: `sudo ufw status`
- Verify port configuration in systemd service file

#### WebSocket closes immediately
- Check if using Safe methods on public node: `--rpc-methods Safe`
- Verify SSL certificates if using WSS
- Check Nginx logs: `sudo tail -f /var/log/nginx/error.log`

---

## Understanding Substrate & D9 Architecture

Before deploying your testnet, it's helpful to understand what you're actually building. This section explains the core concepts of Substrate and what makes D9 special.

### What is Substrate?

**Substrate** is a **blockchain development framework** created by Parity Technologies (the team behind Polkadot).

Think of it as:
- **Not a blockchain** itself, but a **toolkit** for building blockchains
- Like Ruby on Rails for web apps, but for blockchains
- Provides all the hard infrastructure (networking, consensus, database) so you focus on your business logic

**Key advantages**:
- Battle-tested code (powers Polkadot and 100+ other chains)
- Highly modular and customizable
- Built-in upgrade capability (no hard forks needed!)
- Rich ecosystem of tools and libraries

### Runtime vs Node: The Core Split

Substrate blockchains have two main components:

#### 1. **The Node** (Outer Layer)
- Handles **infrastructure** concerns
- Networking: P2P communication between nodes
- Database: Storing blockchain data
- RPC: API for external applications
- Consensus: Coordinating block production

**Think**: The "plumbing" that every blockchain needs

#### 2. **The Runtime** (Inner Layer - Your Business Logic)
- Defines **what your blockchain does**
- Written in Rust, compiled to WebAssembly (WASM)
- Contains all your custom logic (pallets)
- Can be upgraded without restarting the chain!

**Think**: The "application code" that makes your chain unique

```
┌─────────────────────────────────────────┐
│            USER APPLICATIONS            │
│         (Wallets, dApps, etc.)          │
└──────────────────┬──────────────────────┘
                   │ RPC/WebSocket
                   │
┌──────────────────▼──────────────────────┐
│           SUBSTRATE NODE                │
│  ┌─────────────────────────────────┐   │
│  │         RPC Server              │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │      P2P Networking             │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │    Database (RocksDB)           │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │   Consensus (Aura + Grandpa)    │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │      RUNTIME (WASM)             │   │
│  │                                 │   │
│  │  ┌──────────────────────────┐  │   │
│  │  │  D9 Balances Pallet      │  │   │
│  │  └──────────────────────────┘  │   │
│  │  ┌──────────────────────────┐  │   │
│  │  │  D9 Referral Pallet      │  │   │
│  │  └──────────────────────────┘  │   │
│  │  ┌──────────────────────────┐  │   │
│  │  │  D9 Node Voting Pallet   │  │   │
│  │  └──────────────────────────┘  │   │
│  │  ┌──────────────────────────┐  │   │
│  │  │  D9 Treasury Pallet      │  │   │
│  │  └──────────────────────────┘  │   │
│  │  ┌──────────────────────────┐  │   │
│  │  │  ... more pallets        │  │   │
│  │  └──────────────────────────┘  │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### WASM: Why Your Runtime is Upgradeable

**WebAssembly (WASM)** is the secret sauce that makes Substrate special:

1. **Runtime is stored on-chain** as WASM bytecode
2. **Runtime can be upgraded** without stopping the chain
3. **No hard forks needed** - just submit a transaction with new runtime
4. **All nodes automatically** use the new runtime

**This means**:
- Fix bugs without coordinating node upgrades
- Add new features seamlessly
- Users don't need to update anything

### Consensus: Aura + Grandpa

D9 uses a **hybrid consensus** mechanism with two components:

#### Aura (Block Production)
- **Authority Round** consensus
- Validators take turns producing blocks in a round-robin fashion
- Each validator gets a time slot (e.g., 6 seconds)
- Fast and efficient for known validator sets

**Analogy**: Like taking turns speaking in a meeting - everyone gets their turn.

#### Grandpa (Finality)
- **GHOST-based Recursive Ancestor Deriving Prefix Agreement**
- Finalizes batches of blocks (not just one at a time)
- Requires 2/3+ of validators to agree
- Even if some validators are offline, finality continues

**Analogy**: Like signing off on a batch of documents - more efficient than signing each one individually.

**Together**:
1. **Aura** produces blocks quickly (~6 seconds)
2. **Grandpa** finalizes them in batches (~6-12 seconds lag)
3. **Result**: Fast block production + strong finality guarantees

### Pallets: Modular Building Blocks

**Pallets** are modules that add functionality to your runtime.

Think of them as:
- Plugins or packages
- Self-contained pieces of logic
- Reusable across different chains

**Standard Substrate Pallets** (used by D9):
- `pallet-aura` - Block production
- `pallet-grandpa` - Finality
- `pallet-session` - Validator session management
- `pallet-im-online` - Validator heartbeats
- `pallet-timestamp` - On-chain time
- `pallet-sudo` - Superuser access (for development)

**D9 Custom Pallets** (what makes D9 unique):

| Pallet | Purpose |
|--------|---------|
| `pallet-d9-balances` | Token transfers with referral integration |
| `pallet-d9-referral` | Multi-level referral system (up to 19 levels) |
| `pallet-d9-treasury` | Community-controlled fund management |
| `pallet-d9-node-voting` | Vote for validators, validator selection |
| `pallet-d9-node-rewards` | Distribute rewards to validators and voters |
| `pallet-d9-council-lock` | Council governance and voting |
| `pallet-d9-multi-sig` | Multi-signature wallet functionality |

### What Makes D9 Special?

D9 is **not just another Substrate chain** - it has unique features:

#### 1. **Built-in Referral System**
- Every account can have a referrer (parent)
- Referral relationships are on-chain and permanent
- Automatically integrated with balance transfers
- Supports up to 19 levels of depth
- Enables viral growth and network effects

#### 2. **Community-Driven Validator Selection**
- Traditional chains: Validators are fixed or chosen by stake
- D9: Community votes for validators using D9 tokens
- More votes = higher chance of becoming active validator
- Voters share in validator rewards

#### 3. **Treasury with Proposals**
- Community-controlled funds
- Anyone can propose spending
- Council approves proposals
- Transparent fund allocation

#### 4. **Integrated Reward Distribution**
- Validators earn block rewards
- Voters earn rewards proportional to their support
- Referral bonuses can be applied
- Compound options available

### Session Keys: Why Validators Need Three Keys

Validators need **three different keys** because each serves a different purpose:

#### 1. **Aura Key** (Sr25519)
- **Purpose**: Proves authority to produce blocks
- **Used when**: Your validator's turn to create a block
- **What happens without it**: Validator can't produce blocks

#### 2. **Grandpa Key** (Ed25519)
- **Purpose**: Participates in finality voting
- **Used when**: Voting to finalize blocks
- **What happens without it**: Validator can't help finalize blocks

#### 3. **ImOnline Key** (Sr25519)
- **Purpose**: Proves validator is online and responsive
- **Used when**: Sending periodic heartbeat messages
- **What happens without it**: Network thinks validator is offline

**Why different cryptographic schemes?**
- Sr25519: Schnorr signatures - fast, good for frequent operations
- Ed25519: EdDSA signatures - specific properties needed for Grandpa

**All three keys** are derived from the **same seed phrase** using different derivation paths:
- Aura: `seed_phrase`
- Grandpa: `seed_phrase//grandpa`
- ImOnline: `seed_phrase//im_online`

### The Genesis Block (Chain Spec)

Every blockchain starts with a **genesis block** - block #0.

The **chain specification** defines:
- Initial state (who has tokens, who are validators)
- Runtime code (WASM)
- Chain ID (must be unique)
- Consensus parameters (block time, etc.)

**Critical**: All nodes must use the **exact same chain spec** or they won't connect!

### Block Production Flow

Here's what happens when a new block is created:

```
1. Time Slot Arrives
   ↓
2. Aura: Check whose turn it is
   ↓
3. Validator collects transactions from pool
   ↓
4. Validator executes transactions (Runtime)
   ↓
5. Validator creates block and signs with Aura key
   ↓
6. Block broadcast to all peers via P2P
   ↓
7. Other validators import block
   ↓
8. Grandpa: Validators vote on finality
   ↓
9. Once 2/3+ vote: Block is finalized
   ↓
10. Finalized block cannot be reverted
```

**Typical timing**:
- Block production: ~6 seconds
- Finality: ~6-12 seconds (1-2 blocks behind)

### Transaction Lifecycle

When you submit a transaction:

```
1. User signs transaction with private key
   ↓
2. Submit via RPC to any node
   ↓
3. Node validates signature and fees
   ↓
4. Transaction enters mempool (transaction pool)
   ↓
5. Block producer picks transaction from pool
   ↓
6. Transaction executed in Runtime
   ↓
7. Transaction included in block
   ↓
8. Block imported by all nodes
   ↓
9. Block finalized by Grandpa
   ↓
10. Transaction is permanent on-chain
```

### Storage: How Data is Stored

Substrate uses **key-value storage** with a **Merkle tree** structure:

- **State**: Current value of all storage items
- **State root**: Hash of entire state (in block header)
- **Changing state root**: Proves state changed
- **Storage proofs**: Can prove a value existed at a specific block

**D9 Storage Examples**:
- Account balances: `d9Balances -> AccountId -> Balance`
- Referrals: `d9Referral -> AccountId -> ParentAccountId`
- Validator votes: `d9NodeVoting -> ValidatorId -> TotalVotes`

### Events: Watching What Happens

When something happens on-chain, the Runtime emits **events**:

**Examples**:
- `d9Balances.Transfer` - Tokens transferred
- `d9NodeVoting.VoteCast` - Vote recorded
- `d9Treasury.Proposed` - New spending proposal

**Why events matter**:
- Applications subscribe to events for real-time updates
- Block explorers use events to show activity
- Your dApp needs events to update UI

### Why This Architecture Matters

Understanding this architecture helps you:

1. **Debug issues**: Know where to look (node vs runtime)
2. **Optimize performance**: Understand bottlenecks
3. **Design applications**: Know what's possible
4. **Upgrade safely**: Understand what can change
5. **Contribute**: Know where to add features

### Further Learning

**Official Resources**:
- [Substrate Docs](https://docs.substrate.io) - Comprehensive guides
- [Substrate Tutorials](https://docs.substrate.io/tutorials/) - Hands-on learning
- [Polkadot Wiki](https://wiki.polkadot.network) - Deeper concepts

**D9-Specific**:
- [D9 Pallet Documentation](./pallets/README.md) - D9 custom pallets
- [D9 Architecture](./architecture/README.md) - D9 design decisions

---

## Deployment Steps

### 1. Prepare Chain Specification

The chain specification defines the genesis state and network parameters.

#### Option A: Use Existing Testnet Spec

If you have `testnet-spec.json` in your repository:

```bash
# On your local machine or build server
cd ~/d9-node
cp testnet-spec.json /tmp/testnet-spec.json

# Copy to each node
scp /tmp/testnet-spec.json user@node1:/usr/local/bin/testnet-spec.json
scp /tmp/testnet-spec.json user@node2:/usr/local/bin/testnet-spec.json
# ... repeat for all nodes
```

#### Option B: Generate Custom Testnet Spec

```bash
# Generate a new chain specification
./target/release/d9-node build-spec --chain testnet --disable-default-bootnode > testnet-spec-plain.json

# Edit testnet-spec-plain.json to customize:
# - Initial balances
# - Initial validators
# - Network parameters

# Convert to raw format
./target/release/d9-node build-spec --chain testnet-spec-plain.json --raw > testnet-spec.json
```

---

## Understanding Chain Specifications (Genesis Configuration)

### What is a Chain Specification?

The chain specification (chain spec) is the **genesis configuration** for your blockchain. It defines:
- **Initial state**: Account balances, validator set, governance parameters
- **Runtime**: The blockchain logic (WASM bytecode)
- **Network ID**: Unique identifier for your chain
- **Consensus rules**: Block time, epoch length, etc.

Think of it as the **"birth certificate"** of your blockchain - it must be identical on all nodes or they won't connect.

### Plain vs Raw Format

**Plain Format** (`testnet-spec-plain.json`):
- Human-readable JSON
- You can edit this directly
- Contains clear field names
- Must be converted to raw before use

**Raw Format** (`testnet-spec.json`):
- Encoded/hashed values
- Used by the actual node
- Cannot be edited directly
- Generated from plain format

**Workflow**: Edit Plain → Convert to Raw → Distribute Raw to all nodes

### Complete Workflow: From Seed to Running Validator

Here's the complete workflow visualized:

```
┌─────────────────────────────────────────────────────────────────┐
│                    VALIDATOR SETUP WORKFLOW                     │
└─────────────────────────────────────────────────────────────────┘

STEP 1: Generate Seed Phrases (On Secure Machine)
┌──────────────────────────────────────────────────────────────┐
│ ./d9-node key generate --scheme Sr25519 --words 12          │
│                                                              │
│ Output: "word1 word2 word3 ... word12"                      │
│ Secret seed: 0xabc123...                                    │
│ Public key (SS58): Dn5FLSig...                              │
└──────────────────────────────────────────────────────────────┘
         │
         │ Repeat for each validator (SEED1, SEED2, SEED3)
         ↓

STEP 2: Derive Public Keys (For Chain Spec)
┌──────────────────────────────────────────────────────────────┐
│ # For each seed, get three public keys:                     │
│                                                              │
│ Aura:       ./d9-node key inspect --scheme Sr25519 "$SEED"  │
│ Grandpa:    ./d9-node key inspect --scheme Ed25519 \        │
│               "$SEED//grandpa"                               │
│ ImOnline:   ./d9-node key inspect --scheme Sr25519 \        │
│               "$SEED//im_online"                             │
│                                                              │
│ Account:    ./d9-node key inspect --network reynolds "$SEED"│
└──────────────────────────────────────────────────────────────┘
         │
         │ Copy public keys to notepad
         ↓

STEP 3: Create Chain Specification
┌──────────────────────────────────────────────────────────────┐
│ # Generate base spec                                         │
│ ./d9-node build-spec --chain testnet \                      │
│   --disable-default-bootnode > testnet-plain.json           │
│                                                              │
│ # Edit testnet-plain.json:                                  │
│   - Add balances                                            │
│   - Add session.keys with public keys from Step 2          │
│   - Set genesis validators                                  │
│                                                              │
│ # Convert to raw                                            │
│ ./d9-node build-spec --chain testnet-plain.json \           │
│   --raw > testnet.json                                      │
└──────────────────────────────────────────────────────────────┘
         │
         │ Distribute testnet.json to all nodes
         ↓

STEP 4: Deploy Chain Spec to All Nodes
┌──────────────────────────────────────────────────────────────┐
│ scp testnet.json user@validator1:/usr/local/bin/            │
│ scp testnet.json user@validator2:/usr/local/bin/            │
│ scp testnet.json user@validator3:/usr/local/bin/            │
│ scp testnet.json user@bootnode:/usr/local/bin/              │
└──────────────────────────────────────────────────────────────┘
         │
         ↓

STEP 5: Insert Private Keys into Validators (CRITICAL!)
┌──────────────────────────────────────────────────────────────┐
│ # On Validator 1 (use SEED1)                                │
│ ./d9-node key insert --base-path /home/ubuntu/node-data \   │
│   --chain /usr/local/bin/testnet.json \                     │
│   --scheme Sr25519 --suri "$SEED1" --key-type aura          │
│                                                              │
│ ./d9-node key insert --base-path /home/ubuntu/node-data \   │
│   --chain /usr/local/bin/testnet.json \                     │
│   --scheme Ed25519 --suri "$SEED1//grandpa" --key-type gran │
│                                                              │
│ ./d9-node key insert --base-path /home/ubuntu/node-data \   │
│   --chain /usr/local/bin/testnet.json \                     │
│   --scheme Sr25519 --suri "$SEED1//im_online" \             │
│   --key-type imon                                            │
│                                                              │
│ # Repeat on Validator 2 with SEED2                          │
│ # Repeat on Validator 3 with SEED3                          │
└──────────────────────────────────────────────────────────────┘
         │
         │ Keys now in /home/ubuntu/node-data/chains/*/keystore/
         ↓

STEP 6: Verify Keys Match Chain Spec
┌──────────────────────────────────────────────────────────────┐
│ # On each validator, check keystore                         │
│ ls -la /home/ubuntu/node-data/chains/*/keystore/            │
│                                                              │
│ Should see 3 files:                                          │
│   617572...     (aura - Sr25519)                             │
│   6772616e...   (grandpa - Ed25519)                          │
│   696d6f6e...   (imon - Sr25519)                             │
│                                                              │
│ # Verify public keys match chain spec                       │
│ cat testnet.json | jq '.genesis.runtime.session.keys'       │
└──────────────────────────────────────────────────────────────┘
         │
         ↓

STEP 7: Start Nodes
┌──────────────────────────────────────────────────────────────┐
│ # Start bootnode first                                       │
│ systemctl start d9-node                                      │
│                                                              │
│ # Get bootnode peer ID from logs                            │
│ journalctl -u d9-node -n 50 | grep "Local node identity"    │
│                                                              │
│ # Start validators with bootnode address                    │
│ systemctl start d9-node                                      │
└──────────────────────────────────────────────────────────────┘
         │
         ↓

STEP 8: Verify Network is Running
┌──────────────────────────────────────────────────────────────┐
│ # Check logs on each node                                   │
│ journalctl -u d9-node -f                                     │
│                                                              │
│ Look for:                                                    │
│   ✓ "Discovered new external address"                       │
│   ✓ "Syncing, target=#X (3 peers)"                          │
│   ✓ "Prepared block for proposing"                          │
│   ✓ "Imported #1"                                            │
│   ✓ "Finalized #1"                                           │
└──────────────────────────────────────────────────────────────┘

SUCCESS: Network is producing and finalizing blocks! 🎉
```

### Anatomy of a Chain Spec

Here's what a plain chain spec contains:

```json
{
  "name": "D9 Testnet",
  "id": "d9_testnet",
  "chainType": "Live",
  "bootNodes": [],
  "telemetryEndpoints": null,
  "protocolId": "d9",
  "properties": {
    "tokenSymbol": "D9",
    "tokenDecimals": 12,
    "ss58Format": 9
  },
  "genesis": {
    "runtime": {
      // Initial configuration for each pallet
      "system": {},
      "d9Balances": {
        "balances": [
          // Pre-funded accounts
        ]
      },
      "session": {
        "keys": [
          // Initial validator session keys
        ]
      },
      // ... more pallets
    }
  }
}
```

### Customizing Your Chain Spec

#### 1. Set Chain Identity

```json
{
  "name": "My D9 Testnet",
  "id": "my_d9_testnet",
  "chainType": "Live",  // or "Development", "Local"
  "protocolId": "my-d9"
}
```

**Important**:
- `id` must be unique (nodes with different IDs won't connect)
- `protocolId` should match your network name

#### 2. Configure Token Properties

```json
"properties": {
  "tokenSymbol": "D9T",  // Your token symbol
  "tokenDecimals": 12,    // Number of decimal places
  "ss58Format": 9         // Address format (9 = addresses start with "Dn")
}
```

#### 3. Add Initial Balances

Pre-fund accounts for testing:

```json
"d9Balances": {
  "balances": [
    ["Dn5FHneW52S7C8VmMJgMU34zG8hh9MAYWn1Cv8GfPLscYmMzS", 1000000000000000],
    ["Dn5GNW8sW9R9Z7K3xH8YmXq4R7vZ1pJ8xC4vK9mN2sT5eP3aQ", 500000000000000]
  ]
}
```

**Note**: Balance amounts are in the smallest unit (with 12 decimals, so 1000000000000000 = 1000 tokens)

#### 4. Configure Initial Validators (CRITICAL)

Your initial validators must be in the chain spec for the network to start:

```json
"session": {
  "keys": [
    [
      "Dn5FHneW52...",  // Validator account (stash)
      "Dn5FHneW52...",  // Validator account (controller - same as stash for testnet)
      {
        "aura": "5GrwvaEF...",      // Aura public key (Sr25519)
        "grandpa": "5FA9nQDV...",   // Grandpa public key (Ed25519)
        "im_online": "5GNJqTP..."   // ImOnline public key (Sr25519)
      }
    ],
    // Add validator 2
    [
      "Dn5GNW8sW9...",
      "Dn5GNW8sW9...",
      {
        "aura": "5HpG9w8E...",
        "grandpa": "5GvvtdD1...",
        "im_online": "5EfLKF8..."
      }
    ],
    // Add validator 3 (minimum 3 for network to function!)
    [
      "Dn5DqM3nP7...",
      "Dn5DqM3nP7...",
      {
        "aura": "5FLSigC9...",
        "grandpa": "5DFBuDd5...",
        "im_online": "5FoLWcX..."
      }
    ]
  ]
}
```

**How to get these values:**

```bash
# For each validator, generate keys and get public keys
SEED_PHRASE="your twelve word seed phrase here"

# Get Aura public key
./target/release/d9-node key inspect --scheme Sr25519 "$SEED_PHRASE"

# Get Grandpa public key
./target/release/d9-node key inspect --scheme Ed25519 "$SEED_PHRASE//grandpa"

# Get ImOnline public key
./target/release/d9-node key inspect --scheme Sr25519 "$SEED_PHRASE//im_online"

# Get account address (for stash/controller)
./target/release/d9-node key inspect --network reynolds "$SEED_PHRASE"
```

#### 5. Configure Authority Set (Aura)

```json
"aura": {
  "authorities": []  // Leave empty - populated from session.keys
}
```

#### 6. Configure Grandpa

```json
"grandpa": {
  "authorities": []  // Leave empty - populated from session.keys
}
```

### Complete Example: Creating a Custom Testnet

**Step 1: Generate base spec**

```bash
./target/release/d9-node build-spec --chain testnet --disable-default-bootnode > custom-testnet-plain.json
```

**Step 2: Generate 3 validator key sets**

```bash
# Validator 1
SEED1=$(./target/release/d9-node key generate --scheme Sr25519 --words 12 | grep "Secret phrase" | cut -d':' -f2- | xargs)
echo "Validator 1 Seed: $SEED1"

# Validator 2
SEED2=$(./target/release/d9-node key generate --scheme Sr25519 --words 12 | grep "Secret phrase" | cut -d':' -f2- | xargs)
echo "Validator 2 Seed: $SEED2"

# Validator 3
SEED3=$(./target/release/d9-node key generate --scheme Sr25519 --words 12 | grep "Secret phrase" | cut -d':' -f2- | xargs)
echo "Validator 3 Seed: $SEED3"

# SAVE THESE SEEDS SECURELY!
```

**Step 3: Get public keys for each validator**

```bash
# For Validator 1
echo "=== Validator 1 ==="
./target/release/d9-node key inspect --scheme Sr25519 "$SEED1" | grep "Public key"
./target/release/d9-node key inspect --scheme Ed25519 "$SEED1//grandpa" | grep "Public key"
./target/release/d9-node key inspect --scheme Sr25519 "$SEED1//im_online" | grep "Public key"
./target/release/d9-node key inspect --network reynolds "$SEED1" | grep "SS58 Address"

# Repeat for Validator 2 and 3...
```

**Step 4: Edit the chain spec**

Open `custom-testnet-plain.json` and:
1. Update name, id, protocolId
2. Add initial balances
3. Add all 3 validators to `session.keys`

**Step 5: Convert to raw format**

```bash
./target/release/d9-node build-spec --chain custom-testnet-plain.json --raw > custom-testnet.json
```

**Step 6: Distribute to all nodes**

```bash
# Copy to each server
scp custom-testnet.json user@validator1:/usr/local/bin/testnet-spec.json
scp custom-testnet.json user@validator2:/usr/local/bin/testnet-spec.json
scp custom-testnet.json user@validator3:/usr/local/bin/testnet-spec.json
scp custom-testnet.json user@bootnode:/usr/local/bin/testnet-spec.json
scp custom-testnet.json user@rpc-node:/usr/local/bin/testnet-spec.json
```

**Step 7: Insert keys on validators**

On each validator, insert the session keys corresponding to that validator's entry in the chain spec:

```bash
# On Validator 1 (using SEED1)
./target/release/d9-node key insert --base-path /home/ubuntu/node-data --chain /usr/local/bin/testnet-spec.json --scheme Sr25519 --suri "$SEED1" --key-type aura
./target/release/d9-node key insert --base-path /home/ubuntu/node-data --chain /usr/local/bin/testnet-spec.json --scheme Ed25519 --suri "$SEED1//grandpa" --key-type gran
./target/release/d9-node key insert --base-path /home/ubuntu/node-data --chain /usr/local/bin/testnet-spec.json --scheme Sr25519 --suri "$SEED1//im_online" --key-type imon
```

### Common Chain Spec Mistakes

#### ❌ **Mistake 1: Different chain specs on different nodes**
```
Error: "Verification failed: Execution failed: Consensus"
```
**Fix**: Ensure EXACT same raw chain spec on all nodes (same file, same hash)

#### ❌ **Mistake 2: Forgot to convert plain to raw**
```
Error: "Failed to decode chain spec"
```
**Fix**: Always use `build-spec --raw` before distributing

#### ❌ **Mistake 3: Less than 3 validators in chain spec**
```
Network starts but never finalizes blocks
```
**Fix**: Add minimum 3 validators to `session.keys`

#### ❌ **Mistake 4: Wrong SS58 format**
```
Addresses look wrong or don't start with "Dn"
```
**Fix**: Set `ss58Format: 9` in properties

#### ❌ **Mistake 5: Validator keys don't match chain spec**
```
Validator doesn't produce blocks
```
**Fix**: Ensure keystore keys match the public keys in chain spec

#### ❌ **Mistake 6: Forgot bootNodes in chain spec**
```
Nodes can't discover each other
```
**Fix**: Either add bootnode multiaddress to chain spec, or use `--bootnodes` flag

### Verifying Your Chain Spec

```bash
# Check chain spec is valid
./target/release/d9-node build-spec --chain testnet-spec.json

# Get chain spec hash (should be same on all nodes)
./target/release/d9-node build-spec --chain testnet-spec.json 2>&1 | grep "Chain"

# Inspect what's in the spec
cat testnet-spec.json | jq .name
cat testnet-spec.json | jq .properties
```

### Tips for Testing

1. **Start simple**: Use the default testnet spec first before customizing
2. **Version control**: Keep your plain chain spec in git
3. **Document seeds**: Maintain a secure record of which seed corresponds to which validator
4. **Test locally**: Try running all nodes on one machine first (different ports)
5. **Unique IDs**: Use different chain IDs for different testnets to avoid confusion

---

### 2. Deploy Bootnode

The bootnode helps other nodes discover each other through peer discovery.

**IMPORTANT**: ❌ **Bootnodes do NOT need validator keys** (Aura, Grandpa, ImOnline)
- Bootnodes only facilitate peer discovery
- They do NOT participate in consensus
- They only need a node key (for P2P identification)

#### Generate Bootnode Key

On the bootnode server:

```bash
# Generate a unique node key (P2P identity only)
./target/release/d9-node key generate-node-key > /tmp/bootnode-key.txt

# Save the peer ID for later use
BOOTNODE_PEER_ID=$(./target/release/d9-node key inspect-node-key --file /tmp/bootnode-key.txt)
echo "Bootnode Peer ID: $BOOTNODE_PEER_ID"
```

**Note**: This is just a P2P network identity, NOT validator session keys.

#### Start Bootnode

```bash
# Create bootnode directory
mkdir -p /home/ubuntu/bootnode-data

# Start bootnode (NO --validator flag, NO session keys needed)
./target/release/d9-node \
  --base-path /home/ubuntu/bootnode-data \
  --chain /usr/local/bin/testnet-spec.json \
  --name "D9-Bootnode" \
  --node-key-file /tmp/bootnode-key.txt \
  --port 40100 \
  --rpc-port 40200 \
  --unsafe-rpc-external \
  --rpc-cors all \
  --no-telemetry
```

**Notice**: No `--validator` flag because bootnodes are NOT validators!

#### Get Bootnode Address

```bash
# The bootnode multiaddress format:
# /ip4/<BOOTNODE_IP>/tcp/40100/p2p/<BOOTNODE_PEER_ID>

# Example:
# /ip4/192.168.1.100/tcp/40100/p2p/12D3KooWEyoppNCUx8Yx66oV9fJnriXwCcXwDDUA2kj6vnc6iDEp
```

Replace `<BOOTNODE_IP>` with your bootnode's public IP address.

Save this bootnode address - you'll need it when starting validators.

### 3. Deploy Validator Nodes

**Deploy at least 3 validator nodes for a functional testnet.**

**CRITICAL**: ✅ **Validators MUST have ALL session keys:**
- **Aura** (Sr25519) - For block production
- **Grandpa** (Ed25519) - For block finality
- **ImOnline** (Sr25519) - For heartbeat/liveness

**Without these keys, validators cannot participate in consensus and your network will not produce blocks!**

#### Install D9 Node on Each Validator

On each validator server (node1, node2, node3, etc.):

```bash
# Option 1: Use installation script
curl -sSf https://raw.githubusercontent.com/D-Nine-Chain/d9_node/main/scripts/install-d9-node.sh -o install-d9-node.sh
chmod +x install-d9-node.sh
./install-d9-node.sh

# Option 2: Build from source
curl -sSf https://raw.githubusercontent.com/D-Nine-Chain/d9_node/main/scripts/build-node.sh | bash
```

#### Copy Chain Specification

```bash
# On each validator
sudo cp testnet-spec.json /usr/local/bin/
```

### 4. Generate and Insert Session Keys

**CRITICAL - Read This First!**

✅ **Validators need ALL THREE session keys:**
1. **Aura** (Sr25519) - Block production authority - produces blocks
2. **Grandpa** (Ed25519) - Finality gadget - finalizes blocks
3. **ImOnline** (Sr25519) - Heartbeat mechanism - proves validator is online

**Each validator must have a UNIQUE set of keys** (different seed phrase per validator)

❌ **Full nodes and bootnodes do NOT need these keys**
- Only validators participating in consensus need session keys
- RPC nodes, full nodes, and bootnodes run without session keys

**What happens without keys:**
- Missing Aura key → Validator cannot produce blocks
- Missing Grandpa key → Validator cannot finalize blocks
- Missing ImOnline key → Validator appears offline
- Network needs 3 validators with complete keys to function

#### Generate Keys for Validator 1

```bash
# Stop the service if running
sudo systemctl stop d9-node.service

# Generate a new seed phrase (save this securely!)
SEED_PHRASE=$(./target/release/d9-node key generate --scheme Sr25519 --words 12 | grep "Secret phrase:" | cut -d':' -f2- | xargs)
echo "Validator 1 Seed Phrase: $SEED_PHRASE"
echo "SAVE THIS PHRASE SECURELY!"

# Insert Aura key
./target/release/d9-node key insert \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --scheme Sr25519 \
  --suri "${SEED_PHRASE}" \
  --key-type aura

# Insert Grandpa key
./target/release/d9-node key insert \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --scheme Ed25519 \
  --suri "${SEED_PHRASE}//grandpa" \
  --key-type gran

# Insert ImOnline key
./target/release/d9-node key insert \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --scheme Sr25519 \
  --suri "${SEED_PHRASE}//im_online" \
  --key-type imon

# Get the validator address
ADDRESS_JSON=$(./target/release/d9-node key inspect --network reynolds --output-type json "${SEED_PHRASE}")
SS58_ADDRESS=$(echo "$ADDRESS_JSON" | jq -r '.ss58Address')
echo "Validator 1 Address: Dn${SS58_ADDRESS}"
```

#### Repeat for All Validators

Repeat the above process for each validator node, saving each seed phrase and address securely.

### 5. Configure Validators

#### Create Systemd Service for Each Validator

On each validator, create or update `/etc/systemd/system/d9-node.service`:

```ini
[Unit]
Description=D9 Validator Node
After=network.target

[Service]
Type=simple
User=ubuntu
ExecStart=/usr/local/bin/d9-node \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --name "D9-Validator-1" \
  --validator \
  --port 40100 \
  --rpc-port 40200 \
  --bootnodes /ip4/<BOOTNODE_IP>/tcp/40100/p2p/<BOOTNODE_PEER_ID>

Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Replace:
- `D9-Validator-1` with unique names for each validator
- `<BOOTNODE_IP>` with your bootnode IP
- `<BOOTNODE_PEER_ID>` with your bootnode peer ID

#### Enable and Start Services

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable d9-node.service

# Start the validator
sudo systemctl start d9-node.service

# Check status
sudo systemctl status d9-node.service
```

### 6. Start the Network

#### Start Bootnode First

```bash
# On bootnode server
sudo systemctl start d9-node.service
journalctl -u d9-node.service -f
```

Wait until you see log messages indicating the node is running.

#### Start All Validators

```bash
# On each validator server
sudo systemctl start d9-node.service

# Monitor logs
journalctl -u d9-node.service -f
```

You should see messages like:
- `Discovered new external address`
- `Peer connected`
- `Preparing 0.0 bps`
- `Imported #1`

## Network Testing & Verification

### Complete Test Plan

After deploying your D9 testnet, follow this comprehensive test plan to ensure everything is working correctly.

---

### Phase 1: Node Connectivity (Immediate)

#### 1.1 Check All Nodes Are Running

```bash
# On each node (bootnode, validators, RPC node)
sudo systemctl status d9-node.service

# Expected: Active (running)
```

#### 1.2 Verify Peer Discovery

```bash
# On any validator, check peer count
journalctl -u d9-node.service -f

# Look for:
# ✓ "Discovered new external address"
# ✓ "Peer connected" (should see 3-4 peers minimum)
```

**Expected Result**: Each validator should connect to:
- Bootnode (1 peer)
- Other validators (2-3 peers)
- Total: 3-4 peers minimum

#### 1.3 Check Network Health via RPC

```bash
# Test on RPC node or validator with RPC enabled
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  http://localhost:40200

# Expected output:
{
  "jsonrpc": "2.0",
  "result": {
    "peers": 3,           # At least 3
    "isSyncing": false,   # Should be false
    "shouldHavePeers": true
  },
  "id": 1
}
```

#### 1.4 List Connected Peers

```bash
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_peers"}' \
  http://localhost:40200 | jq .

# Should show peer info for bootnode + other validators
```

---

### Phase 2: Validator Keys (5 minutes)

#### 2.1 Verify Session Keys in Keystore

On **each validator** (not bootnode or RPC node):

```bash
# Check keystore has all required keys
ls -la /home/ubuntu/node-data/chains/*/keystore/

# You MUST see exactly 3 files per validator:
# - 617572* (Aura - Sr25519)
# - 6772616e* (Grandpa - Ed25519)
# - 696d6f6e* (ImOnline - Sr25519)
```

**If missing any keys**: Your validator won't work! Go back to [Section 4: Generate and Insert Session Keys](#4-generate-and-insert-session-keys)

#### 2.2 Verify Keys Match Chain Spec

```bash
# On validator, check what keys are loaded
journalctl -u d9-node.service | grep -i "loaded"

# Should see messages about loading Aura, Grandpa, ImOnline keys
```

---

### Phase 3: Block Production (10 minutes)

#### 3.1 Monitor Block Import

```bash
# On any node, watch for blocks
journalctl -u d9-node.service -f | grep "Imported"

# Expected output (blocks should appear every ~6 seconds):
# Imported #1 (0x1234...)
# Imported #2 (0x5678...)
# Imported #3 (0x9abc...)
```

**Wait 2-3 minutes** - You should see steadily increasing block numbers.

#### 3.2 Check Current Block Height

```bash
# Get latest block number
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "chain_getHeader"}' \
  http://localhost:40200 | jq .result.number

# Should show current block height (hex format)
# Example: "0x64" = block 100
```

#### 3.3 Verify All Validators Are Producing

```bash
# Check logs for "Prepared block for proposing"
journalctl -u d9-node.service | grep "Prepared block"

# Each of your 3 validators should appear in rotation
```

**Expected**: All 3 validators take turns producing blocks (round-robin).

---

### Phase 4: Finality (10 minutes)

#### 4.1 Check Finality is Advancing

```bash
# Watch for finality messages
journalctl -u d9-node.service -f | grep -i "finalized"

# Expected output:
# Finalized block #10
# Finalized block #11
```

Finality should advance every 6-12 seconds (approximately every 1-2 blocks).

#### 4.2 Verify Grandpa is Working

```bash
# Check for Grandpa voter activity
journalctl -u d9-node.service | grep -i "grandpa"

# Look for:
# - "Grandpa voter"
# - "finality: Imported"
```

#### 4.3 Check Finality Lag

```bash
# Get best block and finalized block
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "chain_getHeader"}' \
  http://localhost:40200 | jq -r .result.number > /tmp/best.txt

curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "chain_getFinalizedHead"}' \
  http://localhost:40200 | jq -r .result > /tmp/finalized.txt

# Finality lag should be < 5 blocks
```

**Problem if**: Finality lag grows continuously → Check all validators have Grandpa keys.

---

### Phase 5: D9-Specific Features (15 minutes)

#### 5.1 Test Balance Transfer

Use Polkadot.js Apps or @polkadot/api:

```bash
# Connect to your RPC node
# URL: ws://<RPC_NODE_IP>:9944 or wss://rpc.yourdomain.com

# Try transferring tokens between pre-funded accounts
```

**Expected**: Transaction appears in block, balance updates correctly.

#### 5.2 Test Referral System Query

```javascript
// Using @polkadot/api
const parent = await api.query.d9Referral.referralRelationships(address);
console.log('Referrer:', parent.toHuman());

// Should return referral data if set up in chain spec
```

#### 5.3 Test Node Voting

```javascript
// Query validator candidates
const candidates = await api.query.d9NodeVoting.candidateList();
console.log('Candidates:', candidates.toHuman());

// Query votes for a validator
const votes = await api.query.d9NodeVoting.candidateVotes(validatorAddress);
console.log('Votes:', votes.toHuman());
```

#### 5.4 Verify ImOnline Heartbeats

```bash
# Check validators are sending heartbeats
journalctl -u d9-node.service | grep -i "heartbeat"

# All 3 validators should send heartbeats regularly
```

---

### Phase 6: RPC Node (if deployed)

#### 6.1 Test External RPC Access

From your local machine (not the server):

```bash
# Test HTTP RPC
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_chain"}' \
  http://<RPC_NODE_IP>:9933

# Test WebSocket (using websocat or browser)
echo '{"id":1, "jsonrpc":"2.0", "method": "system_chain"}' | \
  websocat ws://<RPC_NODE_IP>:9944
```

#### 6.2 Test WSS (if using Nginx)

```bash
# Test secure WebSocket
echo '{"id":1, "jsonrpc":"2.0", "method": "system_chain"}' | \
  websocat wss://rpc.yourdomain.com
```

#### 6.3 Connect with Polkadot.js Apps

1. Open https://polkadot.js.org/apps
2. Click top-left dropdown
3. Select "Development" → "Custom endpoint"
4. Enter: `wss://rpc.yourdomain.com` or `ws://<IP>:9944`
5. Should connect and show chain data

---

### Phase 7: Performance Baseline

#### 7.1 Check Block Time

```bash
# Monitor block production rate
journalctl -u d9-node.service -f | grep "Imported" | ts '[%Y-%m-%d %H:%M:%S]'

# Blocks should appear every ~6 seconds
```

#### 7.2 Check Memory Usage

```bash
# On each validator
free -h
ps aux | grep d9-node

# Typical usage: 500MB - 2GB RAM per node
```

#### 7.3 Check Disk Usage

```bash
# Check database size
du -sh /home/ubuntu/node-data/

# After 1 hour: ~500MB - 1GB
# After 24 hours: ~2GB - 5GB
# After 1 week: ~10GB - 20GB
```

---

### Success Criteria

Your testnet is **successfully deployed** if ALL of these are true:

**Network Health**:
- [ ] All nodes are running (`systemctl status` shows active)
- [ ] Each validator has 3-4 connected peers
- [ ] Bootnode shows connections from all validators

**Block Production**:
- [ ] New blocks appearing every ~6 seconds
- [ ] Block height continuously increasing
- [ ] All 3 validators producing blocks in rotation

**Finality**:
- [ ] Finality advancing (blocks being finalized)
- [ ] Finality lag < 5 blocks
- [ ] No stalls in finality for > 30 seconds

**Validator Keys**:
- [ ] Each validator has 3 keys in keystore (Aura, Grandpa, ImOnline)
- [ ] No "missing key" errors in logs
- [ ] Heartbeats visible in logs

**Transactions**:
- [ ] Can submit a balance transfer via RPC
- [ ] Transaction appears in a block within 12 seconds
- [ ] Balance updates correctly after transaction

**D9 Features**:
- [ ] Can query D9 pallet state (balances, referrals, voting)
- [ ] D9-specific transactions work (if tested)

**RPC Access** (if deployed):
- [ ] RPC node responds to queries
- [ ] Can connect from external client
- [ ] WSS works (if configured with Nginx)

---

### Performance Expectations

**Normal Operation**:
- **Block time**: ~6 seconds per block
- **Finality**: 1-2 blocks behind best block
- **Peer count**: 3-4 peers per validator
- **CPU usage**: 5-20% per validator node
- **Memory**: 500MB - 2GB per node
- **Network**: < 1 Mbps per node

**Warning Signs**:
- ⚠️ Finality lag > 10 blocks
- ⚠️ No new blocks for > 30 seconds
- ⚠️ Peer count dropping to 0-1
- ⚠️ Memory usage > 4GB
- ⚠️ CPU constantly at 100%

If you see warning signs, check the [Troubleshooting](#troubleshooting) section.

---

### Quick Verification Script

Save this script to quickly verify your network status:

```bash
#!/bin/bash
# verify-d9-network.sh

echo "=== D9 Testnet Verification ==="
echo ""

# Check service status
echo "1. Checking node status..."
systemctl is-active d9-node.service || echo "❌ Node not running!"

# Check peers
echo ""
echo "2. Checking peer count..."
PEERS=$(curl -s -H "Content-Type: application/json" -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' http://localhost:40200 | jq -r .result.peers)
echo "Connected peers: $PEERS"
[ "$PEERS" -ge 3 ] && echo "✅ Good peer count" || echo "⚠️  Low peer count"

# Check block height
echo ""
echo "3. Checking block height..."
BLOCK=$(curl -s -H "Content-Type: application/json" -d '{"id":1, "jsonrpc":"2.0", "method": "chain_getHeader"}' http://localhost:40200 | jq -r .result.number)
echo "Current block: $BLOCK"

# Check keystore
echo ""
echo "4. Checking validator keys..."
KEY_COUNT=$(ls /home/ubuntu/node-data/chains/*/keystore/ 2>/dev/null | wc -l)
echo "Keys in keystore: $KEY_COUNT"
[ "$KEY_COUNT" -eq 3 ] && echo "✅ All keys present" || echo "⚠️  Missing keys!"

echo ""
echo "=== Verification Complete ==="
```

Run it with:
```bash
chmod +x verify-d9-network.sh
./verify-d9-network.sh
```

## Troubleshooting

### Nodes Not Connecting

**Problem**: Validators cannot connect to bootnode or each other.

**Solutions**:
```bash
# 1. Verify firewall allows port 40100
sudo ufw allow 40100/tcp

# 2. Check bootnode is running
ssh user@bootnode "sudo systemctl status d9-node.service"

# 3. Verify bootnode address is correct
# Format: /ip4/<IP>/tcp/40100/p2p/<PEER_ID>

# 4. Check network connectivity
ping <bootnode-ip>
telnet <bootnode-ip> 40100
```

### No Blocks Being Produced

**Problem**: Network starts but no blocks are produced.

**Solutions**:
```bash
# 1. Verify session keys are inserted correctly
ls -la /home/ubuntu/node-data/chains/*/keystore/

# 2. Check validator count matches chain spec
# Minimum validators required: 3

# 3. Ensure all validators are running
# Check logs on all nodes

# 4. Restart validators in sequence
sudo systemctl restart d9-node.service
```

### Insufficient Peers

**Problem**: Nodes show 0 or 1 peers.

**Solutions**:
```bash
# 1. Add multiple bootnodes
# Edit systemd service to include multiple bootnodes:
--bootnodes /ip4/<IP1>/tcp/40100/p2p/<PEER1> \
--bootnodes /ip4/<IP2>/tcp/40100/p2p/<PEER2>

# 2. Check NAT/firewall configuration
# Ensure P2P port is accessible from internet

# 3. Verify chain specification matches on all nodes
md5sum /usr/local/bin/testnet-spec.json
```

### Database Corruption

**Problem**: Node crashes with database errors.

**Solutions**:
```bash
# Stop the node
sudo systemctl stop d9-node.service

# Remove database (will resync from network)
rm -rf /home/ubuntu/node-data/chains/*/db

# Restart node
sudo systemctl start d9-node.service
```

### Keys Not Working

**Problem**: Validator keys don't seem to be recognized.

**Solutions**:
```bash
# 1. Verify key format in keystore
ls -la /home/ubuntu/node-data/chains/*/keystore/

# 2. Re-insert keys with correct scheme
# Aura: Sr25519
# Grandpa: Ed25519
# ImOnline: Sr25519

# 3. Check chain specification matches
# The --chain parameter must match the chain used for key insertion

# 4. Restart node after inserting keys
sudo systemctl restart d9-node.service
```

### High Resource Usage

**Problem**: Node consuming too much CPU/RAM.

**Solutions**:
```bash
# 1. Monitor resource usage
htop

# 2. Adjust execution strategy in service file
--wasm-execution Compiled

# 3. Limit database cache
--db-cache 512

# 4. Add to systemd service:
[Service]
MemoryLimit=8G
CPUQuota=400%
```

## Useful Commands Reference

### Node Management

```bash
# Start node
sudo systemctl start d9-node.service

# Stop node
sudo systemctl stop d9-node.service

# Restart node
sudo systemctl restart d9-node.service

# View status
sudo systemctl status d9-node.service

# View logs (live)
journalctl -u d9-node.service -f

# View logs (last 100 lines)
journalctl -u d9-node.service -n 100

# View logs (last hour)
journalctl -u d9-node.service --since "1 hour ago"
```

### Key Management

```bash
# Generate new seed
./target/release/d9-node key generate --scheme Sr25519 --words 12

# Inspect seed (get address)
./target/release/d9-node key inspect --network reynolds "your seed phrase here"

# Insert Aura key
./target/release/d9-node key insert \
  --base-path /home/ubuntu/node-data \
  --chain /usr/local/bin/testnet-spec.json \
  --scheme Sr25519 \
  --suri "your seed phrase" \
  --key-type aura

# List keystore contents
ls -la /home/ubuntu/node-data/chains/*/keystore/
```

### Network Diagnostics

```bash
# Check system health
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  http://localhost:40200

# List connected peers
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_peers"}' \
  http://localhost:40200

# Get node version
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_version"}' \
  http://localhost:40200

# Get chain name
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_chain"}' \
  http://localhost:40200
```

## Common Pitfalls & FAQs

This section provides quick answers to the most common issues teams encounter when deploying D9 testnets.

### Network Won't Produce Blocks

**Symptom**: Network starts, nodes connect, but no blocks are produced (`best: #0`).

**Quick Diagnosis**:
```bash
# Check if validators have session keys
ls -la /home/ubuntu/node-data/chains/*/keystore/
# Should see files starting with: 617572 (aura), 6772616e (grandpa), 696d6f6e (imon)

# Check number of running validators
# You need MINIMUM 3 validators for block production
```

**Common Causes & Solutions**:

1. **Insufficient Validators**
   - **Problem**: Less than 3 validators running
   - **Fix**: Start at least 3 validator nodes with proper session keys
   - **Why**: Grandpa finality requires 2/3 + 1 validators (2 out of 3 minimum)

2. **Missing Session Keys**
   - **Problem**: Validators running but keys not inserted
   - **Check**: `ls /home/ubuntu/node-data/chains/*/keystore/` shows empty or incomplete
   - **Fix**: Insert all three required keys (aura, grandpa, imon) on EACH validator
   ```bash
   # You need ALL THREE key types on each validator:
   ./target/release/d9-node key insert --base-path /home/ubuntu/node-data \
     --chain /usr/local/bin/testnet-spec.json --scheme Sr25519 \
     --suri "your seed" --key-type aura

   ./target/release/d9-node key insert --base-path /home/ubuntu/node-data \
     --chain /usr/local/bin/testnet-spec.json --scheme Ed25519 \
     --suri "your seed//grandpa" --key-type gran

   ./target/release/d9-node key insert --base-path /home/ubuntu/node-data \
     --chain /usr/local/bin/testnet-spec.json --scheme Sr25519 \
     --suri "your seed//im_online" --key-type imon

   # Restart after inserting keys
   sudo systemctl restart d9-node.service
   ```

3. **Wrong Chain Specification**
   - **Problem**: Keys inserted with different `--chain` than runtime
   - **Check**: Ensure same chain spec file used everywhere
   - **Fix**: Re-insert keys with correct chain spec path

4. **Validators Not in Genesis**
   - **Problem**: Validator addresses not in initial authorities in chain spec
   - **Fix**: Regenerate chain spec with correct validator addresses in `palletSession.keys`

---

### Validators Not Finalizing Blocks

**Symptom**: Blocks are produced (`best: #123`) but not finalized (`finalized: #0`).

**Quick Diagnosis**:
```bash
# Check logs for Grandpa messages
journalctl -u d9-node.service -n 100 | grep -i grandpa

# Look for: "Authority set" or "Finalizing" messages
```

**Common Causes & Solutions**:

1. **Grandpa Keys Missing or Wrong Scheme**
   - **Problem**: Grandpa keys must use Ed25519 (not Sr25519)
   - **Check**: Keystore should have files starting with `6772616e` (hex for "gran")
   - **Fix**:
   ```bash
   # CORRECT: Ed25519 scheme for Grandpa
   ./target/release/d9-node key insert \
     --scheme Ed25519 \
     --suri "your seed//grandpa" \
     --key-type gran \
     --base-path /home/ubuntu/node-data \
     --chain /usr/local/bin/testnet-spec.json
   ```

2. **Less Than 3 Validators**
   - **Problem**: Grandpa needs 2/3 supermajority (minimum 3 validators)
   - **Fix**: Deploy at least 3 validator nodes

3. **Network Partition**
   - **Problem**: Validators can't communicate with each other
   - **Check**: `system_peers` should show connections between validators
   - **Fix**: Verify firewall allows P2P port (40100) between all validators

---

### Peers Not Connecting

**Symptom**: Node shows `0 peers` or can't discover network.

**Quick Diagnosis**:
```bash
# Check system health
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  http://localhost:40200 | jq

# Should show: "peers": 3 or more
```

**Common Causes & Solutions**:

1. **Bootnode Address Wrong or Unreachable**
   - **Problem**: Invalid bootnode multiaddr format
   - **Correct Format**: `/ip4/1.2.3.4/tcp/40100/p2p/12D3KooW...`
   - **Fix**:
     - Get bootnode peer ID: `journalctl -u d9-node.service | grep "Local node identity"`
     - Verify IP is public/accessible
     - Test connectivity: `telnet <bootnode-ip> 40100`

2. **Firewall Blocking P2P Port**
   - **Problem**: Port 40100 not open
   - **Fix**:
   ```bash
   # On all nodes
   sudo ufw allow 40100/tcp
   sudo ufw reload

   # Test from another node
   telnet <node-ip> 40100
   ```

3. **Different Chain Specifications**
   - **Problem**: Nodes using different genesis blocks
   - **Check**: Compare chain spec hashes on all nodes:
   ```bash
   md5sum /usr/local/bin/testnet-spec.json
   # Hash MUST match on all nodes
   ```
   - **Fix**: Copy exact same chain spec file to all nodes

4. **NAT/Port Forwarding Issues**
   - **Problem**: Nodes behind NAT can't accept incoming connections
   - **Fix**: Configure port forwarding on router or use `--public-addr`:
   ```bash
   --public-addr /ip4/<PUBLIC_IP>/tcp/40100
   ```

---

### Transaction Stuck in Pool

**Symptom**: Transaction submitted but never included in block.

**Quick Diagnosis**:
```bash
# Check transaction pool
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "author_pendingExtrinsics"}' \
  http://localhost:40200 | jq
```

**Common Causes & Solutions**:

1. **Insufficient Balance for Fees**
   - **Problem**: Account has balance but not enough for transaction fee
   - **Fix**: Ensure account has at least 0.01 D9 extra for fees
   ```javascript
   // Check both free and reserved balances
   const { data: { free, reserved } } = await api.query.system.account(address);
   console.log(`Free: ${free}, Reserved: ${reserved}`);
   ```

2. **Invalid Nonce**
   - **Problem**: Nonce out of sequence (gap or duplicate)
   - **Check**: Compare account nonce vs transaction nonce
   ```javascript
   const { nonce } = await api.query.system.account(address);
   console.log(`Current nonce: ${nonce}`);
   // Next transaction must use this nonce
   ```
   - **Fix**: Resubmit with correct nonce

3. **Transaction Dropped (Low Priority)**
   - **Problem**: Pool full, transaction has low priority
   - **Fix**: Increase tip:
   ```javascript
   await api.tx.balances.transfer(recipient, amount)
     .signAndSend(sender, { tip: 1000000000 }); // Add tip
   ```

4. **Node Not Synced**
   - **Problem**: Submitting to node that's not caught up
   - **Check**: `best` should equal `finalized` (±1-2 blocks)
   - **Fix**: Wait for sync or connect to synced node

---

### Keys Not Being Recognized

**Symptom**: Validator keys inserted but node doesn't use them for block production.

**Quick Diagnosis**:
```bash
# List keystore files
ls -l /home/ubuntu/node-data/chains/*/keystore/

# Should see 3 files per validator:
# 617572... (aura - Sr25519)
# 6772616e... (grandpa - Ed25519)
# 696d6f6e... (imon - Sr25519)
```

**Common Causes & Solutions**:

1. **Wrong Key Scheme**
   - **Problem**: Using wrong cryptographic scheme for key type
   - **Correct Schemes**:
     - Aura: **Sr25519**
     - Grandpa: **Ed25519** ⚠️ (commonly wrong - people use Sr25519)
     - ImOnline: **Sr25519**
   - **Fix**: Delete wrong keys and re-insert with correct scheme

2. **Keystore Path Mismatch**
   - **Problem**: Keys inserted to different `--base-path` than runtime uses
   - **Check**: Service file `--base-path` matches key insertion command
   - **Fix**: Ensure both use same path (e.g., `/home/ubuntu/node-data`)

3. **Chain Spec Mismatch**
   - **Problem**: Keys inserted with different `--chain` than service uses
   - **Fix**:
   ```bash
   # Service file and key insertion MUST use same chain spec
   # Service: --chain /usr/local/bin/testnet-spec.json
   # Keys: --chain /usr/local/bin/testnet-spec.json
   ```

4. **Node Not Restarted After Insertion**
   - **Problem**: Keys inserted but node not restarted
   - **Fix**:
   ```bash
   sudo systemctl restart d9-node.service
   journalctl -u d9-node.service -f
   # Watch for "Loaded block-authoring keys" message
   ```

5. **File Permissions**
   - **Problem**: Keystore files not readable by node process
   - **Fix**:
   ```bash
   sudo chown -R ubuntu:ubuntu /home/ubuntu/node-data
   chmod 600 /home/ubuntu/node-data/chains/*/keystore/*
   ```

---

### RPC/WebSocket Connection Fails

**Symptom**: Cannot connect to RPC node via Polkadot.js or custom app.

**Quick Diagnosis**:
```bash
# Test RPC endpoint
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
  http://<node-ip>:40200

# Test WebSocket (from browser console)
const ws = new WebSocket('ws://<node-ip>:9944');
ws.onopen = () => console.log('Connected!');
ws.onerror = (e) => console.error('Error:', e);
```

**Common Causes & Solutions**:

1. **RPC Not Exposed Externally**
   - **Problem**: Node only listening on localhost (127.0.0.1)
   - **Fix**: Add flags to systemd service:
   ```ini
   --rpc-external \
   --rpc-cors all \
   --ws-external
   ```
   ⚠️ **Security**: Only expose RPC/WS on dedicated RPC node, NOT validators

2. **Firewall Blocking Ports**
   - **Problem**: Ports 40200 (RPC) and 9944 (WS) not open
   - **Fix**:
   ```bash
   sudo ufw allow 40200/tcp
   sudo ufw allow 9944/tcp
   sudo ufw reload
   ```

3. **CORS Issues**
   - **Problem**: Browser blocks connection due to CORS policy
   - **Symptoms**: Console shows "CORS policy" error
   - **Fix**: Add `--rpc-cors all` or specific origins:
   ```bash
   --rpc-cors "https://polkadot.js.org,http://localhost:3000"
   ```

4. **Wrong Port in Connection String**
   - **Problem**: Connecting to wrong port
   - **Defaults**: RPC=9933, Custom RPC=40200, WS=9944
   - **Fix**: Match service configuration:
   ```javascript
   // If service uses --rpc-port 40200
   const api = await ApiPromise.create({
     provider: new WsProvider('ws://node-ip:9944')  // WS uses default 9944
   });
   ```

5. **SSL/TLS Issues with WSS**
   - **Problem**: Using `wss://` without SSL certificate
   - **Fix**: Either:
     - Use `ws://` for testing (insecure)
     - Set up Nginx with Let's Encrypt for production `wss://`

---

### Network Producing Blocks But App Can't Query Data

**Symptom**: Nodes healthy, blocks produced, but queries return errors.

**Quick Diagnosis**:
```bash
# Check if RPC methods are restricted
curl -H "Content-Type: application/json" \
  -d '{"id":1, "jsonrpc":"2.0", "method": "rpc_methods"}' \
  http://localhost:40200 | jq
```

**Common Causes & Solutions**:

1. **Connecting to Validator Instead of RPC Node**
   - **Problem**: Validators shouldn't expose RPC (security risk)
   - **Fix**: Deploy dedicated RPC node without `--validator` flag:
   ```bash
   # RPC node configuration
   --rpc-external \
   --ws-external \
   --rpc-cors all \
   --rpc-methods Safe  # Restrict dangerous methods
   # NO --validator flag
   ```

2. **RPC Methods Restricted**
   - **Problem**: Using `--rpc-methods Safe` blocks certain queries
   - **Fix**: For testing only, use `--rpc-methods Unsafe` (NOT for production)

3. **Node Not Fully Synced**
   - **Problem**: Querying recent blocks that node doesn't have yet
   - **Check**:
   ```bash
   curl -s http://localhost:40200 -H "Content-Type: application/json" \
     -d '{"id":1, "jsonrpc":"2.0", "method": "system_syncState"}' | jq
   # currentBlock should equal highestBlock
   ```
   - **Fix**: Wait for full sync

4. **Querying Wrong Chain**
   - **Problem**: App configured for different chain/network
   - **Fix**: Verify chain name matches:
   ```bash
   curl -H "Content-Type: application/json" \
     -d '{"id":1, "jsonrpc":"2.0", "method": "system_chain"}' \
     http://localhost:40200
   ```

---

### FAQ: Quick Reference

**Q: Do bootstrap nodes need validator keys?**
**A:** ❌ **NO**. Bootnodes only help with peer discovery. Only validator nodes need Aura, Grandpa, and ImOnline keys.

---

**Q: How many validators minimum for a working network?**
**A:** **3 validators** minimum. Grandpa finality requires 2/3 supermajority, which means 2 out of 3 validators (2/3 + 1 = 3).

---

**Q: Can I run validator and RPC on the same node?**
**A:** Technically yes, but **not recommended**. Security best practice: validators should not expose RPC/WS publicly. Use dedicated RPC node.

---

**Q: What's the difference between `best` and `finalized` blocks?**
**A:**
- `best`: Latest block produced (Aura)
- `finalized`: Block confirmed by 2/3+ validators (Grandpa)
- Normally `finalized` lags `best` by 1-3 blocks

---

**Q: Why can't users send transactions to my validator?**
**A:** Validators use P2P port (40100) for consensus only. Users need RPC/WebSocket (40200/9944) which should only be exposed on dedicated RPC nodes, not validators.

---

**Q: Do I need separate seed phrases for each key type?**
**A:** No. Use same seed phrase, derive different keys:
- Aura: `seed`
- Grandpa: `seed//grandpa`
- ImOnline: `seed//im_online`

---

**Q: How do I know if my validator is actively producing blocks?**
**A:** Check logs for "Starting consensus session":
```bash
journalctl -u d9-node.service -f | grep "consensus\|Prepared block"
```

---

**Q: Can I change validator keys after network launch?**
**A:** Yes, but requires session rotation:
1. Insert new keys
2. Call `session.setKeys()` extrinsic
3. Wait for session change (~1 era)

---

**Q: What happens if a validator goes offline?**
**A:**
- Network continues with remaining validators (if ≥3 still online)
- Offline validator misses block production slots
- May be slashed for downtime (check runtime configuration)

---

**Q: My node shows "Idle" - is this normal?**
**A:** Depends:
- **Bootnode/RPC node**: Yes, normal (they don't produce blocks)
- **Validator**: No, check that session keys are inserted and network has ≥3 validators

---

**Q: How long does initial sync take?**
**A:**
- **New testnet**: Minutes (small chain)
- **Mainnet**: Hours to days depending on chain size
- Check progress: `best` should approach `finalized`

---

## Security Considerations

### Seed Phrase Security

- **NEVER** share seed phrases over insecure channels
- Store seed phrases in encrypted password managers
- Keep offline backups in secure physical locations
- Use different seed phrases for testnet and mainnet

### Network Security

```bash
# Configure firewall (example with ufw)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 40100/tcp  # P2P
# Only allow RPC from specific IPs if needed
sudo ufw allow from <YOUR_IP> to any port 40200
sudo ufw enable
```

### Service Security

```bash
# Run node as non-root user (already configured in systemd)
# Limit file permissions
chmod 700 /home/ubuntu/node-data
chmod 600 /home/ubuntu/node-data/chains/*/keystore/*
```

## Next Steps

After deploying your testnet:

1. **Monitor Performance**: Watch logs and resource usage for the first 24 hours
2. **Test Functionality**: Try transactions, staking, and governance features
3. **Document Issues**: Keep notes on any problems for mainnet deployment
4. **Plan Upgrades**: Test runtime upgrade procedures on testnet first
5. **Backup Keys**: Ensure all validator keys are backed up securely

## Resources

- [D9 Node Documentation](../README.md)
- [Installation Guide](./installation.md)
- [Running a Node](./running-a-node.md)
- [GitHub Repository](https://github.com/D-Nine-Chain/d9-node)
- [Discord Community](https://discord.gg/d9chain)

## Support

For help with testnet deployment:

- [GitHub Issues](https://github.com/D-Nine-Chain/d9-node/issues)
- [Discord Community](https://discord.gg/d9chain)

---

**Important**: This is a testnet deployment guide. For production/mainnet deployment, additional security measures and monitoring are required.
