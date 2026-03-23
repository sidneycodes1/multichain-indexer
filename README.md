# Multichain Transaction Indexer

## Overview

This is a fork of the [fystack multichain-indexer](https://github.com/fystack/multichain-indexer), an open-source tool that monitors blockchain transactions across 12+ networks in real time.

The core problem it solves is straightforward: if your business manages hundreds or thousands of wallet addresses across multiple blockchains, you need a reliable way to know the moment funds arrive at any of those addresses. Most teams either build this from scratch — which takes months — or pay for SaaS tools like Alchemy Webhooks or Moralis, which get expensive fast.

This indexer is self-hosted, production-ready, and handles the entire pipeline: watching blocks, matching destination addresses against your system, and firing events to your backend when a match is found.

I maintain this fork and offer it as a deployment and management service for crypto businesses and Web3 startups that need this infrastructure without the overhead of building or maintaining it themselves.

---

## Features

- Real-time transaction monitoring across 12+ blockchains from a single binary
- Bloom filter address matching — sub-millisecond lookups against millions of addresses
- Four cooperating workers handling live blocks, historical backfill, failed retries, and manual rescans
- NATS JetStream event streaming for instant downstream processing
- Automatic RPC failover with rate limiting and exponential backoff
- Full restart safety — all progress is persisted, so it resumes exactly where it left off
- Support for both native transfers and token transfers (ERC-20, SPL, TRC-20, etc.)

---

## Supported Chains

- EVM chains: Ethereum, BNB Chain, Polygon, Arbitrum, Optimism
- TRON
- Bitcoin
- Solana
- Aptos
- Sui
- Cosmos (Hub, Osmosis, Celestia)
- TON

---

## Tech Stack

- **Language:** Go
- **Message broker:** NATS JetStream
- **Key-value store:** BadgerDB, Consul, or in-memory
- **Address filtering:** Bloom filter (Redis-backed or in-memory)
- **Database:** PostgreSQL (wallet address repository)
- **Cache:** Redis (manual worker ranges and bloom filter)
- **Infrastructure:** Docker, docker-compose

---

## Installation

### Prerequisites

Make sure you have the following installed:

- Go 1.21+
- Docker and docker-compose
- Git

### Steps

**1. Clone the repository**

```bash
git clone https://github.com/sidneycodes1/multichain-indexer.git
cd multichain-indexer
```

**2. Install dependencies**

```bash
go mod download
```

**3. Set up configuration**

```bash
cp configs/config.example.yaml configs/config.yaml
```

Open `configs/config.yaml` and fill in your RPC URLs, API keys, and chain settings.

**4. Start required services**

```bash
docker-compose up -d
```

This starts NATS, PostgreSQL, Redis, and Consul locally.

**5. Build the binary**

```bash
go build -o indexer cmd/indexer/main.go
```

**6. Run the indexer**

```bash
# Real-time indexing
./indexer index --chains=ethereum_mainnet,solana_mainnet

# Real-time + historical backfill
./indexer index --chains=ethereum_mainnet,solana_mainnet --catchup

# Include manual worker for missing blocks
./indexer index --chains=ethereum_mainnet,solana_mainnet --manual
```

---

## Usage

### Running specific chains

The chain names you pass to `--chains` must match the keys defined in your `config.yaml` file. For example:

```bash
./indexer index --chains=ethereum_mainnet,tron_mainnet,solana_mainnet
```

### Debug mode

```bash
./indexer index --chains=ethereum_mainnet --debug
```

### Setting up NATS event consumption

```bash
# Create the stream
nats stream add transfer --subjects="transfer.event.*" --storage=file --retention=workqueue

# Create a consumer
nats consumer add transfer my-consumer --filter="transfer.event.dispatch" --deliver=all --ack=explicit

# Start consuming
nats consumer sub transfer my-consumer
```

### Loading wallet addresses into the bloom filter

```bash
./wallet-kv-load run --config configs/config.yaml --batch 10000
```

---

## Project Structure

```
multichain-indexer/
├── cmd/
│   └── indexer/          # Entry point for the main binary
├── configs/              # Configuration files and examples
├── internal/             # Core application logic
│   ├── worker/           # RegularWorker, CatchupWorker, ManualWorker, RescannerWorker
│   ├── indexer/          # Chain-specific indexer implementations
│   └── bloom/            # Bloom filter logic
├── pkg/                  # Shared packages and utilities
├── sql/                  # Database migration files
├── .github/workflows/    # CI/CD configuration
└── docker-compose.yml    # Local development services
```

---

## Worker Architecture

The indexer runs four types of workers that cooperate to ensure no transactions are missed:

**RegularWorker**
Processes the latest blocks as they are produced. Handles chain reorgs for EVM chains. Sends failed blocks to the retry queue.

**CatchupWorker**
Backfills historical blocks in defined ranges. Tracks its own progress per range and cleans up completed ranges automatically.

**ManualWorker**
Handles explicitly defined missing block ranges stored in a Redis sorted set. Splits large ranges into smaller chunks and uses Redis locks to prevent duplicate processing across concurrent instances.

**RescannerWorker**
Retries blocks that previously failed. Removes blocks after reaching the maximum retry limit.

---

## Event Format

When a matching transaction is detected, the indexer publishes a JSON event to NATS on the `transfer.event.dispatch` subject.

Example event for an ERC-20 token transfer:

```json
{
  "txHash": "0x1b2c3d4e...",
  "networkId": "ethereum_mainnet",
  "blockNumber": 21500200,
  "fromAddress": "0xA1b2C3...",
  "toAddress": "0xF1e2D3...",
  "assetAddress": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
  "amount": "50000000",
  "type": "token_transfer",
  "txFee": "0.00042",
  "timestamp": 1704067800
}
```

**Important:** The bloom filter can produce false positives. Always verify that the `toAddress` field exists in your own database before crediting any account or triggering fulfillment logic.

---

## Deployment Service

If you need this running in production but do not want to manage the infrastructure yourself, I offer a deployment and management service.

**Pricing**

| Tier | Chains | Support | Setup Fee | Monthly |
|---|---|---|---|---|
| Starter | 1 to 3 chains | 1 month | $300 | $100/month |
| Standard | 4 to 8 chains | 3 months | $600 | $200/month |
| Premium | 12+ chains | Priority support + SLA | $1,000 | $400/month |

I am currently offering one free deployment in exchange for a testimonial. If you are an early-stage startup, reach out.

Contact: **sydneybawa@gmail.com** or **[@sidneycodes1](https://twitter.com/sidneycodes1)** on X

---

## Contributing

Contributions are welcome. Please follow these steps:

1. Fork the repository
2. Create a new branch: `git checkout -b feat/your-feature-name`
3. Make your changes and write tests where applicable
4. Commit with a clear message: `git commit -m 'feat: describe your change'`
5. Push your branch: `git push origin feat/your-feature-name`
6. Open a pull request against the `main` branch

Please keep pull requests focused. One feature or fix per PR makes review much easier.

---

## Roadmap

- Add support for additional EVM Layer 2 networks
- Improve internal metrics and observability (Prometheus/Grafana)
- Build a lightweight web dashboard for monitoring indexer health
- Add webhook delivery directly from the indexer without requiring NATS consumer setup
- Improve documentation with more configuration examples

---

## License

This project is licensed under the MIT License. See the [LICENSE](./LICENSE) file for details.

---

## Author

**Sidney**
Full-stack and Web3 developer based in Nigeria.
GitHub: [sidneycodes1](https://github.com/sidneycodes1)
X: [@sidneycodes1](https://twitter.com/sidneycodes1)
Email: sydneybawa@gmail.com

Original project by [fystack.io](https://fystack.io).
