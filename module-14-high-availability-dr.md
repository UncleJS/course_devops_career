# Module 14: High Availability & Disaster Recovery

> **Course**: DevOps Career Path  
> **Audience**: Beginner → Intermediate  
> **Prerequisites**: Module 06 (Kubernetes), Module 07 (Cloud Fundamentals), Module 09 (Ansible)

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 14 of 15](https://img.shields.io/badge/module-14%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Advanced-red) ![MySQL 8.4+](https://img.shields.io/badge/MySQL-8.4%2B-4479A1?logo=mysql&logoColor=white) ![HAProxy 3.1+](https://img.shields.io/badge/HAProxy-3.1%2B-106DA9) ![Keepalived 2.3+](https://img.shields.io/badge/Keepalived-2.3%2B-grey) ![Replication · Failover](https://img.shields.io/badge/patterns-Replication%20%C2%B7%20Failover-blue)

---

## Table of Contents

1. [Overview](#overview)
2. [Learning Objectives](#learning-objectives)
3. [Core Concepts: RTO, RPO, SLA](#core-concepts-rto-rpo-sla)
4. [Availability Tiers & Nines](#availability-tiers--nines)
5. [Failure Mode Analysis](#failure-mode-analysis)
6. [High Availability Architecture Patterns](#high-availability-architecture-patterns)
7. [Load Balancing](#load-balancing)
8. [Database High Availability](#database-high-availability)
9. [Backup Strategies](#backup-strategies)
10. [Multi-Region Architecture](#multi-region-architecture)
11. [Kubernetes HA](#kubernetes-ha)
12. [Disaster Recovery Planning](#disaster-recovery-planning)
13. [DR Testing & Chaos Engineering](#dr-testing--chaos-engineering)
14. [Cloud HA Services](#cloud-ha-services)
15. [Tools & Commands Reference](#tools--commands-reference)
16. [Hands-On Labs](#hands-on-labs)
17. [Further Reading](#further-reading)

---

## Overview

High Availability (HA) and Disaster Recovery (DR) are the engineering disciplines that ensure your systems survive component failures, data center outages, and catastrophic events. HA focuses on eliminating single points of failure to maximize uptime. DR focuses on restoring operations after a significant failure with defined time and data loss targets. This module covers both disciplines with practical implementation patterns.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module, you will be able to:

- Define and calculate RTO, RPO, MTTR, and MTBF
- Translate business SLA requirements into infrastructure designs
- Identify and eliminate single points of failure (SPOFs)
- Design active-active and active-passive HA configurations
- Configure HAProxy and Nginx for load balancing
- Set up MySQL/MariaDB primary-replica and Galera cluster replication
- Design and implement backup strategies with automated verification
- Architect multi-region deployments for AWS, Azure, and GCP
- Configure Kubernetes for production-grade HA
- Write and execute DR runbooks
- Run chaos engineering experiments with Chaos Monkey and Chaos Mesh

[↑ Back to TOC](#table-of-contents)

---

## Core Concepts: RTO, RPO, SLA

### Key metrics

| Metric | Full Name | Definition | Example |
|--------|-----------|------------|---------|
| **RTO** | Recovery Time Objective | Max acceptable time to restore service | 4 hours |
| **RPO** | Recovery Point Objective | Max acceptable data loss (time) | 1 hour |
| **MTBF** | Mean Time Between Failures | Average time between failures | 720 hours |
| **MTTR** | Mean Time To Repair/Recover | Average time to restore after failure | 2 hours |
| **Availability** | Uptime percentage | MTBF / (MTBF + MTTR) | 99.72% |
| **MTTD** | Mean Time To Detect | Average time to detect a failure | 5 minutes |

### RTO vs RPO visualized

```
Last backup          Failure           Recovery complete
     │                  │                     │
─────▼──────────────────▼─────────────────────▼─────────
     │◄──── RPO ────────►│◄────── RTO ─────────►│
     │    (data loss)     │    (downtime)        │
     │    max 1 hour      │    max 4 hours       │
```

### Translating business requirements

| Business Requirement | RTO | RPO | Tier |
|---------------------|-----|-----|------|
| Core payment processing | < 1 min | 0 (no data loss) | Mission Critical |
| Customer-facing website | < 15 min | < 5 min | Business Critical |
| Internal admin portal | < 4 hours | < 1 hour | Important |
| Batch reporting system | < 24 hours | < 24 hours | Standard |
| Dev/test environment | < 72 hours | < 1 week | Low priority |

[↑ Back to TOC](#table-of-contents)

---

## Availability Tiers & Nines

| Availability | Downtime / Year | Downtime / Month | Tier |
|-------------|----------------|-----------------|------|
| 99% | 3.65 days | 7.3 hours | Basic |
| 99.9% | 8.76 hours | 43.8 minutes | Standard |
| 99.95% | 4.38 hours | 21.9 minutes | High |
| 99.99% | 52.6 minutes | 4.38 minutes | Very High |
| 99.999% | 5.26 minutes | 26.3 seconds | Five Nines |
| 99.9999% | 31.5 seconds | 2.63 seconds | Six Nines |

> Achieving five nines requires eliminating all single points of failure, including in the deployment process itself. A 30-minute maintenance window occurs 6 times per year — that alone costs you 3 hours, which breaks 99.97%.

### Uptime calculation

```
Availability = (Total Time - Downtime) / Total Time × 100%

Example: 8 hours downtime in a year
= (8760 - 8) / 8760 × 100%
= 99.909%
```

[↑ Back to TOC](#table-of-contents)

---

## Failure Mode Analysis

### Single Points of Failure (SPOF) — common examples

```
❌ Single web server                → ✅ Multiple servers + load balancer
❌ Single database (no replica)     → ✅ Primary + replica(s) or cluster
❌ Single load balancer             → ✅ HA pair (active/passive) or cloud LB
❌ Single availability zone         → ✅ Multi-AZ deployment
❌ Single region                    → ✅ Multi-region (for DR)
❌ Single power feed                → ✅ Redundant PDUs (data center)
❌ Single DNS provider              → ✅ Secondary DNS / Route53 + Cloudflare
❌ Single CI/CD server              → ✅ Managed CI/CD or HA runner pool
❌ Single network path (BGP)        → ✅ Multiple ISPs / BGP multi-homing
❌ Single cloud account             → ✅ Account isolation + cross-account DR
```

### SPOF identification template

```
For each system component, ask:
1. What happens if this component fails?
2. How long until failure is detected?
3. How long until failover or recovery?
4. Is there data loss?
5. What is the blast radius?
```

[↑ Back to TOC](#table-of-contents)

---

## High Availability Architecture Patterns

### Active-Passive (Failover)

```
         Client
           │
           ▼
     ┌─────────────┐
     │  Primary    │◄──── Receives all traffic (ACTIVE)
     │  (active)   │
     └──────┬──────┘
            │ Heartbeat + replication
     ┌──────▼──────┐
     │  Secondary  │◄──── Standby, no traffic (PASSIVE)
     │  (passive)  │      Promotes if primary fails
     └─────────────┘
```

**Use case**: Databases (MySQL primary-replica), Keepalived/VRRP, some stateful applications  
**RTO**: Seconds to minutes (failover time)  
**Cost**: Pay for standby capacity at all times

### Active-Active (Load Sharing)

```
         Client
           │
           ▼
    ┌─────────────┐
    │ Load        │
    │ Balancer    │
    └──┬──────┬───┘
       │      │
┌──────▼──┐ ┌─▼──────┐
│ Server  │ │ Server │◄── Both handle traffic simultaneously
│   01    │ │   02   │
└─────────┘ └────────┘
```

**Use case**: Stateless web/app servers, microservices  
**RTO**: Near-zero (load balancer removes failed node automatically)  
**Cost**: All capacity is productive

### N+1 Redundancy

Have one more unit than the minimum required.

```
Minimum needed: 2 servers to handle load
N+1:            3 servers deployed
If 1 fails:     2 remaining still meet load requirements
```

### Geographic redundancy tiers

| Tier | Scope | RTO | Use case |
|------|-------|-----|----------|
| **Rack-level** | Same server room | Seconds | Standard HA |
| **AZ-level** | Same city, different data centers | < 1 min | Cloud HA |
| **Region-level** | Same country, 100s of km apart | Minutes–hours | Regional DR |
| **Cross-region** | Different countries/continents | Hours | Full DR |

[↑ Back to TOC](#table-of-contents)

---

## Load Balancing

### HAProxy configuration

```
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    maxconn 50000
    user haproxy
    group haproxy
    daemon
    stats socket /var/lib/haproxy/stats level admin expose-fd listeners

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option forwardfor
    option http-server-close
    retries 3
    timeout connect 5s
    timeout client 50s
    timeout server 50s
    timeout http-request 10s
    timeout http-keep-alive 10s
    timeout queue 30s

# Stats page
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats auth admin:StatsPassword123

# HTTP → HTTPS redirect
frontend http_in
    bind *:80
    redirect scheme https code 301

# HTTPS frontend
frontend https_in
    bind *:443 ssl crt /etc/haproxy/certs/example.com.pem
    default_backend web_backend

    # ACL routing
    acl is_api path_beg /api/
    use_backend api_backend if is_api

# Web servers backend
backend web_backend
    balance roundrobin
    option httpchk GET /health HTTP/1.1\r\nHost:\ example.com
    http-check expect status 200
    default-server inter 3s fall 3 rise 2
    server web-01 192.168.1.10:8080 check
    server web-02 192.168.1.11:8080 check
    server web-03 192.168.1.12:8080 check backup  # backup: only if all active fail

# API servers backend
backend api_backend
    balance leastconn    # Use least connections for API
    option httpchk GET /api/health
    http-check expect status 200
    default-server inter 5s fall 2 rise 3 slowstart 60s
    server api-01 192.168.1.20:3000 check
    server api-02 192.168.1.21:3000 check

# TCP backend example (PostgreSQL)
listen postgres_cluster
    bind *:5432
    mode tcp
    balance roundrobin
    option tcp-check
    default-server inter 3s fall 3 rise 2
    server pg-primary 192.168.1.30:5432 check
    server pg-replica 192.168.1.31:5432 check backup
```

### HAProxy balance algorithms

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| `roundrobin` | Requests distributed evenly | Stateless apps of similar capacity |
| `leastconn` | Send to server with fewest connections | Long-lived connections (DB, WebSocket) |
| `source` | Client IP hashing (sticky) | Session affinity required |
| `uri` | URL hashing | Cache backends |
| `random` | Random server | Stateless, large server pools |

### HAProxy runtime management

```bash
# Reload config without dropping connections
haproxy -f /etc/haproxy/haproxy.cfg -c   # Validate config
systemctl reload haproxy                  # Graceful reload

# Check server states
echo "show servers state" | socat stdio /var/lib/haproxy/stats

# Drain a server (stop sending new connections)
echo "set server web_backend/web-01 state drain" | socat stdio /var/lib/haproxy/stats

# Disable a server (remove from pool for maintenance)
echo "set server web_backend/web-01 state maint" | socat stdio /var/lib/haproxy/stats

# Re-enable a server
echo "set server web_backend/web-01 state ready" | socat stdio /var/lib/haproxy/stats
```

### Keepalived — Virtual IP failover (VRRP)

```bash
# /etc/keepalived/keepalived.conf — on PRIMARY (MASTER)
vrrp_script chk_haproxy {
    script "killall -0 haproxy"   # Check if HAProxy is running
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 110              # Higher = preferred master
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secretpassword
    }

    virtual_ipaddress {
        192.168.1.100/24      # VIP — floats to whichever node is master
    }

    track_script {
        chk_haproxy
    }

    notify_master "/etc/keepalived/notify.sh MASTER"
    notify_backup "/etc/keepalived/notify.sh BACKUP"
    notify_fault  "/etc/keepalived/notify.sh FAULT"
}
```

```bash
# On BACKUP node — same config with:
# state BACKUP
# priority 100    (lower than master)
```

[↑ Back to TOC](#table-of-contents)

---

## Database High Availability

### MySQL / MariaDB Primary-Replica Replication

```bash
# --- PRIMARY SERVER ---

# /etc/mysql/conf.d/replication.cnf
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_expire_logs_seconds = 604800
max_binlog_size = 100M
binlog_format = ROW    # ROW is safer than STATEMENT
innodb_flush_log_at_trx_commit = 1   # Sync on every commit (ACID)
sync_binlog = 1

# Create replication user on primary
CREATE USER 'replicator'@'192.168.1.%' IDENTIFIED BY 'ReplPassword123!';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'192.168.1.%';
FLUSH PRIVILEGES;

# Get current binary log position (lock tables briefly)
FLUSH TABLES WITH READ LOCK;
SHOW BINARY LOG STATUS;   -- MySQL 8.4+ / SHOW MASTER STATUS on older versions
# Note: File and Position
UNLOCK TABLES;
```

```bash
# --- REPLICA SERVER ---

# /etc/mysql/conf.d/replication.cnf
[mysqld]
server-id = 2
relay_log = /var/log/mysql/mysql-relay-bin.log
read_only = 1
log_replica_updates = 1    # Allow replica to also be replicated to (chain)
# Note: log_slave_updates is a deprecated alias removed in MySQL 9.0

# Configure replication (MySQL 8.0.23+ syntax — CHANGE REPLICATION SOURCE TO)
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='192.168.1.30',
    SOURCE_USER='replicator',
    SOURCE_PASSWORD='ReplPassword123!',
    SOURCE_LOG_FILE='mysql-bin.000001',  # From SHOW BINARY LOG STATUS
    SOURCE_LOG_POS=154;

START REPLICA;

# Check status (MySQL 8.0.22+; use SHOW SLAVE STATUS\G on MySQL < 8.0.22)
SHOW REPLICA STATUS\G
# Look for:
# Replica_IO_Running: Yes
# Replica_SQL_Running: Yes
# Seconds_Behind_Source: 0
```

### MariaDB Galera Cluster (Multi-Master)

```ini
# /etc/mysql/conf.d/galera.cnf — on ALL nodes

[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

# Galera Provider
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Cluster name and members
wsrep_cluster_name="my_galera_cluster"
wsrep_cluster_address="gcomm://db-01,db-02,db-03"

# This node's address
wsrep_node_address="192.168.1.30"    # Change per node
wsrep_node_name="db-01"              # Change per node

# SST (State Snapshot Transfer) method
wsrep_sst_method=rsync
```

```bash
# Bootstrap the cluster (first node ONLY — once only)
galera_new_cluster

# Join other nodes
systemctl start mysql     # On db-02 and db-03

# Check cluster status
mysql -e "SHOW STATUS LIKE 'wsrep_%';" | grep -E "wsrep_cluster_size|wsrep_local_state_comment|wsrep_connected"
```

### PostgreSQL HA with Patroni

Patroni is a Python-based HA solution for PostgreSQL using distributed consensus (etcd/Consul/ZooKeeper).

```yaml
# /etc/patroni/patroni.yml (on each node)
scope: my-postgres-cluster
namespace: /db/
name: pg-node-01   # Change per node

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.40:8008

etcd:
  hosts: etcd-01:2379,etcd-02:2379,etcd-03:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 30
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_connections: 200
        max_wal_senders: 10
        max_replication_slots: 10

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.40:5432
  data_dir: /var/lib/postgresql/14/data
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: ReplPassword123!
    superuser:
      username: postgres
      password: SuperPassword456!
```

```bash
# Check cluster status
patronictl -c /etc/patroni/patroni.yml list

# Manual failover
patronictl -c /etc/patroni/patroni.yml failover my-postgres-cluster --master pg-node-01 --candidate pg-node-02 --force

# Switchover (planned maintenance)
patronictl -c /etc/patroni/patroni.yml switchover my-postgres-cluster
```

### Redis Sentinel (HA for Redis)

```conf
# sentinel.conf
sentinel monitor mymaster 192.168.1.50 6379 2   # 2 sentinels must agree
sentinel auth-pass mymaster RedisPassword123!
sentinel down-after-milliseconds mymaster 5000   # 5s before marking down
sentinel failover-timeout mymaster 60000         # Max failover time
sentinel parallel-syncs mymaster 1               # Replicas to sync at once
```

```bash
redis-sentinel /etc/redis/sentinel.conf &

# Check sentinel status
redis-cli -p 26379 sentinel masters
redis-cli -p 26379 sentinel slaves mymaster
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
```

[↑ Back to TOC](#table-of-contents)

---

## Backup Strategies

### 3-2-1 Backup Rule

```
3 copies of data
2 different storage media types
1 copy offsite (geographically separate)

Example:
  Copy 1: Production database (primary)
  Copy 2: Daily backup on NAS in same DC
  Copy 3: Daily backup in S3 (offsite)
```

### Backup types

| Type | Description | Restore Speed | Storage |
|------|-------------|---------------|---------|
| **Full** | Complete copy of all data | Fastest | Large |
| **Incremental** | Changes since last backup (any type) | Slow (chain restore) | Small |
| **Differential** | Changes since last FULL backup | Medium | Medium |
| **Snapshot** | Point-in-time storage snapshot | Fast | Medium |
| **Continuous (CDC)** | Continuous log shipping / CDC | Near-zero RPO | Ongoing |

### MySQL / MariaDB backup with mysqldump

```bash
#!/bin/bash
# /usr/local/bin/mysql-backup.sh

set -euo pipefail

DB_HOST="localhost"
DB_USER="backup_user"
DB_PASS="BackupPassword123!"
BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup directory
mkdir -p "${BACKUP_DIR}/${DATE}"

# Backup all databases individually
mysql -h"${DB_HOST}" -u"${DB_USER}" -p"${DB_PASS}" \
  -e "SHOW DATABASES;" --batch --skip-column-names | \
  grep -Ev "^(information_schema|performance_schema|sys)$" | \
  while read db; do
    echo "Backing up: ${db}"
    mysqldump -h"${DB_HOST}" -u"${DB_USER}" -p"${DB_PASS}" \
      --single-transaction \
      --routines \
      --triggers \
      --events \
      --flush-logs \
      "${db}" | gzip > "${BACKUP_DIR}/${DATE}/${db}.sql.gz"
  done

# Create checksum file
md5sum "${BACKUP_DIR}/${DATE}"/*.sql.gz > "${BACKUP_DIR}/${DATE}/checksums.md5"

# Upload to S3
aws s3 sync "${BACKUP_DIR}/${DATE}/" \
  "s3://my-db-backups/mysql/${DATE}/" \
  --storage-class STANDARD_IA

# Verify upload
aws s3 ls "s3://my-db-backups/mysql/${DATE}/" --summarize

# Remove old local backups
find "${BACKUP_DIR}" -maxdepth 1 -type d -mtime "+${RETENTION_DAYS}" -exec rm -rf {} +

echo "Backup complete: ${DATE}"
```

### MySQL backup with Percona XtraBackup (hot backup)

```bash
# Full backup (no table locks — safe for production)
xtrabackup --backup \
  --user=backup_user \
  --password=BackupPassword123! \
  --target-dir=/backup/mysql/full/$(date +%Y%m%d)

# Prepare (apply transaction logs)
xtrabackup --prepare --target-dir=/backup/mysql/full/20260302

# Restore
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backup/mysql/full/20260302
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

### Filesystem and database snapshot backup

```bash
# LVM snapshot for consistent filesystem backup
# 1. Create snapshot of /dev/vg0/data (10GB snapshot space)
lvcreate -L10G -s -n data_snap /dev/vg0/data

# 2. Mount the snapshot
mount -o ro /dev/vg0/data_snap /mnt/snapshot

# 3. Back up the snapshot
tar czf /backup/data-$(date +%Y%m%d).tar.gz -C /mnt/snapshot .

# 4. Remove snapshot
umount /mnt/snapshot
lvremove -f /dev/vg0/data_snap
```

### Kubernetes backup with Velero

```bash
# Install Velero with S3 storage
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# Create a backup
velero backup create production-backup \
  --include-namespaces production \
  --ttl 720h    # 30 days

# Schedule daily backups
velero schedule create daily-production \
  --schedule="0 2 * * *" \
  --include-namespaces production \
  --ttl 720h

# Check backup status
velero backup describe production-backup
velero backup logs production-backup

# Restore from backup
velero restore create --from-backup production-backup
velero restore describe production-backup-restore-1

# List all backups
velero backup get
```

### Backup verification (critical!)

> A backup is only as good as its last successful restore test. Automated verification is mandatory.

```bash
#!/bin/bash
# /usr/local/bin/verify-backup.sh
# Run daily — restore latest backup to test instance and verify data

BACKUP_DATE=$(date +%Y%m%d)
TEST_DB="backup_verify_${BACKUP_DATE}"
RESTORE_HOST="db-test-01"

# Download latest backup
aws s3 cp "s3://my-db-backups/mysql/${BACKUP_DATE}/appdb.sql.gz" /tmp/

# Verify checksum
aws s3 cp "s3://my-db-backups/mysql/${BACKUP_DATE}/checksums.md5" /tmp/
md5sum --check /tmp/checksums.md5

# Restore to test database
mysql -h"${RESTORE_HOST}" -u root -p -e "CREATE DATABASE ${TEST_DB};"
zcat /tmp/appdb.sql.gz | mysql -h"${RESTORE_HOST}" -u root -p "${TEST_DB}"

# Verify row counts match
PROD_COUNT=$(mysql -h db-primary -u app_user -p appdb -se "SELECT COUNT(*) FROM orders;")
TEST_COUNT=$(mysql -h"${RESTORE_HOST}" -u root -p "${TEST_DB}" -se "SELECT COUNT(*) FROM orders;")

if [ "${PROD_COUNT}" == "${TEST_COUNT}" ]; then
    echo "✅ Backup verification PASSED: ${PROD_COUNT} orders verified"
else
    echo "❌ Backup verification FAILED: prod=${PROD_COUNT}, backup=${TEST_COUNT}"
    # Send alert
    curl -X POST "$SLACK_WEBHOOK" \
      -d "{\"text\": \"⚠️ DB Backup verification FAILED for ${BACKUP_DATE}\"}"
    exit 1
fi

# Cleanup
mysql -h"${RESTORE_HOST}" -u root -p -e "DROP DATABASE ${TEST_DB};"
rm -f /tmp/appdb.sql.gz /tmp/checksums.md5
```

[↑ Back to TOC](#table-of-contents)

---

## Multi-Region Architecture

### Active-Passive Multi-Region

```
Region A (Primary — active)        Region B (DR — passive)
┌───────────────────────────┐      ┌───────────────────────────┐
│  Load Balancer             │      │  Load Balancer (standby)   │
│  App servers (3x)          │  ──► │  App servers (warm standby)│
│  DB Primary                │ repl │  DB Replica                │
│  Cache (Redis)             │      │  Cache                     │
└───────────────────────────┘      └───────────────────────────┘
        ▲                                      ▲
        │           DNS Failover               │
        └──────────────────────────────────────┘
              Route53 / Cloudflare / Azure DNS
              Health check → failover in ~60s
```

### Active-Active Multi-Region

```
Region A                            Region B
┌────────────────────────┐          ┌────────────────────────┐
│  App servers           │          │  App servers           │
│  DB (read + write)     │◄────────►│  DB (read + write)     │
│  Cache                 │          │  Cache                 │
└────────────────────────┘          └────────────────────────┘
          ▲                                   ▲
          │         Global Load Balancer       │
          └───────────────────────────────────┘
              Cloudflare / AWS Global Accel.
              Latency-based routing
```

### AWS Multi-Region DNS failover (Route 53)

```bash
# Create primary health check
aws route53 create-health-check \
  --caller-reference "primary-check-001" \
  --health-check-config '{
    "IPAddress": "52.1.2.3",
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "api.example.com",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'

# Create PRIMARY record (us-east-1)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "52.1.2.3"}],
        "HealthCheckId": "<health-check-id>"
      }
    }]
  }'

# Create SECONDARY record (eu-west-1)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "secondary",
        "Failover": "SECONDARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "34.5.6.7"}]
      }
    }]
  }'
```

[↑ Back to TOC](#table-of-contents)

---

## Kubernetes HA

### Control Plane HA

For production Kubernetes, run **3 or 5 control plane nodes** (odd number for etcd quorum).

```
┌─────────────────────────────────────────────────────────┐
│                  HA Control Plane                       │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Control      │  │ Control      │  │ Control      │  │
│  │ Plane 1      │  │ Plane 2      │  │ Plane 3      │  │
│  │ API Server   │  │ API Server   │  │ API Server   │  │
│  │ Scheduler    │  │ Scheduler    │  │ Scheduler    │  │
│  │ Controller   │  │ Controller   │  │ Controller   │  │
│  │ etcd         │  │ etcd         │  │ etcd         │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│         │                │                 │            │
│         └────────────────┼─────────────────┘            │
│                          │                              │
│              ┌───────────▼──────────┐                   │
│              │  Load Balancer (VIP) │                   │
│              │  (haproxy/nlb/kube)  │                   │
│              └──────────────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### etcd backup and restore

```bash
# Backup etcd (run on control plane node)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d_%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-20260302_140000.db --write-out=table

# Restore etcd
systemctl stop kube-apiserver kube-scheduler kube-controller-manager etcd

ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20260302_140000.db \
  --data-dir=/var/lib/etcd-restore \
  --name=cp-01 \
  --initial-cluster="cp-01=https://192.168.1.10:2380,cp-02=https://192.168.1.11:2380,cp-03=https://192.168.1.12:2380" \
  --initial-cluster-token=etcd-cluster-prod \
  --initial-advertise-peer-urls=https://192.168.1.10:2380

# Update etcd config to use new data directory
# Restart services
systemctl start etcd kube-apiserver kube-scheduler kube-controller-manager
```

### Pod Disruption Budgets (PDB)

PDBs ensure a minimum number of pods are always available during voluntary disruptions (node drains, rolling updates).

```yaml
# Ensure at least 2 of 3 replicas are always available
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-api-pdb
  namespace: production
spec:
  minAvailable: 2       # Can also use: maxUnavailable: 1
  selector:
    matchLabels:
      app: my-api
```

### Node Affinity — spread across AZs

```yaml
# Ensure pods are spread across availability zones
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: my-api

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-api
          topologyKey: kubernetes.io/hostname  # One pod per node
```

[↑ Back to TOC](#table-of-contents)

---

## Disaster Recovery Planning

### DR Strategies (by cost/RTO)

| Strategy | RTO | RPO | Cost | Description |
|----------|-----|-----|------|-------------|
| **Backup & Restore** | Hours | Hours | $ | Restore from backup in DR region |
| **Pilot Light** | 10–30 min | Minutes | $$ | Minimal DR environment always running; scale up on disaster |
| **Warm Standby** | Minutes | Seconds–minutes | $$$ | Reduced-capacity DR environment always running |
| **Active-Active** | Near-zero | Near-zero | $$$$ | Full capacity in both regions; automatic failover |

### DR Runbook template

```markdown
# DR Runbook: Production Database Failure

## Trigger conditions
- Primary database unreachable for > 5 minutes
- Replication lag > 30 minutes
- Data corruption detected

## Severity: P1 — Critical

## Stakeholders
- On-call engineer (PagerDuty escalation)
- Database team lead
- Engineering manager (if > 30 min)
- Communications (if customer-facing > 1 hour)

## Step 1 — Validate the failure (5 min)
1. Check database health: `mysql -h db-primary -u monitor -e "SELECT 1;"`
2. Check replica status: `SHOW REPLICA STATUS\G`
3. Check HAProxy status: http://haproxy:8404/stats
4. Check CloudWatch/Prometheus alerts

## Step 2 — Escalation decision (2 min)
- If primary unreachable AND replica lag = 0: Proceed to Step 3 (promote replica)
- If primary unreachable AND replica lag > 0: Assess data loss vs downtime tradeoff

## Step 3 — Promote replica (10 min)
```sql
-- On replica server (MySQL 8.0.22+ syntax)
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL read_only = 0;
```

## Step 4 — Update HAProxy (2 min)
```bash
echo "set server postgres_cluster/pg-replica state active" | socat stdio /var/lib/haproxy/stats
echo "set server postgres_cluster/pg-primary state maint" | socat stdio /var/lib/haproxy/stats
```

## Step 5 — Validate
1. Test read/write connectivity to new primary
2. Check application health endpoints
3. Monitor error rates for 10 minutes

## Step 6 — Communication
- Update status page (status.example.com)
- Notify stakeholders via Slack #incidents
- Start incident ticket in PagerDuty

## Step 7 — Post-incident
- Schedule post-mortem within 48 hours
- Fix original primary and re-establish replication
- Update this runbook if process was unclear
```

### DR testing checklist

```
Quarterly DR Test Checklist:
□ Notify stakeholders of planned test date
□ Schedule test during low-traffic window
□ Document current state (backup timestamps, replication lag)
□ Execute failover procedure
□ Verify application functionality after failover
□ Measure actual RTO achieved
□ Measure actual data loss (RPO achieved)
□ Document any gaps from target RTO/RPO
□ Restore to normal state
□ Update runbooks with lessons learned
□ Schedule next test
```

[↑ Back to TOC](#table-of-contents)

---

## DR Testing & Chaos Engineering

### Chaos Engineering principles

> "Chaos Engineering is the discipline of experimenting on a system in order to build confidence in the system's capability to withstand turbulent conditions in production." — Netflix

#### GameDay process

1. **Define steady state** — what does "normal" look like? (error rate, latency, throughput)
2. **Hypothesize** — "If we kill one web server, steady state will be maintained"
3. **Introduce chaos** — kill the server, inject latency, corrupt a request
4. **Observe** — did steady state hold?
5. **Fix weaknesses** — if steady state broke, fix the SPOF
6. **Repeat**

### Chaos Mesh — Kubernetes chaos engineering

```bash
# Install Chaos Mesh
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

```yaml
# Kill 33% of pods in production namespace
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-experiment
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: fixed-percent
  value: "33"
  selector:
    namespaces:
      - production
    labelSelectors:
      "app": "my-api"
  scheduler:
    cron: "@once"
```

```yaml
# Inject 200ms network latency on all pods
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-latency-experiment
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
  delay:
    latency: "200ms"
    correlation: "100"
    jitter: "0ms"
  duration: "5m"
```

```yaml
# CPU stress — 80% utilization for 2 minutes
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-experiment
spec:
  mode: one
  selector:
    namespaces: [production]
  stressors:
    cpu:
      workers: 4
      load: 80
  duration: "2m"
```

### Manual chaos techniques

```bash
# Kill a random pod
kubectl -n production delete pod \
  $(kubectl -n production get pods -l app=my-api -o name | shuf -n 1)

# Drain a node (simulate node failure)
kubectl drain node-03 --ignore-daemonsets --delete-emptydir-data --grace-period=30

# Simulate network partition (tc - traffic control)
tc qdisc add dev eth0 root netem delay 500ms 100ms loss 5%
tc qdisc show dev eth0
# Remove:
tc qdisc del dev eth0 root

# Fill disk space (test disk full handling)
dd if=/dev/zero of=/tmp/bigfile bs=1M count=10240 &
# Remove:
rm /tmp/bigfile

# Exhaust file descriptors
ulimit -n 10  # Then run application — it will fail to open sockets/files
```

[↑ Back to TOC](#table-of-contents)

---

## Cloud HA Services

### AWS

| Service | HA Feature | Notes |
|---------|-----------|-------|
| **EC2** | Multi-AZ + Auto Scaling Groups | Use ASG min=2 across 2+ AZs |
| **RDS** | Multi-AZ standby, Read Replicas | Automatic failover ~60s |
| **Aurora** | Multi-AZ, 6 copies of data | Failover ~30s, Aurora Global 1-5s |
| **ElastiCache** | Multi-AZ Redis cluster | Auto-failover |
| **S3** | 11 nines durability | Built-in, no action needed |
| **ELB** | Cross-zone load balancing | Use ALB for HTTP, NLB for TCP |
| **Route 53** | Health-check based failover | 60s TTL minimum |
| **EKS** | Multi-AZ node groups | Spread pods with topologyKey |

### Azure

| Service | HA Feature |
|---------|-----------|
| **VM Scale Sets** | Availability Zones |
| **Azure SQL** | Business Critical tier — 3 replicas |
| **Azure Cache for Redis** | Zone-redundant, geo-replication |
| **Traffic Manager** | DNS-based global routing |
| **Azure Front Door** | Global CDN + failover |
| **AKS** | Multi-zone node pools |

### GCP

| Service | HA Feature |
|---------|-----------|
| **MIG (Managed Instance Group)** | Multi-zone, auto-healing |
| **Cloud SQL** | HA with failover replica |
| **Cloud Spanner** | Multi-region, 99.999% SLA |
| **Cloud Load Balancing** | Global, anycast |
| **GKE** | Regional cluster (3 AZs) |

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

```bash
# HAProxy
haproxy -f /etc/haproxy/haproxy.cfg -c           # Validate config
systemctl reload haproxy                          # Graceful reload
echo "show info" | socat stdio /var/lib/haproxy/stats
echo "show servers state" | socat stdio /var/lib/haproxy/stats

# Keepalived
systemctl status keepalived
ip addr show eth0 | grep 192.168.1.100            # Check if VIP is local
journalctl -u keepalived -f                       # Watch failover events

# MySQL Replication (MySQL 8.0.22+ syntax)
SHOW BINARY LOG STATUS\G               # On primary (was: SHOW MASTER STATUS)
SHOW REPLICA STATUS\G                  # On replica (was: SHOW SLAVE STATUS)
SHOW REPLICAS\G                        # List replicas (was: SHOW SLAVE HOSTS)
STOP REPLICA; START REPLICA;           # Control replica thread
RESET REPLICA ALL;                     # Remove replication config

# Galera
SHOW STATUS LIKE 'wsrep_cluster_size';
SHOW STATUS LIKE 'wsrep_local_state_comment';
SHOW STATUS LIKE 'wsrep_connected';

# etcd
ETCDCTL_API=3 etcdctl member list
ETCDCTL_API=3 etcdctl endpoint health
ETCDCTL_API=3 etcdctl snapshot status snapshot.db

# Velero
velero backup get
velero backup describe <name>
velero restore create --from-backup <name>
velero schedule get

# Kubernetes HA
kubectl get nodes -o wide
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
kubectl get pdb -A
kubectl get poddisruptionbudget -n production
```

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 1 — HAProxy Round-Robin Load Balancing (Beginner)

**Goal**: Set up HAProxy fronting three simple web servers and verify failover.

```bash
# Start 3 backend servers using Python HTTP server
for port in 8001 8002 8003; do
  mkdir -p /tmp/server${port}
  echo "Server on port ${port}" > /tmp/server${port}/index.html
  python3 -m http.server ${port} --directory /tmp/server${port} &
done

# Install and configure HAProxy
dnf install haproxy -y

# Configure /etc/haproxy/haproxy.cfg with 3 backends on 8001-8003
# Run: while true; do curl -s http://localhost/; sleep 0.5; done
# Kill server on 8001 and observe traffic moving to 8002 and 8003
# Re-start server on 8001 and observe it return to the pool
```

---

### Lab 2 — MySQL Primary-Replica Replication (Intermediate)

**Goal**: Set up MySQL primary-replica replication and simulate failover.

```bash
# Use Docker/Podman to run 2 MySQL instances
docker run -d --name mysql-primary \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -p 3306:3306 mysql:8.0

docker run -d --name mysql-replica \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -p 3307:3306 mysql:8.0

# Follow the primary-replica setup steps from this module
# Then:
# 1. Insert data into primary
# 2. Verify it appears on replica (SHOW REPLICA STATUS\G)
# 3. Stop primary container
# 4. Promote replica: STOP REPLICA; RESET REPLICA ALL; SET GLOBAL read_only=0;
# 5. Write to the (now primary) replica and verify
```

---

### Lab 3 — Kubernetes PodDisruptionBudget (Intermediate)

```bash
# Deploy an app with 3 replicas
kubectl create deployment lab-app --image=nginx:alpine --replicas=3 -n default

# Create a PDB allowing max 1 unavailable
kubectl apply -f - << 'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: lab-app-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: lab-app
EOF

# Try to drain a node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# Observe: it waits until evictions are compliant with PDB

# Check PDB status
kubectl get pdb lab-app-pdb
```

---

### Lab 4 — etcd Backup and Restore (Intermediate)

```bash
# On a kubeadm cluster control plane node

# Create some test data
kubectl create namespace backup-test
kubectl create configmap test-data --from-literal=key=value -n backup-test

# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table

# Delete the namespace
kubectl delete namespace backup-test

# Restore etcd from backup
# (Follow the restore procedure from this module)
# Verify the namespace returns
kubectl get namespace backup-test
```

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [AWS Well-Architected Framework — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/)
- [Google SRE Book — Managing Disasters](https://sre.google/sre-book/managing-disasters/)
- [Chaos Engineering by Casey Rosenthal & Nora Jones](https://www.oreilly.com/library/view/chaos-engineering/9781491988459/)
- [Chaos Mesh Documentation](https://chaos-mesh.org/docs/)
- [Patroni Documentation](https://patroni.readthedocs.io/)
- [Velero Documentation](https://velero.io/docs/)
- [HAProxy Configuration Manual](https://www.haproxy.org/download/2.8/doc/configuration.txt)
- [Kubernetes PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [AWS DR Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [Site Reliability Engineering — Postmortems](https://sre.google/sre-book/postmortem-culture/)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [Creative Commons BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Non-commercial use only. Share alike with attribution.*
