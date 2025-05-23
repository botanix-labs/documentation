# Botanix RPC Node Setup Guide

This guide provides comprehensive instructions for setting up and operating a Botanix RPC node with required microservices. The setup includes Reth Ethereum node and CometBFT consensus node.

> **Update (May 2025)**: This guide has been updated to include the CLI arguments for all services and the latest configuration options for mainnet.

## Table of Contents

- [System Requirements](#system-requirements)
- [Architecture Overview](#architecture-overview)
- [Security Considerations](#security-considerations)
- [Deployment Options](#deployment-options)
  - [Docker Compose Setup](#docker-compose-setup)
  - [Binary Deployment Mode](#binary-deployment-mode)
- [Configuration](#configuration)
  - [Reth Node Configuration](#reth-node-configuration)
  - [CometBFT Consensus Node Configuration](#cometbft-consensus-node-configuration)
- [CLI Arguments](#cli-arguments)
  - [Reth Node CLI Arguments](#reth-node-cli-arguments)
  - [CometBFT CLI Arguments](#cometbft-cli-arguments)
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

### Prerequisites

- Fully synced Bitcoin Node

## Architecture Overview

Our RPC node consists of interconnected microservices:

Each component has a specific role:

- **Reth Node**: Ethereum execution layer client (cometbft abci client)
- **CometBFT Node**: Provides Byzantine Fault Tolerant consensus

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
Botanix-RPC/
  .env
  docker-compose.yml
  config/
    reth/
    cometbft/
  data/
    reth/
    cometbft/
```

#### Env File

Create a `.env` file with the following variables:

```bash
# Bitcoind user creds
BITCOIND_HOST=""
BITCOIND_USER=""
BITCOIND_PASS=""


# Optional environment variables
# P2P_LADDR=tcp://0.0.0.0:26656
# RPC_LADDR=tcp://0.0.0.0:26657
```

The `.env` file contains sensitive configuration variables that are loaded by the Docker containers. These variables should be kept secure and not committed to version control.

#### Docker Compose File

Create a `docker-compose.yml` file:

```yaml
version: '3.7'

services:
  reth-rpc-node:
    env_file:
      - .env
    container_name: reth-rpc-node
    image: us-central1-docker.pkg.dev/botanix-391913/botanix-mainnet-node/botanix-reth
    command:
      - poa
      - --federation-config-path=/reth/botanix_mainnet/config/chain.toml
      - --datadir=/reth/botanix_mainnet/data
      - --http
      - --http.addr=0.0.0.0
      - --http.port=8545
      - --http.api=debug,eth,net,trace,txpool,web3,rpc
      - --http.corsdomain=*
      - --ws
      - --ws.port=8546
      - -vvv
      - --bitcoind.url=${BITCOIND_HOST}
      - --bitcoind.username=${BITCOIND_USER}
      - --bitcoind.password=${BITCOIND_PASS}
      - --btc-network=bitcoin
      - --p2p-secret-key=/reth/botanix_mainnet/config/discovery-secret
      - --port=30303
      - --metrics=0.0.0.0:9001
      - --abci-port=26658
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
    volumes:
      - ./config/reth:/reth/botanix_mainnet/config:rw
      - ./data/reth:/reth/botanix_mainnet/data:rw
    restart: unless-stopped
    networks:
      - rpc_net

  cometbft-consensus-node:
    container_name: cometbft-consensus-node
    image: cometbft/cometbft:v1.0.0 
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
      --moniker="INPUT_YOUR_MONIKER_HERE"
      --key-type=secp256k1
      --p2p.persistent_peers=${PERSISTENT_PEERS_LIST:-}
      --p2p.laddr=${P2P_LADDR:-tcp://0.0.0.0:26656}
      --rpc.laddr=${RPC_LADDR:-tcp://0.0.0.0:26657}
    environment:
      - ALLOW_DUPLICATE_IP=TRUE
      - LOG_LEVEL=INFO
      - CHAIN_ID=3637
    networks:
      - rpc_net

networks:
  rpc_net:
    name: rpc_net
    driver: bridge
```

#### Starting the Services

```bash
cd Botanix-RPC
docker-compose up -d
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
wget https://storage.googleapis.com/botanix-artifact-registry/releases/reth/v1.1.2/reth-x86_64-unknown-linux-gnu.tar.gz

# Download the binary checksum
wget https://storage.googleapis.com/botanix-artifact-registry/releases/reth/v1.1.2/reth-x86_64-unknown-linux-gnu.tar.gz.sha256sum

# Verify binary checksum 
sha256sum -c reth-x86_64-unknown-linux-gnu.tar.gz.sha256sum

# Extract the tarball
tar -xzf reth-x86_64-unknown-linux-gnu.tar.gz

# Make the binary executable
chmod +x reth

# Optionally move to a system path
sudo mv reth /usr/local/bin/
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

Create a chain.toml config file at `config/reth/chain.toml`

```bash
reth poa \
    --federation-config-path=PATH_TO_FEDERATION_CONFIG \
    --datadir=/reth/botanix_mainnet \
    --http \
    --http.addr=0.0.0.0 \
    --http.port=8545 \
    --http.api=debug,eth,net,trace,txpool,web3,rpc \
    --http.corsdomain=* \
    --ws \
    --ws.addr=0.0.0.0 \
    --ws.port=8546 \
    --ws.api=debug,eth,net,trace,txpool,web3,rpc \ 
    -vvv \
    --bitcoind.url=${BITCOIND_HOST} \
    --bitcoind.username=${BITCOIND_USER} \
    --bitcoind.password=${BITCOIND_PASS} \
    --btc-network=bitcoin \
    --p2p-secret-key=/reth/botanix_mainnet/discovery-secret \
    --port=30303 \
    --metrics=0.0.0.0:9001 \
    --abci-port=26658 \
    --abci-host=0.0.0.0 \
    --cometbft-rpc-port=26657 \
    --cometbft-rpc-host=COMETBFT_HOST \
    --sync.enable_state_sync \
    --sync.enable_historical_sync 
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

   [mempool]
    type = "nop"
    wal_dir = ""
   ```

2. **genesis.json**:

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

## CLI Arguments

This section documents all available command-line arguments for each service, providing detailed descriptions and usage examples.

### Reth Node CLI Arguments

| Argument | Description | Default | Required |
|----------|-------------|---------|----------|
| `--federation-config-path` | Path to the federation configuration file | - | Yes |
| `--datadir` | Directory for storing node data | - | Yes |
| `--http` | Enable the HTTP server | - | No |
| `--http.addr` | HTTP server listening interface | 127.0.0.1 | No |
| `--http.port` | HTTP server listening port | 8545 | No |
| `--http.api` | API namespaces to enable on the HTTP server | eth,net,web3 | No |
| `--http.corsdomain` | CORS domains to allow | - | No |
| `--ws` | Enable the WebSocket server | - | No |
| `--ws.addr` | WebSocket server listening interface | 127.0.0.1 | No |
| `--ws.port` | WebSocket server listening port | 8546 | No |
| `--ws.api` | API namespaces to enable on the WebSocket server | eth,net,web3 | No |
| `--authrpc.addr` | Auth RPC server listening interface | 127.0.0.1 | No |
| `--authrpc.port` | Auth RPC server listening port | 8551 | No |
| `--p2p-secret-key` | Path to P2P secret key file | - | Yes |
| `--port` | P2P listening port | 30303 | No |
| `--metrics` | Metrics reporting interface:port | - | No |
| `--abci-port` | ABCI server listening port | 26658 | No |
| `--abci-host` | ABCI server listening interface | 127.0.0.1 | No |
| `--cometbft-rpc-port` | CometBFT RPC port | 26657 | No |
| `--cometbft-rpc-host` | CometBFT RPC host | - | Yes |
| `--sync.enable_state_sync` | Enable state synchronization | - | No |
| `--sync.enable_historical_sync` | Enable historical synchronization | - | No |
| `-v[vv]` | Verbosity level (v, vv, vvv) | - | No |

### CometBFT CLI Arguments

| Argument | Description | Default | Required |
|----------|-------------|---------|----------|
| `--home` | Directory for config and data | $HOME/.cometbft | No |
| `--proxy_app` | ABCI application socket address | tcp://127.0.0.1:26658 | Yes |
| `--moniker` | Node name | - | Yes |
| `--p2p.persistent_peers` | Comma-separated list of persistent peers | - | No |
| `--p2p.laddr` | P2P listen address | tcp://0.0.0.0:26656 | No |
| `--rpc.laddr` | RPC listen address | tcp://127.0.0.1:26657 | No |
| `--p2p.seed_mode` | Enable seed mode | false | No |
| `--p2p.seeds` | Comma-separated list of seed nodes | - | No |
| `--log_level` | Log level (debug, info, error) | info | No |
| `--log_format` | Log output format (plain, json) | plain | No |
| `--k` | Key algorithm to use for generating node keys | secp256k1 | No |

### Allowed Ports (Recommended)

#### Reth Node Ports

| Port   | Protocol | Functionality      | Exposure        | Required | Additional Info                                           |
|--------|----------|-------------------|-----------------|----------|----------------------------------------------------------|
| 30303  | TCP/UDP  | P2P communication | Public          | Yes      | To enable connectivity to peers                           |
| 8545   | TCP      | JSON-RPC API      | Public        | Yes      |           |
| 9001   | TCP      | Metrics Port      | Internal          | Optional |           |
| 26658  | TCP      | ABCI client port  | Internal        | Yes      | Internal to microservices                                 |
| 30304  | TCP/UDP  | P2P communication | Public          | No       | Backup/alternate P2P port (unused)                        |
| 8546   | TCP      | WebSocket Port    | Public        | Optional | WebSocket connections for subscriptions                   |
  
#### CometBFT Consensus Node Ports

| Port   | Protocol | Functionality      | Exposure     | Required | Additional Info                                 |
|--------|----------|-------------------|--------------|----------|------------------------------------------------|
| 26656  | TCP      | P2P communication | Public       | Yes      | Required for validator communication            |
| 26657  | TCP      | RPC API           | Internal     | Yes      | RPC interface for queries/transactions          |
| 26660  | TCP      | Metrics Port      | Internal       | Optional | Expose metrics for monitoring dashboard         |

## Maintenance

### Backup Procedures

1. **Database Backups**

   ```bash
   # Backup Reth data
   tar czf reth-backup-$(date +%F).tar.gz data/reth

   # Backup CometBFT data
   tar czf cometbft-backup-$(date +%F).tar.gz data/cometbft
   ```

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
   | Reth           | v1.1.2         | v1.x.x             | Minor versions are compatible  |
   | CometBFT       | v1.0.1          | v1.x.x             | Not compatible with v0.37+     |



#### Upgrade Execution

1. **Docker-based Upgrade**

   ```bash
   # Pull new images
   docker pull us-central1-docker.pkg.dev/botanix-391913/botanix-mainnet-node/botanix-reth:${VERSION_TAG}
   ```

2. **Binary-based Upgrade**

   ```bash
   # Download new binaries
   wget https://storage.googleapis.com/botanix-artifact-registry/releases/reth/v1.1.2/reth-x86_64-unknown-linux-gnu.tar.gz
   
   # Verify checksums
   sha256sum -c reth-x86_64-unknown-linux-gnu.tar.gz.sha256sum
   
   # Stop running services
   systemctl stop reth cometbft
   
   # Backup binaries
   cp /usr/local/bin/reth /usr/local/bin/reth.bak
   
   # Extract and install new binaries
   tar -xzf reth-x86_64-unknown-linux-gnu.tar.gz
   chmod +x reth
   mv reth /usr/local/bin/
   
   # Restart services
   systemctl start reth cometbft
   ```

3. **Configuration Updates**
   - Review and apply any configuration changes required by the new version
   - Update any environment variables or command line flags

#### Post-Upgrade Verification

1. **Verify Services**

   ```bash
   # Check service status
   docker-compose ps  # For Docker setup
   systemctl status reth cometbft  # For binary setup
   
   # Verify logs for errors
   docker-compose logs -f  # For Docker setup
   journalctl -u reth -u cometbft -f  # For binary setup
   ```

2. **Verify Connectivity**

   ```bash
   # Test RPC endpoints
   curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' http://localhost:8545
   ```

3. **Monitor Performance**
   - Check resource usage (CPU, memory, disk)

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
   systemctl stop reth cometbft
   
   # Restore backup binaries
   mv /usr/local/bin/reth.bak /usr/local/bin/reth
   
   # Restart services
   systemctl start reth cometbft
   ```

3. **Database Rollback**

   ```bash
   # Stop services
   docker-compose down  # For Docker setup
   systemctl stop reth cometbft  # For binary setup
   
   # Restore database backups
   rm -rf data/reth data/cometbft
   tar xzf reth-backup-YYYY-MM-DD.tar.gz
   tar xzf cometbft-backup-YYYY-MM-DD.tar.gz
   
   # Restart services
   docker-compose up -d  # For Docker setup
   systemctl start reth cometbft  # For binary setup
   ```

> **Important:** Always test upgrades in a staging environment before applying to production nodes.

## Troubleshooting

### Common Issues

1. **Reth Node Not Syncing**
   - Check network connectivity to peers
   - Check logs for error messages

2. **CometBFT Failures**
   - Check network connectivity between peers
   - Look for timeout errors in logs
   - Verify connectivity to abci client (Reth node)

## FAQ

### General Questions

**Q: How much disk space will the Reth node require over time?**
A: A Full node will grow approximately 25-35GB per month. Plan for at least 2TB of storage for a full year of operation.

**Q: What are the bandwidth requirements for running an RPC node?**
A: You should provision at least 100 Mbps of bandwidth with unlimited data transfer.

**Q: Can I run an RPC node on a virtual machine?**
A: Yes, but dedicated bare metal is also an option for production validators. If using a VM, ensure dedicated CPU cores, guaranteed RAM, and low-latency SSD storage.

**Q: How do I know if my node is properly connected to the network?**
A: Check peer connections using: `curl -s http://localhost:26657/net_info | jq '.result.n_peers'`. You should have at least 10-13 peer connections.

### Setup and Configuration

**Q: How do I join a specific Botanix network (testnet vs mainnet)?**
A: The network is determined by configuration files and startup parameters. Use the specific genesis file and persistent peer list provided by the Botanix team for the network you're joining.


### Monitoring and Maintenance

**Q: What monitoring alerts should I set up?**
A: At minimum:

- Resource utilization (CPU, RAM, disk)
- Network connectivity and peer count
- Block production
- Disk space usage and growth rate
- Service availability for all components

**Q: How often should I perform database backups?**
A: Daily incremental backups and weekly full backups are recommended. Critical configuration changes should trigger immediate backups.

**Q: How do I restore my node from backups?**
A: Follow the database rollback procedure in the upgrade section. Ensure you have all required components: database files, configuration files, and keys.