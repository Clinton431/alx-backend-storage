# MySQL Advanced Guide with Examples

## Overview
This comprehensive guide covers advanced MySQL concepts with practical examples. Designed for database administrators, developers, and architects, it provides actionable techniques to optimize MySQL database systems for performance, scalability, and enterprise-level management.

## Table of Contents
- [Performance Optimization](#performance-optimization)
- [Advanced Indexing Strategies](#advanced-indexing-strategies)
- [Query Optimization Techniques](#query-optimization-techniques)
- [Transaction Management](#transaction-management)
- [Replication and High Availability](#replication-and-high-availability)
- [Partitioning](#partitioning)
- [Security Best Practices](#security-best-practices)
- [Backup and Recovery Strategies](#backup-and-recovery-strategies)
- [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)
- [Advanced Configuration](#advanced-configuration)

## Performance Optimization

### Buffer Pool Configuration
```sql
-- Set InnoDB buffer pool size (75% of available memory for dedicated DB servers)
SET GLOBAL innodb_buffer_pool_size = 8589934592; -- 8GB in bytes

-- Configure multiple buffer pool instances for better concurrency
SET GLOBAL innodb_buffer_pool_instances = 8;
```

### Query Cache Optimization
```sql
-- Enable query cache (MySQL 5.7 and earlier)
SET GLOBAL query_cache_type = 1;
SET GLOBAL query_cache_size = 268435456; -- 256MB

-- Check query cache efficiency
SHOW STATUS LIKE 'Qcache%';
```

### I/O Optimization
```sql
-- Use O_DIRECT to bypass the OS cache
SET GLOBAL innodb_flush_method = 'O_DIRECT';

-- Configure read-ahead
SET GLOBAL innodb_read_ahead_threshold = 56;

-- Adjust I/O capacity
SET GLOBAL innodb_io_capacity = 2000;
SET GLOBAL innodb_io_capacity_max = 4000;
```

### Thread Pool Management
```sql
-- Install thread pool plugin (Enterprise Edition)
INSTALL PLUGIN thread_pool SONAME 'thread_pool.so';

-- Configure thread pool settings
SET GLOBAL thread_pool_size = 16;
SET GLOBAL thread_pool_stall_limit = 100;
```

## Advanced Indexing Strategies

### Composite Indexes
```sql
-- Create a composite index for queries filtering by both columns
CREATE INDEX idx_user_status_created ON users(status, created_at);

-- For queries with range conditions
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
```

### Covering Indexes
```sql
-- Index includes all columns needed by the query (no table access required)
CREATE INDEX idx_products_covering ON products(category_id, price, name, stock);

-- Query that can use the covering index
SELECT name, price FROM products WHERE category_id = 5 AND price > 100;
```

### Partial Indexes
```sql
-- Index only active records (MySQL 8.0+)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- Create a functional index (MySQL 8.0+)
CREATE INDEX idx_lower_email ON users((LOWER(email)));
```

### Index Maintenance
```sql
-- Analyze table to update index statistics
ANALYZE TABLE orders;

-- Optimize table to rebuild indexes
OPTIMIZE TABLE orders;

-- Check index usage
SELECT 
    INDEX_NAME, COUNT_STAR, COUNT_READ, COUNT_FETCH 
FROM 
    performance_schema.table_io_waits_summary_by_index_usage 
WHERE 
    OBJECT_SCHEMA = 'database_name' AND OBJECT_NAME = 'table_name';
```

## Query Optimization Techniques

### EXPLAIN Plan Analysis
```sql
-- Basic EXPLAIN
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- Detailed EXPLAIN with JSON format (MySQL 5.6+)
EXPLAIN FORMAT=JSON SELECT 
    c.name, SUM(oi.quantity * oi.unit_price) AS total
FROM 
    customers c
    JOIN orders o ON c.id = o.customer_id
    JOIN order_items oi ON o.id = oi.order_id
WHERE 
    o.order_date BETWEEN '2023-01-01' AND '2023-12-31'
GROUP BY 
    c.name
HAVING 
    total > 1000
ORDER BY 
    total DESC;
```

### Subquery Optimization
```sql
-- Rewrite subquery as JOIN (often more efficient)
-- Instead of:
SELECT * FROM products WHERE category_id IN (SELECT id FROM categories WHERE active = 1);

-- Use:
SELECT p.* 
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE c.active = 1;

-- Use EXISTS for existence checks
SELECT c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.order_date > '2023-01-01');
```

### JOIN Optimization
```sql
-- Use proper JOIN types (INNER, LEFT, RIGHT)
-- Order tables from smallest to largest when possible
SELECT 
    c.name, o.order_date, p.name AS product_name
FROM 
    small_table s
    JOIN medium_table m ON s.id = m.small_id
    JOIN large_table l ON m.id = l.medium_id;

-- Avoid cartesian joins
SELECT /*+ JOIN_ORDER(c, o, oi) */ 
    c.name, o.order_id
FROM 
    customers c
    JOIN orders o ON c.id = o.customer_id
    JOIN order_items oi ON o.id = oi.order_id;
```

### Common Table Expressions (CTEs)
```sql
-- Use CTEs for better readability and optimization
WITH customer_totals AS (
    SELECT 
        customer_id, 
        SUM(total_amount) AS total_spent
    FROM 
        orders
    WHERE 
        order_date > '2023-01-01'
    GROUP BY 
        customer_id
),
high_value AS (
    SELECT 
        customer_id
    FROM 
        customer_totals
    WHERE 
        total_spent > 10000
)
SELECT 
    c.name, c.email, ct.total_spent
FROM 
    customers c
    JOIN customer_totals ct ON c.id = ct.customer_id
    JOIN high_value hv ON c.id = hv.customer_id;
```

## Transaction Management

### Isolation Levels
```sql
-- Set transaction isolation level
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Check current isolation level
SELECT @@transaction_isolation;
```

### Deadlock Prevention
```sql
-- Set lock wait timeout
SET innodb_lock_wait_timeout = 50;

-- Enable deadlock detection
SET GLOBAL innodb_deadlock_detect = ON;

-- Consistent order of accessing tables
START TRANSACTION;
SELECT * FROM table_a WHERE id = 1 FOR UPDATE;
SELECT * FROM table_b WHERE id = 2 FOR UPDATE;
-- Update operations
COMMIT;

-- Show deadlock information
SHOW ENGINE INNODB STATUS;
```

### Savepoints
```sql
-- Using savepoints for partial rollbacks
START TRANSACTION;
INSERT INTO orders (customer_id, order_date) VALUES (123, NOW());
SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 456, 1);
-- If something goes wrong with order items
ROLLBACK TO SAVEPOINT after_order;
-- Continue with other operations
COMMIT;
```

## Replication and High Availability

### Master-Slave Replication Setup
```sql
-- On Master Server
CREATE USER 'replication_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';

-- Get master status
SHOW MASTER STATUS;

-- On Slave Server
CHANGE MASTER TO
    MASTER_HOST='master_ip',
    MASTER_USER='replication_user',
    MASTER_PASSWORD='password',
    MASTER_LOG_FILE='recorded_log_file',
    MASTER_LOG_POS=recorded_log_position;

START SLAVE;

-- Check slave status
SHOW SLAVE STATUS\G
```

### Group Replication (MySQL 8.0+)
```sql
-- On all servers in my.cnf
[mysqld]
server_id=1 # unique for each server
gtid_mode=ON
enforce_gtid_consistency=ON
binlog_checksum=NONE
plugin_load_add='group_replication.so'
group_replication_group_name="aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
group_replication_start_on_boot=off
group_replication_local_address="server_ip:33061"
group_replication_group_seeds="server1_ip:33061,server2_ip:33061,server3_ip:33061"
group_replication_bootstrap_group=off

-- On first node to bootstrap
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;

-- On other nodes
START GROUP_REPLICATION;

-- Check status
SELECT * FROM performance_schema.replication_group_members;
```

### Automatic Failover with MySQL Router
```bash
# Install MySQL Router
sudo apt-get install mysql-router

# Bootstrap router with InnoDB Cluster
mysqlrouter --bootstrap root@primary_node:3306 --user=mysqlrouter

# Router configuration (mysqlrouter.conf)
[routing:primary]
bind_address=0.0.0.0
bind_port=6446
destinations=metadata-cache://default/?role=PRIMARY
routing_strategy=first-available

[routing:secondary]
bind_address=0.0.0.0
bind_port=6447
destinations=metadata-cache://default/?role=SECONDARY
routing_strategy=round-robin
```

## Partitioning

### Range Partitioning
```sql
-- Create a table partitioned by date ranges
CREATE TABLE orders (
    id INT NOT NULL,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10,2),
    PRIMARY KEY (id, order_date)
)
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Query targeting specific partition
EXPLAIN SELECT * FROM orders WHERE order_date BETWEEN '2022-01-01' AND '2022-12-31';
```

### List Partitioning
```sql
-- Create a table partitioned by region
CREATE TABLE sales (
    id INT NOT NULL,
    country VARCHAR(2) NOT NULL,
    sale_date DATE NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (id, country)
)
PARTITION BY LIST (country) (
    PARTITION p_north_america VALUES IN ('US', 'CA', 'MX'),
    PARTITION p_europe VALUES IN ('UK', 'FR', 'DE', 'IT', 'ES'),
    PARTITION p_asia VALUES IN ('CN', 'JP', 'IN', 'KR'),
    PARTITION p_other VALUES IN ('AU', 'BR', 'ZA')
);
```

### Hash Partitioning
```sql
-- Create a table with hash partitioning
CREATE TABLE users (
    id INT NOT NULL,
    username VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id)
)
PARTITION BY HASH (id)
PARTITIONS 8;
```

### Partition Management
```sql
-- Add a new partition
ALTER TABLE orders ADD PARTITION (PARTITION p2025 VALUES LESS THAN (2026));

-- Remove a partition (data will be deleted)
ALTER TABLE orders DROP PARTITION p2020;

-- Reorganize partitions
ALTER TABLE orders REORGANIZE PARTITION p_future INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Check partition information
SELECT 
    TABLE_SCHEMA, TABLE_NAME, PARTITION_NAME, PARTITION_ORDINAL_POSITION,
    PARTITION_METHOD, PARTITION_EXPRESSION, TABLE_ROWS
FROM 
    information_schema.PARTITIONS
WHERE 
    TABLE_NAME = 'orders';
```

## Security Best Practices

### Role-Based Access Control (MySQL 8.0+)
```sql
-- Create roles
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- Grant privileges to roles
GRANT SELECT ON myapp.* TO 'app_read';
GRANT SELECT, INSERT, UPDATE ON myapp.* TO 'app_write';
GRANT ALL PRIVILEGES ON myapp.* TO 'app_admin';

-- Create users and assign roles
CREATE USER 'reader'@'%' IDENTIFIED BY 'password';
CREATE USER 'writer'@'%' IDENTIFIED BY 'password';
CREATE USER 'admin'@'%' IDENTIFIED BY 'password';

GRANT 'app_read' TO 'reader'@'%';
GRANT 'app_read', 'app_write' TO 'writer'@'%';
GRANT 'app_admin' TO 'admin'@'%';

-- Set default roles
SET DEFAULT ROLE ALL TO 
    'reader'@'%', 
    'writer'@'%', 
    'admin'@'%';
```

### Encryption at Rest
```sql
-- Enable table encryption
SET GLOBAL table_encryption_privilege_check=ON;

-- Create encrypted tablespace
CREATE TABLESPACE encrypted_ts 
    ADD DATAFILE 'encrypted_ts.ibd' 
    ENCRYPTION='Y';

-- Create encrypted table
CREATE TABLE sensitive_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ssn VARCHAR(16) NOT NULL,
    dob DATE NOT NULL
) TABLESPACE encrypted_ts;

-- Encrypt existing tables
ALTER TABLE users ENCRYPTION='Y';
```

### Audit Logging
```sql
-- Enable audit log plugin (Enterprise Edition)
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- Configure audit log
SET GLOBAL audit_log_format = 'JSON';
SET GLOBAL audit_log_policy = 'ALL';
SET GLOBAL audit_log_connection_policy = 'ALL';
SET GLOBAL audit_log_statement_policy = 'ALL';

-- Create audit filter (MySQL 8.0 Enterprise)
SELECT audit_log_filter_set_filter('log_connections', '{ "filter": { "connection": { "any": true } } }');
SELECT audit_log_filter_set_user('%', 'log_connections');
```

### SSL Configuration
```sql
-- Require SSL for specific user
CREATE USER 'secure_user'@'%' IDENTIFIED BY 'password' REQUIRE SSL;
ALTER USER 'existing_user'@'%' REQUIRE SSL;

-- Configure MySQL server for SSL in my.cnf
[mysqld]
ssl_ca=ca.pem
ssl_cert=server-cert.pem
ssl_key=server-key.pem
```

## Backup and Recovery Strategies

### Physical Backup with mysqldump
```bash
# Full backup
mysqldump --all-databases --single-transaction --triggers --routines --events > full_backup.sql

# Backup specific databases
mysqldump --databases db1 db2 --single-transaction > databases_backup.sql

# Compressed backup
mysqldump --all-databases | gzip > backup.sql.gz

# With progress and parallel processing
mysqldump --all-databases --single-transaction | pv | gzip > backup.sql.gz
```

### Binary Log Backup for Point-in-Time Recovery
```bash
# Enable binary logging in my.cnf
[mysqld]
log_bin = mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800 # 7 days

# Extract binary logs for point-in-time recovery
mysqlbinlog --start-datetime="2023-04-15 12:00:00" --stop-datetime="2023-04-15 13:00:00" \
    /var/lib/mysql/mysql-bin.000001 > recovery.sql
```

### MySQL Enterprise Backup
```bash
# Full backup
mysqlbackup --user=root --password --backup-dir=/backup \
    --with-timestamp --compress backup-and-apply-log

# Incremental backup
mysqlbackup --user=root --password --backup-dir=/backup/incr \
    --incremental --incremental-base=dir:/backup/last_full \
    --with-timestamp backup-and-apply-log
```

### Database Cloning (MySQL 8.0+)
```sql
-- Clone plugin installation
INSTALL PLUGIN clone SONAME 'mysql_clone.so';

-- Create clone user
CREATE USER clone_user@localhost IDENTIFIED BY 'password';
GRANT BACKUP_ADMIN ON *.* TO clone_user@localhost;

-- Clone from donor to recipient
CLONE INSTANCE FROM 'clone_user'@'donor_host':3306 
IDENTIFIED BY 'password';
```

## Monitoring and Troubleshooting

### Performance Schema Monitoring
```sql
-- Enable Performance Schema
SET GLOBAL performance_schema = ON;

-- Monitor query execution
SELECT 
    DIGEST_TEXT, 
    COUNT_STAR, 
    AVG_TIMER_WAIT/1000000000 as avg_execution_time_ms,
    SUM_ROWS_SENT, 
    SUM_ROWS_EXAMINED,
    FIRST_SEEN,
    LAST_SEEN
FROM 
    performance_schema.events_statements_summary_by_digest
ORDER BY 
    avg_execution_time_ms DESC
LIMIT 10;

-- Monitor table I/O
SELECT 
    object_schema,
    object_name,
    count_read,
    count_write,
    count_fetch,
    count_insert,
    count_update,
    count_delete
FROM 
    performance_schema.table_io_waits_summary_by_table
ORDER BY 
    sum_timer_wait DESC
LIMIT 10;
```

### Slow Query Log Analysis
```sql
-- Enable slow query log in my.cnf
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
log_queries_not_using_indexes = 1

-- Analyze slow query log with pt-query-digest
pt-query-digest /var/log/mysql/mysql-slow.log > slow_query_analysis.txt
```

### InnoDB Metrics Monitoring
```sql
-- Enable InnoDB metrics
SET GLOBAL innodb_monitor_enable = 'all';

-- View buffer pool stats
SELECT * FROM information_schema.INNODB_BUFFER_PAGE_LRU LIMIT 10;

-- Check buffer pool hit ratio
SELECT 
    (1 - (
        SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
    ) / (
        SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
    )) * 100 AS buffer_pool_hit_ratio;
```

### Sys Schema Diagnostics
```sql
-- Top 10 queries by total execution time
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;

-- Tables with full table scans
SELECT * FROM sys.schema_tables_with_full_table_scans ORDER BY rows_full_scanned DESC LIMIT 10;

-- Find unused indexes
SELECT * FROM sys.schema_unused_indexes;

-- Find redundant indexes
SELECT * FROM sys.schema_redundant_indexes;
```

## Advanced Configuration

### Multi-Tenant Architecture
```sql
-- Schema-based multi-tenancy
CREATE SCHEMA tenant1;
CREATE SCHEMA tenant2;

-- Table-based multi-tenancy with tenant_id column
CREATE TABLE shared_customers (
    id INT AUTO_INCREMENT,
    tenant_id INT NOT NULL,
    name VARCHAR(255),
    email VARCHAR(255),
    PRIMARY KEY (id),
    INDEX idx_tenant (tenant_id)
);

-- Row-based security policy (MySQL 8.0+)
CREATE ROLE tenant1_role, tenant2_role;

CREATE DEFINER = 'root'@'localhost' 
FUNCTION tenant_id() RETURNS INT
DETERMINISTIC
RETURN (SELECT @tenant_id);

GRANT EXECUTE ON FUNCTION tenant_id TO tenant1_role, tenant2_role;

CREATE DEFINER = 'root'@'localhost'
TRIGGER ins_tenant_data BEFORE INSERT ON shared_customers
FOR EACH ROW
SET NEW.tenant_id = tenant_id();
```

### Connection Pooling Configuration
```ini
# ProxySQL configuration
mysql_servers =
(
    {
        address="mysql1.example.com"
        port=3306
        hostgroup=0
        max_connections=200
    },
    {
        address="mysql2.example.com"
        port=3306
        hostgroup=1
        max_connections=200
    }
)

mysql_users =
(
    {
        username="appuser"
        password="password"
        default_hostgroup=0
        max_connections=1000
        transaction_persistent=1
    }
)

mysql_query_rules =
(
    {
        rule_id=1
        active=1
        match_pattern="^SELECT"
        destination_hostgroup=1
        apply=1
    },
    {
        rule_id=2
        active=1
        match_pattern=".*"
        destination_hostgroup=0
        apply=1
    }
)
```

### Geographical Data Processing
```sql
-- Create spatial data table
CREATE TABLE stores (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    location POINT NOT NULL,
    SPATIAL INDEX(location)
);

-- Insert location data
INSERT INTO stores (name, location) 
VALUES ('Downtown Store', ST_GeomFromText('POINT(40.7128 -74.0060)'));

-- Find stores within 10km of a point
SELECT 
    id, name, 
    ST_Distance_Sphere(
        location, 
        ST_GeomFromText('POINT(40.7300 -73.9950)')
    ) / 1000 AS distance_km
FROM 
    stores
WHERE 
    ST_Distance_Sphere(
        location, 
        ST_GeomFromText('POINT(40.7300 -73.9950)')
    ) <= 10000 -- 10km in meters
ORDER BY 
    distance_km;
```

### Large Scale Architecture Settings
```ini
# My.cnf configuration for high traffic servers
[mysqld]
# Memory settings
innodb_buffer_pool_size = 64G
innodb_buffer_pool_instances = 16
innodb_log_buffer_size = 64M
key_buffer_size = 256M
max_connections = 2000
thread_cache_size = 100

# Storage settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
innodb_io_capacity = 5000
innodb_io_capacity_max = 10000

# Performance settings
join_buffer_size = 4M
sort_buffer_size = 4M
read_rnd_buffer_size = 2M
read_buffer_size = 2M
max_heap_table_size = 256M
tmp_table_size = 256M

# Replication settings
binlog_format = ROW
sync_binlog = 0
slave_parallel_workers = 16
slave_parallel_type = LOGICAL_CLOCK
```

### Monitoring System Integration
```yaml
# Prometheus MySQL Exporter configuration
scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):\d+'
        replacement: '${1}'

# Grafana dashboard example
{
  "dashboard": {
    "title": "MySQL Overview",
    "panels": [
      {
        "title": "MySQL Connections",
        "targets": [
          {
            "expr": "mysql_global_status_threads_connected",
            "legendFormat": "Connected"
          },
          {
            "expr": "mysql_global_variables_max_connections",
            "legendFormat": "Max Connections"
          }
        ]
      },
      {
        "title": "MySQL Query Rate",
        "targets": [
          {
            "expr": "rate(mysql_global_status_queries[5m])",
            "legendFormat": "Queries per second"
          }
        ]
      }
    ]
  }
}
```

This advanced MySQL guide provides you with practical examples and implementations for each major topic. Apply these techniques to optimize your database infrastructure, improve performance, and manage enterprise-scale MySQL deployments effectively.
