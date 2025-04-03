# Botanix Federation Node Setup Guide

This guide provides comprehensive instructions for setting up and operating a Botanix Federation node with all required microservices. The setup includes Reth Ethereum node, Bitcoin signing server, CometBFT consensus node, and Grafana Alloy monitoring.

## Table of Contents

- [System Requirements](#system-requirements)
- [Architecture Overview](#architecture-overview)
- [Security Considerations](#security-considerations)
- [Key Generation](#key-generation)
- [Key Management](#key-management)
- [Deployment Options](#deployment-options)
  - [Docker Compose Setup](#docker-compose-setup)
  - [Binary Installation](#binary-installation)
- [Configuration](#configuration)
  - [Reth Node](#reth-node-configuration)
  - [Bitcoin Signing Server](#bitcoin-signing-server-configuration)
  - [CometBFT Consensus Node](#cometbft-consensus-node-configuration)
  - [Grafana Alloy](#grafana-alloy-configuration)
- [Block Fees](#block-fees)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

## System Requirements

### Minimum Hardware Requirements

| Component | Requirement |
|-----------|-------------|
| CPU | 8+ cores, x86_64 architecture |
| RAM | 32GB minimum, 64GB recommended |
| Disk | 2TB NVMe SSD (8TB+ recommended for archive nodes) |
| Network | 100 Mbps, stable connection with low latency |

### Operating System

- Ubuntu 22.04 LTS (recommended)
- Other Linux distributions supported but not officially tested

## Architecture Overview

Our Federation node consists of interconnected microservices:

```ascii
┌─────────────────────────────────────────────────────────────┐
│                 Botanix Federation Host                     │
│                                                             │
│  ┌─────────────┐   ┌───────────────┐   ┌────────────────┐   │
│  │   Bitcoin   │   │    Reth       │   │  CometBFT      │   │
│  │   Signing   │◄──┤    Node       │◄──┤  Consensus     │   │
│  │   Server    │   │               │   │  Node          │   │
│  └─────┬───────┘   └───────┬───────┘   └────────┬───────┘   │
│        │                   │                    │           │
│        │                   │                    │           │
│        └───────────────────▼────────────────────┘           │
│                            │                                │
│                     ┌──────▼───────┐                        │
│                     │  Grafana     │                        │
│                     │  Alloy       │                        │
│                     └──────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

Each component has a specific role:

- **Reth Node**: Ethereum execution layer client (cometbft abci client)
- **Bitcoin Signing Server**: Manages the cryptographic operations for cross-chain transactions
- **CometBFT Consensus Node**: Provides Byzantine Fault Tolerant consensus
- **Grafana Alloy**: Monitoring customized for node operations

## Security Considerations

### Network Security

1. **Firewall Configuration**

   ```bash
   # Allow SSH
   ufw allow 22/tcp
   
   # Allow p2p connections for Reth
   ufw allow 30303/tcp
   ufw allow 30303/udp
   
   # Allow metrics export for Nodes
   ufw allow 7000/tcp
   ufw allow 9001/tcp
   ufw allow 26660/tcp

   # Allow CometBFT
   ufw allow 26656/tcp
   
   # Enable firewall
   ufw enable
   ```

2. **DDoS Protection**
   - Configure rate limiting on your reverse proxy or firewall
   - Consider using a cloud-based DDoS protection service

3. **Network Segregation**
   - Consider using sentry nodes for cometbft validators see[https://docs.cometbft.com/v0.37/core/validators]
   - Implement a bastion host for validator node access

### System Hardening

1. **Minimize Attack Surface**

   ```bash
   # Remove unnecessary packages
   apt purge telnetd xinetd nis yp-tools tftpd atftpd tftpd-hpa telnet
   
   # Disable unused services
   systemctl disable avahi-daemon cups nfs-kernel-server rpcbind
   ```

2. **Secure SSH**

   ```bash
   # Edit SSH config
   nano /etc/ssh/sshd_config
   
   # Set these values
   PermitRootLogin no
   PasswordAuthentication no
   X11Forwarding no
   MaxAuthTries 3
   
   # Restart SSH
   systemctl restart sshd
   ```

3. **Regular Updates**

   ```bash
   # Set up unattended security updates
   apt install unattended-upgrades
   dpkg-reconfigure -plow unattended-upgrades
   ```

## Key Generation

### Reth Keys

```bash
git clone https://github.com/botanix-labs/init-keys.git
cd init-keys

# generate the keys and store in ./output dir
cargo run
```

### Cometbft Keys

```bash
cometbft init -k "secp256k1" --home /path/to/cometbft
```

## Key Management

### Cold Storage for Validator Keys

1. **Generate Keys Offline**
    in view **

2. **Hardware Security Module (HSM) Integration**
   - YubiHSM 2 or Ledger devices recommended

## Deployment Options

### Docker Compose Setup

#### Prerequisites

1. Install Docker and Docker Compose

   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sh get-docker.sh
   ```

#### Directory Structure

Create the following directory structure:

```text
Botanix-Fed/
├── docker-compose.yml
├── .env
├── config/
│   ├── reth/
│   ├── signing-server/
│   ├── cometbft/
│   └── grafana-alloy/  
└── data/
    ├── reth/
    ├── signing-server/
    ├── cometbft/
    └── grafana-alloy/
```

#### Docker Compose File

Create a `docker-compose.yml` file:

```yaml
version: '3.7'

services:
  bitcoin-signing-server:
    env_file:
      - .bitcoin.env
    container_name: bitcoin-signing-server
    image: us-central1-docker.pkg.dev/botanix-391913/botanix-testnet-btc-server-v1/botanix-btc-server-v1
    hostname: bitcoin-signing-server
    command:
      - --btc-network=mainnet
      - --identifier=FROST_IDENTIFIER
      - --address=0.0.0.0:8080
      - --db=/bitcoin-server/data/db
      - --min-signers=11
      - --max-signers=15
      - --toml=/bitcoin-server/config/config.toml
      - --fee-rate-diff-percentage=30
      - --bitcoind-url=${BITCOIND_HOST}
      - --bitcoind-user=${BITCOIND_USER}
      - --bitcoind-pass=${BITCOIND_PASS}
      - --btc-signing-server-jwt-secret=/bitcoin-server/config/jwt.hex
      - --fall-back-fee-rate-sat-per-vbyte=5
      - --metrics-port=7000
    ports:
      - 8080:8080
      - 7000:7000
    volumes:
      - ./config/signing-server:/bitcoin-server/config:rw
      - ./data/signing-server:/bitcoin-server/data:rw
    restart: unless-stopped
    networks:
      - federation_net

  reth-poa-node:
    env_file:
      - .bitcoin.env
    container_name: reth-poa-node
    image:  us-central1-docker.pkg.dev/botanix-391913/botanix-testnet-node-v1/botanix-poa-node
    command:
      - poa
      - --federation-config-path=/reth/botanix_testnet/config/chain.toml
      - --datadir=/reth/botanix_testnet/data
      - --http
      - --http.addr=0.0.0.0
      - --http.port=8545
      - --http.api=debug,eth,net,trace,txpool,web3,rpc
      - --http.corsdomain=*
      - -vvv
      - --btc-signing-server-jwt-secret=/reth/botanix_testnet/config/jwt.hex
      - --authrpc.addr=127.0.0.1
      - --authrpc.port=8551
      - --btc-server=bitcoin-signing-server:8080
      - --bitcoind.url=${BITCOIND_HOST}
      - --bitcoind.username=${BITCOIND_USER}
      - --bitcoind.password=${BITCOIND_PASS}
      - --frost.min_signers=11
      - --frost.max_signers=15
      - --p2p-secret-key=/reth/botanix_testnet/config/discovery-secret
      - --port=30303
      - --btc-network=mainnet
      - --metrics=0.0.0.0:9001
      - --federation-mode
      - --abci-port=26658
      - --ipcdisable
      - --abci-host=0.0.0.0
      - --cometbft-rpc-port=26657
      - --cometbft-rpc-host=cometbft-consensus-node
    ports:
      - 8545:8545
      - 8546:8546
      - 9001:9001
      - 30303:30303
      - 26658:26658
    depends_on:
      - bitcoin-signing-server
    volumes:
      - ./config/reth:/reth/botanix_testnet/config:rw
      - ./data/reth:/reth/botanix_testnet/data:rw
    restart: unless-stopped
    networks:
      - federation_net


  cometbft-consensus-node:
    container_name: cometbft-consensus-node
    image: us-central1-docker.pkg.dev/botanix-391913/botanix-betanet-cometbft/botanix-betanet-cometbft:v1 
    ports:
      - 26656:26656
      - 26657:26657
      - 26660:26660
    volumes:
      - ./config/cometbft:/cometbft/config:rw
      - ./data/cometbft:/cometbft/data:rw
    restart: unless-stopped
    user: root
    command: >
      node
      --home=/cometbft
      --proxy_app="reth-poa-node:26658"
      --moniker="INPUT_YOUR_MONIKER"
      --p2p.persistent_peers='PERSISTENT_PEERS_LIST'
      --p2p.laddr=${P2P_LADDR:-tcp://0.0.0.0:26656}
      --rpc.laddr=${RPC_LADDR:-tcp://0.0.0.0:26657}
    environment:
      - ALLOW_DUPLICATE_IP=TRUE
      - LOG_LEVEL=DEBUG
      - CHAIN_ID=3637
    networks:
      - federation_net


  grafana-alloy:
    container_name: grafana-alloy
    image: grafana/alloy:latest
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config.alloy
    ports:
      - 12345:12345
    volumes:
      - /etc/hostname:/etc/hostname:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data/alloy:/var/lib/alloy/data
      - ./config/alloy:/etc/alloy/  
    restart: on-failure
    networks:
      - federation_net


  nginx-proxy:
    container_name: nginx-proxy
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./config/proxy:/etc/nginx/conf.d/nginx.conf
    restart: unless-stopped
    networks:
      - federation_net


  cadvisor:
    container_name: cadvisor
    image: gcr.io/cadvisor/cadvisor:v0.47.0
    ports:
      - 8000:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    restart: unless-stopped
    privileged: true
    networks:
      - federation_net

networks:
  federation_net:
    name: federation_net
    driver: bridge
```

#### Starting the Services

```bash
cd Botanix-Fed
docker-compose up --env-file .bitcoin.env -d
```

Monitor the logs:

```bash
docker-compose logs -f CONTAINER_NAME
```

### Binary Installation

If Docker is not suitable for your environment, you can compile and run the components as native binaries.

#### Building from Source

##### Reth Node

```bash

# Download Reth Binary from gcloud bucket
wget https://storage.googleapis.com/botanix-artifact-registry/releases/reth/v1.0.5/reth-x86_64-unknown-linux-gnu.tar.gz

# Download the binary checksum
wget https://storage.googleapis.com/botanix-artifact-registry/releases/reth/v1.0.5/reth-x86_64-unknown-linux-gnu.tar.gz.sha256sum

# Verify binary checksum 
sha256sum -c reth-x86_64-unknown-linux-gnu.tar.gz.sha256sum

# Extract the tarball
tar -xzf reth-x86_64-unknown-linux-gnu.tar.gz

# Make the binary executable
chmod +x reth

# Optionally move to a system path
sudo mv reth /usr/local/bin/

```

##### Bitcoin Signing Server

```bash
# Download BTC Signing Server from gcloud bucket
wget https://storage.googleapis.com/botanix-artifact-registry/releases/btc-server/v1.0.5/btc-server-x86_64-unknown-linux-gnu.tar.gz

# Download the binary checksum
wget https://storage.googleapis.com/botanix-artifact-registry/releases/btc-server/v1.0.5/btc-server-x86_64-unknown-linux-gnu.tar.gz.sha256sum

# Verify binary checksum
sha256sum -c btc-server-x86_64-unknown-linux-gnu.tar.gz.sha256sum

# Extract the tarball
tar -xzf btc-server-x86_64-unknown-linux-gnu.tar.gz

# Make the binary executable
chmod +x btc-server

# Optionally move to a system path
sudo mv btc-server /usr/local/bin/

```

##### CometBFT Consensus Node

```bash
git clone https://github.com/cometbft/cometbft.git
cd cometbft
make install

# Verify installation
cometbft version
```

## Configuration

### Reth Node Configuration

Create a chain.toml config file at `config/reth/chain.toml`:

This would be communicated to federation members

```bash
reth poa \
    --federation-config-path=PATH_TO_FEDERATION_CONFIG \
    --datadir=/reth/botanix_testnet \
    --http \
    --http.addr=0.0.0.0 \
    --http.port=8545 \
    --http.api=debug,eth,net,trace,txpool,web3,rpc \
    --http.corsdomain=* \
    -vvv \
    --btc-signing-server-jwt-secret=/reth/botanix_testnet/jwt.hex \
    --authrpc.addr=127.0.0.1 \
    --authrpc.port=8551 \
    --btc-server=localhost:8080 \
    --bitcoind.url=BITCOIND_HOST \
    --bitcoind.username=BITCOIND_USER \
    --bitcoind.password=${BITCOIND_PASS} \
    --frost.min_signers=11 \
    --frost.max_signers=15 \
    --p2p-secret-key=/reth/botanix_testnet/discovery-secret \
    --port=30303 \
    --btc-network=ma \
    --metrics=0.0.0.0:9001 \
    --federation-mode \
    --abci-port=26658 \
    --ipcdisable \
    --abci-host=0.0.0.0
```

### Bitcoin Signing Server Configuration

Create a config file at `config/signing-server/config.toml`:

```toml
[grpc]
enable-reflection = true
accept-compressed = "Gzip"
send-compressed = "Gzip"
max-decoding-message-size = 52428800
max-encoding-message-size = 52428800
max-channel-size = 128
timeout = 60
max-frame-size = 16777215
concurrency-limit-per-connection = 32
max-concurrent-streams = 1024
tcp-nodelay = true
draw-lookahead-period-count = 10
http2-keepalive-interval = 60
http2-keepalive-timeout = 20
```

### CometBFT Consensus Node Configuration

Key files to configure in `config/cometbft/config.toml`:

1. **config.toml**:

   ```toml
   # Main config
   proxy_app = "tcp://0.0.0.0:26658"
   moniker = "NODE_MONIKER"
   
   [p2p]
   laddr = "tcp://0.0.0.0:26656"
   persistent_peers = "PERSISTENT_PEERS"
   
   [consensus]
   timeout_propose = "3s"
   timeout_prevote = "1s"
   timeout_precommit = "1s"
   timeout_commit = "5s"
   ```

2. **genesis.json**:
    This would be communicated to federation members

   ```json
        {
            "app_hash": "",
            "chain_id": "3637",
            "consensus_params": {
                "block": {
                    "max_bytes": "4194304",
                    "max_gas": "-1"
                },
                "evidence": {
                    "max_age_duration": "172800000000000",
                    "max_age_num_blocks": "100000",
                    "max_bytes": "1048576"
                },
                "feature": {
                    "pbts_enable_height": "1",
                    "vote_extensions_enable_height": "0"
                },
                "synchrony": {
                    "message_delay": "15000000000",
                    "precision": "505000000"
                },
                "validator": {
                "pub_key_types": [
                    "secp256k1"
                ]
                },
                "version": {
                    "app": "0"
                }
            },
            "genesis_time": "",
            "initial_height": "0",
            "validators": []
        }
   ```

### Grafana Alloy Configuration

Set up Grafana with preconfigured dashboards:

**Alloy Provider Config** (`config/grafana-alloy/config.alloy`):

   ```alloy
        discovery.docker "linux" {
            host = "unix:///var/run/docker.sock"
        }

        discovery.relabel "docker" {
            targets = discovery.docker.linux.targets

            rule {
                source_labels = ["__meta_docker_container_name"]
                target_label  = "service_name"
                action        = "replace"
            }

            rule {
                source_labels = ["__meta_docker_container_name"]
                target_label  = "federation"
                action        = "replace"
            }
        }

        loki.source.docker "default" {
            host       = "unix:///var/run/docker.sock"
            targets    = discovery.relabel.docker.output
            labels     = {env="mainnet", node="botanix-federation-member"}
            forward_to = [loki.process.filter_logs.receiver]
        }

        loki.process "filter_logs" {
            stage.docker {}
            forward_to = [loki.write.default.receiver]
        }

        loki.write "default" {
            endpoint {
                url = "REPLACE_WITH_ACTUAL_URL"
                # Replace the above placeholder with the actual URL provided by the Botanix Federation team.
                tenant_id = "botanix-federation"
            }
        }
   ```

### Allowed Ports (Recommended)

#### Reth

   | Port       | Functionality     | Exposure        |
   |------------|-------------------|-----------------|
   | 30303      | P2P communication | Public          |
   | 8545       | RPC Port          | Internal        |
   | 9001       | Metrics Port      | Public          |
   | 26658      | Abci client port  | Internal        |
   | 30304      | P2P communication | Public (unused) |
   | 8456       | WS Port           | Internal        |
   | 8551       | Engine API port   | Public (unused) |

#### BTC Signing Server

   | Port       | Functionality        | Exposure     |
   |------------|----------------------|--------------|
   | 8080       | Signing Service port | Internal     |
   | 7000       | Metrics Port         | Public       |
  
#### Cometbft

   | Port       | Functionality     | Exposure        |
   |------------|-------------------|-----------------|
   | 26656      | P2P communication | Public          |
   | 26657      | RPC Port          | Internal        |
   | 26660      | Metrics Port      | Public          |

## Block Fees

The evm address for the block fees should be generated from the cometbft validator secret key. see[https://github.com/botanix-labs/init-keys]
This should be stored in the data_dir.

## Maintenance

### Backup Procedures

1. **Database Backups**

   ```bash
   # Backup Reth data
   tar czf reth-backup-$(date +%F).tar.gz data/reth
   
    # Backup Signing data
    tar czf btc-signing-$(date +%F).tar.gz data/signing-server

   # Backup CometBFT data
   tar czf cometbft-backup-$(date +%F).tar.gz data/cometbft
   ```

2. **Key Backups**
   - Store encrypted backups in secure offline storage

### Upgrade Procedures

1. **Version Compatibility Matrix**

   | Component      |Compt Versions   |
   |----------------|-----------------|
   | Reth           | v1.0.0          |
   | Bitcoin Signer | v1.0.0          |
   | CometBFT       | v1.x            |

2. **Rollback Plan**
   - Maintain previous version containers/binaries
   - Keep database snapshots before major upgrades
   - Document the exact internal rollback procedure for each service

## Troubleshooting

### Common Issues

1. **Reth Node Not Syncing**
   - Check network connectivity to peers
   - Check logs for error messages

2. **Bitcoin Signing Server Errors**
   - Check connectivity to bitcoind instance
   - Check key permissions and authentication

3. **CometBFT Consensus Failures**
   - Check network connectivity between validators
   - Verify all validators are running the same software version
   - Look for timeout errors in logs
   - Verify connectivity to abci client (Reth node)

### Log Analysis

```bash
# Collect logs from all services
docker-compose logs > all-logs.txt

# Search for specific errors
grep -i "error" all-logs.txt

# Monitor logs in real-time
docker-compose logs -f --tail=100
```

## FAQ

**Q: How much disk space will the Reth node require over time?**
A: A Full node will grow approximately 25-35GB per month. Plan for at least 2TB of storage for a full year of operation.

**Q: What is the recommended key rotation procedure?**
A: For Cometbft keys, generate new keys offline, transfer to HSM, then update the validator configuration see[https://docs.cometbft.com/v0.37/core/validators].

**Q: What monitoring alerts should I set up?**
A: At minimum: resource utilization (CPU, RAM, disk), dkg round status, peer list, and network connectivity.
