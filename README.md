# [WIP] Speed up validator setup 

A guide for validator setup. I have been playing with different hardware so actively switching machines so this acts a helpful resource to myself

## Basic Overview
System recommendation refer to [Solanahcl](https://solanahcl.org) list by [ferric](https://x.com/ferric) / [StakeWare](https://www.stakeware.xyz)

Three disks required with the following configuration:
1. SSD primary OS (~500 GB)
2. NVMe Ledger (≥2TB)
3. NVMe Accounts and snapshot (≥2TB)

Base OS: Ubuntu 22.04

## Disk Setup

Directory structure:
- Ledger Disk → `/mnt/ledger`
- Account & Snapshot Disk → `/mnt/extras`
  - `/mnt/extras/snapshot` (For Snapshots)
  - `/mnt/extras/accounts` (For Accounts)

### Setup Steps

1. Format the block
```bash
sudo mkfs -t ext4 /dev/nvme0n1
```

2. Spin up directory + give sol user permission
```bash
sudo chown -R sol:sol <PATH TO DIR>
```

3. Mount to the directory
```bash
sudo mount /dev/nvme0n1 <PATH TO DIR>
```

## Ports Opening

Note: RPC port remains closed, only SSH and gossip ports are opened.

For new machines with UFW disabled:
1. Add OpenSSH first to prevent lockout if you don't have password access
2. Open required ports:
```bash
sudo ufw allow 8000:8020/tcp
```
```bash
sudo ufw allow 8000:8020/udp
```


# System Tuning and Validator Setup

## System Performance Optimization

### Kernel and Network Tuning
Create and run the following script to optimize system performance:

```bash
#!/bin/bash

# Set sysctl performance variables
cat >> /etc/sysctl.conf <<- EOM
# TCP Buffer Sizes (10k min, 87.38k default, 12M max)
net.ipv4.tcp_rmem=10240 87380 12582912
net.ipv4.tcp_wmem=10240 87380 12582912

# TCP Optimization
net.ipv4.tcp_congestion_control=westwood
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_timestamps=0
net.ipv4.tcp_sack=1
net.ipv4.tcp_low_latency=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_no_metrics_save=1
net.ipv4.tcp_moderate_rcvbuf=1

# Kernel Optimization
kernel.timer_migration=0
kernel.hung_task_timeout_secs=30
kernel.pid_max=49152

# Virtual Memory Tuning
vm.swappiness=30
vm.max_map_count=2000000
vm.stat_interval=10
vm.dirty_ratio=40
vm.dirty_background_ratio=10
vm.min_free_kbytes=3000000
vm.dirty_expire_centisecs=36000
vm.dirty_writeback_centisecs=3000
vm.dirtytime_expire_seconds=43200

# Solana Specific Tuning
net.core.rmem_max=134217728
net.core.rmem_default=134217728
net.core.wmem_max=134217728
net.core.wmem_default=134217728
EOM

# Reload sysctl settings
sysctl -p

# Set CPU governor to performance mode
echo 'GOVERNOR="performance"' | tee /etc/default/cpufrequtils
echo "performance" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Set performance governor for bare metal (ignore errors)
echo "performance" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor || true
```

### Session File Limits
Choose one of the following configurations:

1. Service-specific limits in `/etc/systemd/system.conf`:
```ini
[Service]
LimitNOFILE=1000000
```

2. System-wide limits in `/etc/systemd/system.conf`:
```ini
[Manager]
DefaultLimitNOFILE=1000000
```

## Validator Setup

### Installing Agave/Jito Client

1. Grant execution permissions to the install script:
```bash
chmod +x bin/ice-staking/start/init.sh
```

2. Run the installation with specific version tag:
```bash
bin/ice-staking/start/init.sh -t v1.18.23-jito
```

### Post-Installation Setup

1. Create symlink for Jito client (if used):
```bash
ln -sf /home/sol/.local/share/solana/install/releases/v1.18.15-jito/bin /home/sol/.local/share/solana/install/active_release/
```

2. Add the following to your `.bashrc` or `.bash_profile`:
```bash
# Environment Setup
. "$HOME/.cargo/env"
export PATH="/home/sol/.local/share/solana/install/active_release/bin:$PATH"

# Helpful Aliases
alias catchup='solana catchup --our-localhost'
alias monitor='solana-validator --ledger /mnt/ledger monitor'
alias logtail='tail -f /home/sol/solana-validator.log'
```

### Additional Resources
- Installation script source: [ice-staking repository](https://github.com/dhruvsol/ice-staking/blob/main/start/init.sh)


# Hot-Swap Validator Setup Guide

## Overview
This guide describes how to set up two servers for hot-swapping to maintain 100% uptime during system changes. The process follows the [Identity Transition](https://pumpkins-pool.gitbook.io/pumpkins-pool) methodology by Pumpkin.

## Identity Keypair Configuration

### Required Keypairs
1. **Unstaked Keypair** (`unstaked.json`)
   - Functions as a burner keypair
   - Maintains zero SOL balance to prevent voting capabilities

2. **Staked Keypair** (`staked.json`)
   - Serves as the primary staked keypair
   - Used for validator transitions when needed

### Transferring Keypairs
Transfer the keypairs to your validator server using SCP:
```bash
scp <source_files> ice-ams:
```
> **Note**: Customize the SSH configuration according to your setup. Ensure proper permissions are set for the `sol` user after transfer.

## Log Rotation Configuration

Create and implement log rotation for validator logs:

```bash
cat > logrotate.sol <<EOF
/home/sol/solana-validator.log {
    rotate 7
    daily
    missingok
    postrotate
        systemctl kill -s USR1 sol.service
    endscript
}
EOF

sudo cp logrotate.sol /etc/logrotate.d/sol
systemctl restart logrotate.service
```

## Systemd Service Configuration

Create a systemd service file for the Solana validator:

```ini
[Unit]
Description=Solana Validator
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=sol
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
Environment="SOLANA_METRICS_CONFIG=host=https://metrics.solana.com:8086,db=mainnet-beta,u=mainnet-beta_write,p=password"
Environment="PATH=/home/sol/bin:/home/sol/.local/share/solana/install/active_release/bin:/home/sol/.cargo/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
ExecStart=/home/sol/bin/ice-staking/start/start.sh mainnet-beta

[Install]
WantedBy=multi-user.target
```

## Service Management Commands

### Start Service
```bash
sudo systemctl enable --now sol
```

### Stop Service
```bash
sudo systemctl stop sol
```

### Restart Service
```bash
sudo systemctl restart sol
```
  
After this check the log file snapshot download should have started

```
tail -f solana-validator.log 
```

# Solana Validator Operations Guide

## Metrics & Monitoring Solutions

### 1. Built-in Dashboard Options
- **Solana Metrics Dashboard**
  - Official solution from Solana Labs
  - Access via URL specified in service file
  - Provides real-time validator performance metrics

### 2. Third-Party Solutions
- **Stakeconomy's SolanaMonitoring**
  - Repository: [github.com/stakeconomy/solanamonitoring](https://github.com/stakeconomy/solanamonitoring)
  - Community-maintained monitoring solution
  - Features:
    - Performance tracking
    - Health checks

### 3. Custom Monitoring Stack
- **Grafana + InfluxDB Setup**
  - Fully customizable metrics visualization
  - Time-series data storage
  - Benefits:
    - Custom dashboards
    - Historical data analysis

## Active Monitoring Tools

### 1. Solana Watcher
- Official monitoring tool by Solana Labs
- Documentation: [docs.solanalabs.com/operations/best-practices/monitoring](https://docs.solanalabs.com/operations/best-practices/monitoring)
- Features:
  - Automated health checks
  - System alerts

### 2. Stakewiz Update bot Integration
- Telegram notification system
- Real-time alerts and updates

## Security Best Practices

### 1. Firewall Configuration
- Only open required ports
- Implement port-specific rules
- Regular audit of open ports
- Use UFW (Uncomplicated Firewall) for simple management

### 2. User Management
- ✅ Run validator with non-root user
- ❌ Avoid running as root
- ❌ Validator user should not have sudo privileges
- Create dedicated service account for validator operations

### 3. SSH Security
- Disable password authentication
- Use SSH keys exclusively
- Consider:
  - Custom SSH port
  - Key-based authentication only
  - Rate limiting for failed attempts

### 4. Keypair Security
- Secure storage of validator keypairs
- Best practices:
  - Encrypted backups
  - Access control logs






