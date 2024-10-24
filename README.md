# [WIP] Speed up validator setup 

A guide for validator setup. I have been planing with different hardware so actively switching machines so this acts a helpful resource to myself

## Basic Overview

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


## HotSwap Setup 
I use 2 servers to hot-swap to maintain 100% uptime while changing anything in the system. the guide I follow is [Identity Transition](https://pumpkins-pool.gitbook.io/pumpkins-pool) by Pumpkin 

### Identity Keypair 

1. unstaked.json
  - This is a burner keypair without sol so it won't be able to vote
2. staked.json
  - This is the main staked keypair which will we swap it in case we want 




