# Database Migration: Neon DB to AWS RDS PostgreSQL

## Overview

This document outlines the migration process used to transfer a small database from Neon DB to AWS RDS PostgreSQL. The migration utilized a dump/restore approach with the dump file hosted on a temporary web server for remote access.

## Prerequisites

- PostgreSQL client tools installed (`psql`, `pg_dump`)
- Access credentials to both source (Neon DB) and target (AWS RDS) databases
- Network connectivity to both databases

## Migration Steps

### 1. Database Dump Creation

A database dump was created from the source Neon DB using PostgreSQL's `pg_dump` utility:

```bash
pg_dump -Fc -U [neon_username] -h [neon_host] -d [database_name] -f neon_dump.dump
```

Replace the placeholders with your actual Neon DB credentials:
- `[neon_username]`: Your Neon database username
- `[neon_host]`: Your Neon database host address
- `[database_name]`: Name of your Neon database

### 2. Dump File Hosting

The dump file was hosted on a temporary web server at:
```
https://hostrestore.onrender.com/neon_dump.dump
```

This allowed AWS RDS to access the file for restoration.

### 3. Download Dump File to Restoration Environment

Before restoration, download the dump file to your local environment or EC2 instance:

```bash
# Download the dump file using curl
curl -o neon_dump.dump https://hostrestore.onrender.com/neon_dump.dump

# Or using wget
wget https://hostrestore.onrender.com/neon_dump.dump
```

### 4. Database Restoration

The database was restored to the target AWS RDS PostgreSQL instance using the `pg_restore` command:

```bash
# Method 1: Using pg_restore with connection string
pg_restore -v \
  --dbname="postgresql://hablistpostgres:Codercoder99@hablistdbdev.cex82ommkvyk.us-east-1.rds.amazonaws.com:5432/hablistdbdev?sslmode=require" \
  --no-owner \
  --no-privileges \
  neon_dump.dump

# Method 2: Using separate parameters (more readable)
pg_restore -v \
  --host=hablistdbdev.cex82ommkvyk.us-east-1.rds.amazonaws.com \
  --port=5432 \
  --username=hablistpostgres \
  --dbname=hablistdbdev \
  --no-owner \
  --no-privileges \
  --clean \
  --if-exists \
  neon_dump.dump
```

You will be prompted to enter the password: `Codercoder99`

### 5. Alternative: Using psql for SQL Script Restoration

If you created a plain SQL dump instead of a custom format dump, use this approach:

```bash
# First, create the target database if it doesn't exist
psql "postgresql://hablistpostgres:Codercoder99@hablistdbdev.cex82ommkvyk.us-east-1.rds.amazonaws.com:5432/postgres?sslmode=require" \
  -c "CREATE DATABASE hablistdbdev;"

# Then restore the SQL dump
psql "postgresql://hablistpostgres:Codercoder99@hablistdbdev.cex82ommkvyk.us-east-1.rds.amazonaws.com:5432/hablistdbdev?sslmode=require" \
  -f neon_dump.sql
```

### 6. Verification Commands

After migration, verify the restoration with these commands:

```bash
# Connect to the database and check tables
psql "postgresql://hablistpostgres:Codercoder99@hablistdbdev.cex82ommkvyk.us-east-1.rds.amazonaws.com:5432/hablistdbdev?sslmode=require" \
  -c "\dt"

# Check database size
psql "postgresql://hablistpostgres:Codercoder99@hablistdbdev.cex82ommkvyk.us-east-1.rds.amazonaws.com:5432/hablistdbdev?sslmode=require" \
  -c "SELECT pg_size_pretty(pg_database_size('hablistdbdev'));"

# Count records in each table
psql "postgresql://hablistpostgres:Codercoder99@hablistdbdev.cex82ommkvyk.us-east-1.rds.amazonaws.com:5432/hablistdbdev?sslmode=require" \
  -c "SELECT schemaname, relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;"
```

## Connection Details

**Target Database:** AWS RDS PostgreSQL
- **Host:** hablistdbdev.cex82ommkvyk.us-east-1.rds.amazonaws.com
- **Port:** 5432
- **Database:** hablistdbdev
- **Username:** hablistpostgres
- **Password:** Codercoder99
- **SSL Mode:** require

## Important Notes

1. The `--no-owner` and `--no-privileges` flags are used to avoid permission issues between different database environments
2. The `--clean` and `--if-exists` flags help with idempotent restoration (can be run multiple times)
3. For large databases, consider adding the `--jobs` flag to `pg_restore` to parallelize the restoration

## Security Notes

1. The database credentials have been included in this documentation for reference but should be rotated after migration completion
2. The temporary web server hosting the dump file should be taken down after successful migration
3. Ensure proper network security groups are configured to restrict access to the RDS instance

## Post-Migration Tasks

- [ ] Rotate database credentials
- [ ] Remove the temporary web server hosting the dump file
- [ ] Update application connection strings
- [ ] Perform thorough testing of the application with the new database
- [ ] Set up monitoring and alerts for the RDS instance
- [ ] Establish backup procedures for the RDS instance

This migration approach was suitable for a small database. For larger databases, consider using AWS Database Migration Service or alternative methods that minimize downtime.
