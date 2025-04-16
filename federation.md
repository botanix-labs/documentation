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
# Note: Cometbft keys should be initialised with the secp256k1 algorithm
cometbft init -k "secp256k1" --home /path/to/cometbft
```

#### Generate enode_id for p2p Communication

```bash
# This generates the enode_id needed for p2p communication
cometbft show-node-id --home <PATH_TO_DIR>
```

The resulting node_id is used in the format: `node_id@node_ip_addr:node_p2p_port`
Example: `957218cfa7ccea8585a752a77cb0acc478c5cc4b@34.635.278.90:26656`

## Key Management

### Key Transmission Format

#### Reth

- federation-public-key (obtained from init-keys)

#### Cometbft

```json
{
  "address": "",
  "name": "",
  "power": "10",
  "pub_key": {
    "type": "tendermint/PubKeySecp256k1",
    "value": ""
  }
}

node_id: `node_id@node_ip_addr:node_p2p_port`
```

Where:

- `address`: the address value in the `priv_validator_key.json` file
- `name`: the name of your node
- `value`: the public key value in the `priv_validator_key.json` file

### Storage for Validator Keys

1. **Generate Keys**
   - Generate Cometbft keys first and move them to the init-keys directory

2. **Hardware Security Module (HSM) Integration**
   - TMKMS recommended for cometbft validators

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
    image: us-central1-docker.pkg.dev/botanix-391913/botanix-mainnet-btc-server/botanix-btc-server
    hostname: bitcoin-signing-server
    command:
      - --btc-network=bitcoin
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
    image:  us-central1-docker.pkg.dev/botanix-391913/botanix-mainnet-node/botanix-reth
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
      - --btc-network=bitcoin
      - --metrics=0.0.0.0:9001
      - --federation-mode
      - --abci-port=26658
      - --ipcdisable
      - --abci-host=0.0.0.0
      - --cometbft-rpc-port=26657
      - --cometbft-rpc-host=cometbft-consensus-node
      - --sync.enable_state_sync
      - --sync.enable_historical_sync
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
      - CHAIN_ID=3737
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

#### Binary Deployment Mode

If Docker is not suitable for your environment, you can run the components as native binaries.
We have provided these binaries and their checksum below.

##### Reth Node

```bash

# Download Reth Binary from gcloud bucket
wget https://storage.googleapis.com/botanix-artifact-registry/releases/reth/v.1.0.7-main/reth-x86_64-unknown-linux-gnu.tar.gz

# Download the binary checksum
wget https://storage.googleapis.com/botanix-artifact-registry/releases/reth/v.1.0.7-main/reth-x86_64-unknown-linux-gnu.tar.gz.sha256sum

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
wget https://storage.googleapis.com/botanix-artifact-registry/releases/btc-server/v.1.0.7-main/btc-server-x86_64-unknown-linux-gnu.tar.gz

# Download the binary checksum
wget https://storage.googleapis.com/botanix-artifact-registry/releases/btc-server/v.1.0.7-main/btc-server-x86_64-unknown-linux-gnu.tar.gz.sha256sum

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
    --bitcoind.url=${BITCOIND_HOST} \
    --bitcoind.username=${BITCOIND_USER} \
    --bitcoind.password=${BITCOIND_PASS} \
    --frost.min_signers=11 \
    --frost.max_signers=15 \
    --p2p-secret-key=/reth/botanix_testnet/discovery-secret \
    --port=30303 \
    --btc-network=bitcoin \
    --metrics=0.0.0.0:9001 \
    --federation-mode \
    --abci-port=26658 \
    --ipcdisable \
    --abci-host=0.0.0.0 \
    --cometbft-rpc-port=26657 \
    --cometbft-rpc-host=COMETBFT_HOST \
    --sync.enable_state_sync \
    --sync.enable_historical_sync
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

#### Bitcoin Signing Server CLI Configuration

When running the btc-server binary directly, use the following command-line arguments:

```bash
btc-server \
  --btc-network=bitcoin \
  --identifier=FROST_IDENTIFIER \
  --address=0.0.0.0:8080 \
  --db=/path/to/data/db \
  --min-signers=11 \
  --max-signers=15 \
  --toml=/path/to/config/config.toml \
  --fee-rate-diff-percentage=30 \
  --bitcoind-url=YOUR_BITCOIND_HOST \
  --bitcoind-user=YOUR_BITCOIND_USER \
  --bitcoind-pass=YOUR_BITCOIND_PASS \
  --btc-signing-server-jwt-secret=/path/to/config/jwt.hex \
  --fall-back-fee-rate-sat-per-vbyte=5 \
  --metrics-port=7000
```

Replace `/path/to/` with your actual installation paths, and we would supply the appropriate values for `FROST_IDENTIFIER`.

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
            "chain_id": "3737",
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

        # Read hostname from file for machine identification
        local.file "hostname" {
            path = "/etc/hostname"
        }

        discovery.relabel "docker" {
            targets = discovery.docker.linux.targets

            # Add machine hostname to all metrics
            rule {
                replacement   = local.file.hostname.content
                target_label  = "machine_name"
            }

            # Extract and label specific services with proper names
            rule {
                source_labels = ["__meta_docker_container_name"]
                regex         = "bitcoin-signing-server"
                replacement   = "btc-server"
                target_label  = "service"
            }

            rule {
                source_labels = ["__meta_docker_container_name"]
                regex         = "reth-poa-node"
                replacement   = "reth-node"
                target_label  = "service"
            }

            rule {
                source_labels = ["__meta_docker_container_name"]
                regex         = "cometbft-consensus-node"
                replacement   = "consensus-node"
                target_label  = "service"
            }

            # Federation label for all components
            rule {
                replacement   = "botanix-federation"
                target_label  = "federation"
            }
        }

        # Configure log collection for the three main services
        loki.source.docker "default" {
            host       = "unix:///var/run/docker.sock"
            targets    = discovery.relabel.docker.output
            labels     = {
                env = "mainnet", 
                node_type = "federation-member", 
                machine = local.file.hostname.content
            }
            forward_to = [loki.process.filter_logs.receiver]
        }

        loki.process "filter_logs" {
            stage.docker {}
            
            # Extract log levels for better filtering
            stage.regex {
                expression = "(?i)\\b(error|warn|info|debug)\\b"
                source     = "message"
                labels     = ["level"]
            }
            
            forward_to = [loki.write.default.receiver]
        }

        loki.write "default" {
            endpoint {
                url = "REPLACE_WITH_ACTUAL_URL"
                # Authentication options - choose one:
                
                # Option 1: Basic Auth
                basic_auth {
                    username = "REPLACE_WITH_USERNAME"
                    password = "REPLACE_WITH_PASSWORD"
                }
                
                # Option 2: Bearer Token (commented out - uncomment to use)
                # bearer_token = "REPLACE_WITH_TOKEN"
                
                tenant_id = "botanix-federation"
            }
        }

        # Configure metrics scraping for the three main services
        prometheus.scrape "federation_metrics" {
            targets = discovery.relabel.docker.output
            forward_to = [prometheus.remote_write.default.receiver]
            job_name = "botanix_federation_node"
        }

        prometheus.remote_write "default" {
            endpoint {
                url = "REPLACE_WITH_METRICS_URL"
                
                # Authentication options - choose one:
                
                # Option 1: Basic Auth
                basic_auth {
                    username = "REPLACE_WITH_USERNAME"
                    password = "REPLACE_WITH_PASSWORD"
                }
                
                # Option 2: Bearer Token (commented out - uncomment to use)
                # bearer_token = "REPLACE_WITH_TOKEN"
                
                # Option 3: Authorization Header (commented out - uncomment to use)
                # headers = {
                #     "Authorization" = "REPLACE_WITH_AUTH_HEADER"
                # }
            }
        }
   ```

### Allowed Ports (Recommended)

#### Reth Node Ports

| Port   | Protocol | Functionality      | Exposure        | Required | Additional Info                                           |
|--------|----------|-------------------|-----------------|----------|----------------------------------------------------------|
| 30303  | TCP/UDP  | P2P communication | Public          | Yes      | To enable connectivity to peers                           |
| 8545   | TCP      | JSON-RPC API      | Internal        | Yes      | Recommend to use internally within organization           |
| 9001   | TCP      | Metrics Port      | Public          | Optional | Allow for obtaining metrics for public dashboard          |
| 26658  | TCP      | ABCI client port  | Internal        | Yes      | Internal to microservices                                 |
| 30304  | TCP/UDP  | P2P communication | Public          | No       | Backup/alternate P2P port (unused)                        |
| 8546   | TCP      | WebSocket Port    | Internal        | Optional | WebSocket connections for subscriptions                   |
| 8551   | TCP      | Engine API port   | Internal        | No       | Not used in federation mode                               |

#### Bitcoin Signing Server Ports

| Port   | Protocol | Functionality        | Exposure     | Required | Additional Info                                 |
|--------|----------|----------------------|--------------|----------|------------------------------------------------|
| 8080   | TCP      | Signing Service API  | Internal     | Yes      | Must be accessible from Reth node               |
| 7000   | TCP      | Metrics Port         | Public       | Optional | Expose metrics for monitoring dashboard         |
  
#### CometBFT Consensus Node Ports

| Port   | Protocol | Functionality      | Exposure     | Required | Additional Info                                 |
|--------|----------|-------------------|--------------|----------|------------------------------------------------|
| 26656  | TCP      | P2P communication | Public       | Yes      | Required for validator communication            |
| 26657  | TCP      | RPC API           | Internal     | Yes      | RPC interface for queries/transactions          |
| 26660  | TCP      | Metrics Port      | Public       | Optional | Expose metrics for monitoring dashboard         |

> **Note:** Public metrics endpoints should ideally be protected by authentication or restricted to specific IPs.

## Block Fees

The evm address for the block fees should be generated from the cometbft validator secret key. see[https://github.com/botanix-labs/init-keys]
This should be stored in the reth data_dir.

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

#### Pre-Upgrade Checklist

1. **Review Release Notes**
   - Review all changes in the upcoming release
   - Note any breaking changes or new dependencies
   - Check for configuration format changes

2. **Backup Critical Data**
   - Create snapshots of all databases before upgrading
   - Backup all configuration files
   - Ensure backups are verified and accessible

3. **Version Compatibility Matrix**

   | Component      | Current Version | Compatible Versions | Notes                          |
   |----------------|-----------------|---------------------|--------------------------------|
   | Reth           | v1.0.7-main     | ≥v1.0.x             | Minor versions are compatible  |
   | Bitcoin Signer | v1.0.7-main     | ≥v1.0.x             | Must match Reth version        |
   | CometBFT       | v1.0.0          | ≥v1.0.x             | Not compatible with v0.37+     |

4. **Recommended Upgrade Path**

```bash
v1.0.0 → v1.0.3 → v1.0.7  # Do not skip multiple major versions
```

#### Upgrade Execution

1. **Docker-based Upgrade**

   ```bash
   # Pull new images
   docker pull us-central1-docker.pkg.dev/botanix-391913/botanix-mainnet-node/botanix-reth:latest
   docker pull us-central1-docker.pkg.dev/botanix-391913/botanix-mainnet-btc-server/botanix-btc-server:latest
   ```

2. **Binary-based Upgrade**

   ```bash
   # Download new binaries
   wget https://storage.googleapis.com/botanix-artifact-registry/releases/reth/v.1.0.7-main/reth-x86_64-unknown-linux-gnu.tar.gz
   wget https://storage.googleapis.com/botanix-artifact-registry/releases/btc-server/v.1.0.7-main/btc-server-x86_64-unknown-linux-gnu.tar.gz
   
   # Verify checksums
   sha256sum -c reth-x86_64-unknown-linux-gnu.tar.gz.sha256sum
   sha256sum -c btc-server-x86_64-unknown-linux-gnu.tar.gz.sha256sum
   
   # Stop running services
   systemctl stop reth bitcoin-signing-server cometbft
   
   # Backup binaries
   cp /usr/local/bin/reth /usr/local/bin/reth.bak
   cp /usr/local/bin/btc-server /usr/local/bin/btc-server.bak
   
   # Extract and install new binaries
   tar -xzf reth-x86_64-unknown-linux-gnu.tar.gz
   tar -xzf btc-server-x86_64-unknown-linux-gnu.tar.gz
   chmod +x reth btc-server
   mv reth btc-server /usr/local/bin/
   
   # Restart services
   systemctl start reth bitcoin-signing-server cometbft
   ```

3. **Configuration Updates**
   - Review and apply any configuration changes required by the new version
   - Update any environment variables or command line flags

#### Post-Upgrade Verification

1. **Verify Services**

   ```bash
   # Check service status
   docker-compose ps  # For Docker setup
   systemctl status reth bitcoin-signing-server cometbft  # For binary setup
   
   # Verify logs for errors
   docker-compose logs -f  # For Docker setup
   journalctl -u reth -u bitcoin-signing-server -u cometbft -f  # For binary setup
   ```

2. **Verify Connectivity**

   ```bash
   # Test RPC endpoints
   curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' http://localhost:8545
   ```

3. **Monitor Performance**
   - Check resource usage (CPU, memory, disk)
   - Verify transaction processing
   - Validate block production and consensus participation

#### Rollback Procedure

1. **Docker Rollback**

   ```bash
   # Stop services
   docker-compose down
   
   # Update docker-compose.yml with previous image tags
   # Then restart services
   docker-compose up -d
   ```

2. **Binary Rollback**

   ```bash
   # Stop services
   systemctl stop reth bitcoin-signing-server cometbft
   
   # Restore backup binaries
   mv /usr/local/bin/reth.bak /usr/local/bin/reth
   mv /usr/local/bin/btc-server.bak /usr/local/bin/btc-server
   
   # Restart services
   systemctl start reth bitcoin-signing-server cometbft
   ```

3. **Database Rollback**

   ```bash
   # Stop services
   docker-compose down  # For Docker setup
   systemctl stop reth bitcoin-signing-server cometbft  # For binary setup
   
   # Restore database backups
   rm -rf data/reth data/signing-server data/cometbft
   tar xzf reth-backup-YYYY-MM-DD.tar.gz
   tar xzf btc-signing-YYYY-MM-DD.tar.gz
   tar xzf cometbft-backup-YYYY-MM-DD.tar.gz
   
   # Restart services
   docker-compose up -d  # For Docker setup
   systemctl start reth bitcoin-signing-server cometbft  # For binary setup
   ```

> **Important:** Always test upgrades in a staging environment before applying to production nodes.

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

### General Questions

**Q: How much disk space will the Reth node require over time?**
A: A Full node will grow approximately 25-35GB per month. Plan for at least 2TB of storage for a full year of operation.

**Q: What are the bandwidth requirements for running a Federation node?**
A: You should provision at least 100 Mbps of bandwidth with unlimited data transfer.

**Q: Can I run a Federation node on a virtual machine?**
A: Yes, but dedicated bare metal is also an option for production validators. If using a VM, ensure dedicated CPU cores, guaranteed RAM, and low-latency SSD storage.

**Q: How do I know if my node is properly connected to the network?**
A: Check peer connections using: `curl -s http://localhost:26657/net_info | jq '.result.n_peers'`. You should have at least 10-13 peer connections.

### Setup and Configuration

**Q: How do I join a specific Botanix network (testnet vs mainnet)?**
A: The network is determined by configuration files and startup parameters. Use the specific genesis file and persistent peer list provided by the Botanix team for the network you're joining.

**Q: How do I configure my node to use an external Bitcoind instance?**
A: Modify the following parameters in your configuration:

```bash
--bitcoind.url=http://your-bitcoind-host:8332
--bitcoind.username=your-rpc-username
--bitcoind.password=your-rpc-password
```

**Q: What does the "frost.min_signers" parameter control?**
A: This parameter sets the minimum number of federation members required to sign a transaction.

### Security and Key Management

**Q: What is the recommended key rotation procedure?**
A: For CometBFT validator keys:

1. Generate new keys offline using the secp256k1 curve
2. Transfer to HSM if available
3. Update your validator configuration

See [CometBFT validators documentation](https://docs.cometbft.com/v0.37/core/validators) for detailed steps.

### Monitoring and Maintenance

**Q: What monitoring alerts should I set up?**
A: At minimum:

- Resource utilization (CPU, RAM, disk)
- Network connectivity and peer count
- Block production and consensus participation
- Disk space usage and growth rate
- Service availability for all components
- Failed transaction signing attempts

**Q: How often should I perform database backups?**
A: Daily incremental backups and weekly full backups are recommended. Critical configuration changes should trigger immediate backups.

**Q: How do I restore my node from backups?**
A: Follow the database rollback procedure in the upgrade section. Ensure you have all required components: database files, configuration files, and keys.
