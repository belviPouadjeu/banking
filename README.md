# Internet Banking V3 - Complete Docker Deployment Guide

**Version:** 3.0  
**Date:** February 2026  
**Document Type:** Complete Deployment & Operations Manual  
**Target Audience:** IT Infrastructure Teams, DevOps Engineers, System Administrators

---

## Document Overview

This comprehensive guide consolidates all documentation for deploying Internet Banking V3 using a fully containerized Docker architecture. It replaces legacy JAR-based deployment with modern container orchestration.

**Total Documentation Coverage:** ~95 pages consolidated into one reference

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Requirements](#2-system-requirements)
3. [Architecture Overview](#3-architecture-overview)
4. [Domain Configuration](#4-domain-configuration)
5. [Prerequisites & Software Setup](#5-prerequisites--software-setup)
6. [Quick Start (15-Minute Deployment)](#6-quick-start-15-minute-deployment)
7. [Detailed Installation Guide](#7-detailed-installation-guide)
8. [Configuration Reference](#8-configuration-reference)
9. [Service Catalog](#9-service-catalog)
10. [Operations & Maintenance](#10-operations--maintenance)
11. [Security Hardening](#11-security-hardening)
12. [Monitoring & Logging](#12-monitoring--logging)
13. [Backup & Recovery](#13-backup--recovery)
14. [Troubleshooting](#14-troubleshooting)
15. [Migration from Legacy](#15-migration-from-legacy)
16. [Production Deployment Checklist](#16-production-deployment-checklist)
17. [Appendices](#17-appendices)

---

## 1. Introduction

### 1.1 Purpose

This document provides comprehensive instructions for deploying the Internet Banking V3 application using Docker containerization. It is designed for IT Infrastructure Teams, DevOps Engineers, and System Administrators at banking institutions.

### 1.2 Docker-Based Architecture Benefits

**Transformation Summary:**

| Aspect | Legacy (JAR/systemd) | Modern (Docker) |
|--------|---------------------|-----------------|
| Deployment | Manual JAR files, systemd services | Containerized orchestration |
| Configuration | OS-level files in /etc/ | Environment variables |
| Scalability | Vertical only | Horizontal + Vertical |
| Portability | OS-dependent | Platform-independent |
| Rollback | Manual, risky | Instant versioning |

### 1.3 What's Included

- **15 Containerized Services** (Infrastructure, Backend, Frontend)
- **Complete Configuration Templates** (80+ environment variables)
- **Production-Ready Docker Compose Files**
- **Automated Operational Scripts**
- **NGINX Reverse Proxy with SSL**
- **Database Schema Migration Tools**

---

## 2. System Requirements

### 2.1 Minimum Specifications (Testing/Light Load)

| Resource | Minimum Specification |
|----------|---------------------|
| **RAM** | 32 GB |
| **CPU** | 8 vCPU |
| **Disk** | 256 GB SSD |
| **Network** | Public IP Address |
| **OS** | Ubuntu 24.04 / 22.04 LTS |

⚠️ **Warning**: Production environments require significantly higher resources.

### 2.2 Recommended Production Setup

**Three-Server Architecture:**

1. **Backend Server**: 16 vCPU, 64 GB RAM, 512 GB NVMe SSD
2. **Frontend Server**: 8 vCPU, 16 GB RAM, 256 GB SSD  
3. **Database Server**: 16 vCPU, 64 GB RAM, 1 TB NVMe SSD

**Single Server (Dev/Staging)**: 32 vCPU, 128 GB RAM, 1 TB SSD

### 2.3 Software Requirements

| Software | Version | Required |
|----------|---------|----------|
| Docker Engine | ≥ 24.0 | ✅ Yes |
| Docker Compose | ≥ 2.20 | ✅ Yes |
| Linux OS | Ubuntu 24.04/22.04 | ✅ Yes |
| SSL Certificates | Valid certs | ✅ Production |

---

## 3. Architecture Overview

### 3.1 Service Architecture

**15 Containerized Services:**

**Infrastructure (3):**
- PostgreSQL 16 (ib-postgres:5432)
- Redis 7 (ib-redis:6379)
- MinIO (ib-minio:9000)

**Backend (7):**
- Gateway (ib-gateway-service:8080)
- Auth (ib-auth-service:8082)
- User (ib-user-service:8084)
- Payment (ib-payment-service:8083)
- Transaction (ib-transaction-service:8081)
- ABS Adapter (ib-abs-adapter-service:8085)
- Blobpath (ib-blobpath-service:8086)
- Settings (ib-settings-service:8888)

**Frontend (2):**
- Corporate Client (ib-fe-corporate-client:3002)
- Main BackOffice (ib-fe-main-bo-client:3004)

**Support (3):**
- Schema Migration (ib-schema-migration)
- NGINX Proxy (ib-nginx-proxy:80/443)

### 3.2 Network Architecture

All services communicate through Docker bridge network: `ib-network`

### 3.3 Data Persistence

**Docker Volumes:**
- `postgres-data` - Database storage (CRITICAL)
- `redis-data` - Cache storage
- `minio-data` - Object storage (CRITICAL)
- `blobpath-storage` - Document storage (CRITICAL)

---

## 4. Domain Configuration

### 4.1 Required DNS Records

| Subdomain | Purpose | Example |
|-----------|---------|---------|
| api.bankdomain.com | API Gateway | api.afrilandbank.com |
| corporate.bankdomain.com | Corporate UI | corporate.afrilandbank.com |
| mainbo.bankdomain.com | Admin BackOffice | mainbo.afrilandbank.com |

**DNS Configuration:**
```dns
api.bankdomain.com         A    YOUR_SERVER_IP
corporate.bankdomain.com   A    YOUR_SERVER_IP
mainbo.bankdomain.com      A    YOUR_SERVER_IP
```

### 4.2 SSL Certificates

**Option 1: Let's Encrypt (Free)**
```bash
sudo certbot --nginx -d api.bankdomain.com -d corporate.bankdomain.com -d mainbo.bankdomain.com
```

**Option 2: Commercial Certificates**
Place in `nginx/ssl/` directory

---

## 5. Prerequisites & Software Setup

### 5.1 Docker Engine Installation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker repository
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify installation
docker --version
docker compose version

# Enable Docker
sudo systemctl enable docker --now
```

### 5.2 Firewall Configuration

```bash
# Install and configure UFW
sudo apt install -y ufw
sudo ufw allow 22/tcp   # SSH
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw enable
```

### 5.3 System Optimization

```bash
# Increase file limits
echo "fs.file-max = 2097152" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Configure Docker logging
sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

---

## 6. Quick Start (15-Minute Deployment)

### 6.1 Prerequisites Checklist

- ✅ Docker 24.0+ installed
- ✅ Docker Compose 2.20+ installed  
- ✅ 32 GB RAM minimum
- ✅ 256 GB disk space
- ✅ Public IP configured

### 6.2 Clone or Extract Deployment Package

```bash
# If from Git repository
git clone https://github.com/your-org/internet-banking-v3.git
cd internet-banking-v3

# OR extract from archive
unzip internet-banking-v3.zip
cd internet-banking-v3
```

### 6.3 Configure Environment Variables

```bash
# Copy environment template
cp .env.example .env

# Edit with your values (REQUIRED)
nano .env
```

**Minimum Required Changes:**

```bash
# Database credentials
POSTGRES_PASSWORD=YourStrongPassword123!
DB_PASSWORD=YourStrongPassword123!

# Redis password
REDIS_PASSWORD=YourRedisPassword456!

# MinIO credentials
MINIO_ROOT_PASSWORD=YourMinIOPassword789!

# JWT secret (generate with: openssl rand -base64 32)
JWT_SECRET_KEY=YOUR_GENERATED_SECRET_KEY_HERE

# Core Banking System
ABS_URI=https://cbs.yourbank.com
ABS_SIGNATURE_KEY=YourCBSSignatureKey

# Bank details
BANK_NAME=Your Bank Name
CBS_DEFAULT_BANK_CODE=36035

# Admin credentials (change after first login!)
FIRST_SUPER_ADMIN_EMAIL=admin@yourbank.com
FIRST_SUPER_ADMIN_PASSWORD=ChangeMe123!QWE

# Frontend URLs
REACT_APP_API_BASE_URL=https://api.yourbank.com
REACT_APP_WEB_CORPORATE_BASE_URL=https://corporate.yourbank.com
VITE_API_BASE_URL=https://api.yourbank.com
```

### 6.4 Deploy All Services

```bash
# Make scripts executable
chmod +x *.sh

# Run deployment
./deploy.sh
```

**Deployment Process:**
1. Checks prerequisites
2. Builds Docker images (~10-15 minutes first time)
3. Starts infrastructure (PostgreSQL, Redis, MinIO)
4. Runs database migration
5. Starts backend services
6. Starts frontend services
7. Performs health checks

### 6.5 Verify Deployment

```bash
# Check all containers are running
docker compose ps

# Expected: 15 containers with status "Up"

# Test health endpoints
curl http://localhost:8080/actuator/health
curl http://localhost:8082/auth/actuator/health
curl http://localhost:3002  # Corporate frontend
curl http://localhost:3004  # BackOffice frontend
```

### 6.6 Access the Application

**Local Access (Development):**
- Corporate Frontend: http://localhost:3002
- BackOffice: http://localhost:3004
- API Gateway: http://localhost:8080

**Production Access (with DNS):**
- Corporate: https://corporate.yourbank.com
- BackOffice: https://mainbo.yourbank.com
- API: https://api.yourbank.com

**Default Login:**
- Email: admin@yourbank.com (from .env)
- Password: ChangeMe123!QWE (from .env)

⚠️ **IMPORTANT**: Change admin password immediately after first login!

---

## 7. Detailed Installation Guide

### 7.1 Project Structure

```
internet-banking-v3/
├── docker-compose.yml                    # Infrastructure services
├── docker-compose-backend.yml            # Backend services
├── docker-compose-backend-extended.yml   # Extended backend
├── docker-compose-frontend.yml           # Frontend services
├── .env.example                          # Environment template
├── deploy.sh                             # Deployment script
├── stop.sh                               # Stop all services
├── restart.sh                            # Restart services
├── logs.sh                               # View logs
├── backup.sh                             # Backup script
│
├── nginx/                                # NGINX configuration
│   ├── nginx.conf                        # Main config
│   └── conf.d/
│       ├── api.conf                      # API Gateway
│       ├── corporate.conf                # Corporate frontend
│       └── mainbo.conf                   # BackOffice
│
├── init-scripts/                         # Database initialization
│   └── 01-init-databases.sql
│
├── pkf_ib_db_schema/                     # Database migration service
├── pkf-ib-be-gateway-service/            # API Gateway
├── pkf-ib-be-auth-service/               # Authentication
├── pkf-ib-be-user-service/               # User management
├── pkf-ib-be-payment/                    # Payment processing
├── pkf-ib-be-transaction-service/        # Transactions
├── pkf-ib-be-abs-adapter-service/        # CBS adapter
├── pkf-ib-be-blobpath-service/           # File storage
├── pkf-ib-be-settings-service/           # Settings
├── pkf-ib-fe-corporate-client/           # Corporate UI (React)
└── pkf-ib-fe-main-bo-client/             # BackOffice UI (Vue)
```

### 7.2 Environment Configuration Deep Dive

#### 7.2.1 Database Configuration

```bash
# PostgreSQL settings
POSTGRES_USER=bank_client_ib_user
POSTGRES_PASSWORD=CHANGE_ME_STRONG_PASSWORD
POSTGRES_DB=bank_client_ib_db
DB_USERNAME=bank_client_ib_user
DB_PASSWORD=CHANGE_ME_STRONG_PASSWORD
```

**Password Requirements:**
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, special characters
- Avoid common words

#### 7.2.2 Security Configuration

```bash
# JWT Configuration
JWT_SECRET_KEY=CHANGE_ME_LONG_RANDOM_JWT_SECRET_KEY_AT_LEAST_256_BITS
ACCESS_TOKEN_TTL=1h
REFRESH_TOKEN_TTL=2h
OTP_TTL=5m
OTP_LINK_TTL=24h

# Generate secure JWT secret:
openssl rand -base64 64
```

#### 7.2.3 Core Banking System Integration

```bash
# CBS Adapter Configuration
ABS_URI=https://cbs-middleware.bankdomain.com
ABS_SIGNATURE_KEY=CHANGE_ME_CBS_SIGNATURE_KEY
CBS_TYPE=ABS
CBS_DEFAULT_BANK_CODE=36035
```

**CBS Types Supported:**
- `ABS` - Afriland Banking System
- `T24` - Temenos T24 (future)
- `FLEXCUBE` - Oracle Flexcube (future)

#### 7.2.4 Frontend Configuration

```bash
# React Corporate Client
REACT_APP_API_BASE_URL=https://api.bankdomain.com
REACT_APP_WEB_CORPORATE_BASE_URL=https://corporate.bankdomain.com
REACT_APP_INTERNAL_BANK=36035
REACT_APP_DEFAULT_COUNTRY=ci
REACT_APP_BANK_PHONE_NUMBER=+1234567890
REACT_APP_OTP_CODE_LIFE_TIME=300000

# Feature Flags
REACT_APP_FEATURE_CARD_DISABLE=false
REACT_APP_FEATURE_LOANS_DISABLE=false
REACT_APP_FEATURE_EXCHANGES_DISABLE=false
REACT_APP_FEATURE_DEPOSITS_DISABLE=false

# Vue BackOffice Client
VITE_API_BASE_URL=https://api.bankdomain.com
VITE_INTERNAL_BANK=36035
VITE_DEFAULT_COUNTRY=ci
```

### 7.3 Building Docker Images

#### 7.3.1 Build All Images

```bash
# Build all images at once
docker compose -f docker-compose.yml \
    -f docker-compose-backend.yml \
    -f docker-compose-backend-extended.yml \
    -f docker-compose-frontend.yml \
    build
```

#### 7.3.2 Build Individual Services

```bash
# Build specific backend service
docker compose build auth-service

# Build frontend service
docker compose -f docker-compose-frontend.yml build fe-corporate-client

# Build with no cache (force rebuild)
docker compose build --no-cache gateway-service
```

#### 7.3.3 Image Tagging Strategy

```bash
# Tag for versioning
docker tag ib-gateway-service:latest ib-gateway-service:1.10.0
docker tag ib-auth-service:latest ib-auth-service:1.42.0

# Tag for registry push
docker tag ib-gateway-service:latest registry.yourbank.com/ib-gateway-service:1.10.0
```

### 7.4 Database Initialization

#### 7.4.1 Automatic Initialization

The `init-scripts/01-init-databases.sql` runs automatically on first PostgreSQL startup:

```sql
-- Create databases
CREATE DATABASE bank_client_ib_db;
CREATE DATABASE settings_db;

-- Create user
CREATE USER bank_client_ib_user WITH ENCRYPTED PASSWORD 'YourPassword';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE bank_client_ib_db TO bank_client_ib_user;
GRANT ALL PRIVILEGES ON DATABASE settings_db TO bank_client_ib_user;
```

#### 7.4.2 Schema Migration

```bash
# Migration runs automatically via schema-migration container
docker compose up schema-migration

# Check migration logs
docker compose logs schema-migration

# Manual migration (if needed)
docker compose exec postgres psql -U postgres -d bank_client_ib_db -f /backup/migration.sql
```

### 7.5 Starting Services

#### 7.5.1 Start Infrastructure Only

```bash
# Start PostgreSQL, Redis, MinIO
docker compose up -d postgres redis minio

# Wait for health checks
docker compose ps
```

#### 7.5.2 Start Backend Services

```bash
# Start all backend services
docker compose -f docker-compose.yml \
    -f docker-compose-backend.yml \
    -f docker-compose-backend-extended.yml \
    up -d
```

#### 7.5.3 Start Frontend Services

```bash
# Start all services including frontend
docker compose -f docker-compose.yml \
    -f docker-compose-backend.yml \
    -f docker-compose-backend-extended.yml \
    -f docker-compose-frontend.yml \
    up -d
```

#### 7.5.4 Verify Service Health

```bash
# Check container status
docker compose ps

# View logs for specific service
docker compose logs -f auth-service

# Check health endpoints
curl http://localhost:8080/actuator/health | jq
curl http://localhost:8082/auth/actuator/health | jq
curl http://localhost:8084/user/actuator/health | jq
```

---

## 8. Configuration Reference

### 8.1 Complete Environment Variable List

#### Infrastructure Variables (15 variables)

| Variable | Description | Example | Required |
|----------|-------------|---------|----------|
| `POSTGRES_USER` | Database username | bank_client_ib_user | ✅ |
| `POSTGRES_PASSWORD` | Database password | StrongPass123! | ✅ |
| `POSTGRES_DB` | Main database name | bank_client_ib_db | ✅ |
| `REDIS_PASSWORD` | Redis password | RedisPass456! | ✅ |
| `MINIO_ROOT_USER` | MinIO admin user | minioadmin | ✅ |
| `MINIO_ROOT_PASSWORD` | MinIO password | MinIOPass789! | ✅ |
| `MINIO_ENDPOINT` | MinIO URL | http://minio:9000 | ✅ |
| `MINIO_BUCKET_NAME` | Storage bucket | ib-documents | ✅ |

#### Security Variables (12 variables)

| Variable | Description | Example | Required |
|----------|-------------|---------|----------|
| `JWT_SECRET_KEY` | JWT signing key | base64-encoded-secret | ✅ |
| `ACCESS_TOKEN_TTL` | Access token lifetime | 1h | ✅ |
| `REFRESH_TOKEN_TTL` | Refresh token lifetime | 2h | ✅ |
| `OTP_TTL` | OTP validity period | 5m | ✅ |
| `TOTP_ISSUER` | TOTP issuer name | InternetBankingV3 | ✅ |

#### Bank Configuration (8 variables)

| Variable | Description | Example | Required |
|----------|-------------|---------|----------|
| `BANK_NAME` | Bank display name | AFRILAND FIRST BANK | ✅ |
| `CBS_DEFAULT_BANK_CODE` | Bank identifier | 36035 | ✅ |
| `DEFAULT_CURRENCY` | Base currency | USD | ✅ |
| `DEFAULT_LANGUAGE` | UI language | en | ✅ |

#### CBS Integration (5 variables)

| Variable | Description | Example | Required |
|----------|-------------|---------|----------|
| `ABS_URI` | CBS endpoint | https://cbs.bank.com | ✅ |
| `ABS_SIGNATURE_KEY` | CBS signature key | secret-key | ✅ |
| `CBS_TYPE` | CBS system type | ABS | ✅ |

#### Frontend Variables (25+ variables)

**React Corporate Client:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `REACT_APP_API_BASE_URL` | Backend API URL | - | ✅ |
| `REACT_APP_WEB_CORPORATE_BASE_URL` | Corporate URL | - | ✅ |
| `REACT_APP_INTERNAL_BANK` | Bank code | 36035 | ✅ |
| `REACT_APP_DEFAULT_COUNTRY` | Default country | ci | ✅ |
| `REACT_APP_BANK_PHONE_NUMBER` | Support phone | +1234567890 | ✅ |
| `REACT_APP_FEATURE_CARD_DISABLE` | Disable cards | false | ❌ |
| `REACT_APP_FEATURE_LOANS_DISABLE` | Disable loans | false | ❌ |
| `REACT_APP_FEATURE_EXCHANGES_DISABLE` | Disable FX | false | ❌ |

**Vue BackOffice Client:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `VITE_API_BASE_URL` | Backend API URL | - | ✅ |
| `VITE_INTERNAL_BANK` | Bank code | 36035 | ✅ |
| `VITE_DEFAULT_COUNTRY` | Default country | ci | ✅ |

### 8.2 Port Mapping Reference

#### External Ports (Accessible from Host)

| Port | Service | Purpose | Production |
|------|---------|---------|------------|
| 80 | NGINX | HTTP (redirects to HTTPS) | ✅ |
| 443 | NGINX | HTTPS traffic | ✅ |
| 3002 | Corporate Frontend | React UI (dev only) | ❌ |
| 3004 | BackOffice Frontend | Vue UI (dev only) | ❌ |
| 8080 | Gateway | API (dev only) | ❌ |

#### Internal Ports (Container Network Only)

| Port | Service | Purpose |
|------|---------|---------|
| 5432 | PostgreSQL | Database |
| 6379 | Redis | Cache |
| 9000 | MinIO | Object storage |
| 9001 | MinIO Console | Admin UI |
| 8082 | Auth Service | Authentication |
| 8084 | User Service | User management |
| 8083 | Payment Service | Payments |
| 8081 | Transaction Service | Transactions |
| 8085 | ABS Adapter | CBS integration |
| 8086 | Blobpath Service | File storage |
| 8888 | Settings Service | Configuration |

### 8.3 Volume Management

```bash
# List all volumes
docker volume ls | grep internet-banking

# Inspect volume
docker volume inspect internet-banking-v3_postgres-data

# Backup volume
docker run --rm \
    -v internet-banking-v3_postgres-data:/data \
    -v $(pwd)/backups:/backup \
    alpine tar czf /backup/postgres-backup.tar.gz -C /data .

# Restore volume
docker run --rm \
    -v internet-banking-v3_postgres-data:/data \
    -v $(pwd)/backups:/backup \
    alpine tar xzf /backup/postgres-backup.tar.gz -C /data
```

---

## 9. Service Catalog

### 9.1 Infrastructure Services

#### 9.1.1 PostgreSQL 16

**Purpose**: Primary relational database storing all business data

**Container**: `ib-postgres`  
**Image**: `postgres:16-alpine`  
**Port**: 5432  
**Volume**: `postgres-data`

**Key Features**:
- Automatic initialization via init scripts
- Health checks enabled
- Connection pooling support
- Supports 2 databases: `bank_client_ib_db`, `settings_db`

**Management**:
```bash
# Connect to database
docker compose exec postgres psql -U postgres

# Run SQL file
docker compose exec -T postgres psql -U postgres -d bank_client_ib_db < script.sql

# Database backup
docker compose exec postgres pg_dump -U postgres bank_client_ib_db > backup.sql

# View logs
docker compose logs -f postgres
```

#### 9.1.2 Redis 7

**Purpose**: Session storage, caching, rate limiting

**Container**: `ib-redis`  
**Image**: `redis:7-alpine`  
**Port**: 6379  
**Volume**: `redis-data`

**Key Features**:
- Password-protected
- Persistence enabled (AOF + RDB)
- Used by all backend services

**Management**:
```bash
# Connect to Redis CLI
docker compose exec redis redis-cli -a ${REDIS_PASSWORD}

# Monitor commands
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} MONITOR

# Check memory usage
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} INFO memory

# Flush all cache (CAUTION!)
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} FLUSHALL
```

#### 9.1.3 MinIO

**Purpose**: Object storage for documents, attachments, files

**Container**: `ib-minio`  
**Image**: `minio/minio:latest`  
**Ports**: 9000 (API), 9001 (Console)  
**Volume**: `minio-data`

**Key Features**:
- S3-compatible API
- Web-based management console
- Automatic bucket creation

**Management**:
```bash
# Access MinIO Console
# URL: http://localhost:9001
# Username: minioadmin (from .env)
# Password: from MINIO_ROOT_PASSWORD

# List buckets via CLI
docker compose exec minio mc ls local

# View logs
docker compose logs -f minio
```

### 9.2 Backend Services

#### 9.2.1 Gateway Service

**Purpose**: API Gateway - Single entry point for all API requests

**Container**: `ib-gateway-service`  
**Port**: 8080  
**Technology**: Spring Cloud Gateway

**Responsibilities**:
- Request routing to microservices
- Load balancing
- CORS handling
- Request/response logging

**Health Check**:
```bash
curl http://localhost:8080/actuator/health
```

#### 9.2.2 Auth Service

**Purpose**: Authentication & Authorization

**Container**: `ib-auth-service`  
**Port**: 8082  
**Technology**: Spring Boot + Spring Security

**Responsibilities**:
- User authentication (login/logout)
- JWT token generation and validation
- OTP/TOTP management
- Password reset flows
- Session management

**Key Endpoints**:
- `POST /auth/login` - User login
- `POST /auth/logout` - User logout
- `POST /auth/refresh` - Refresh token
- `POST /auth/forgot-password` - Password reset

**Health Check**:
```bash
curl http://localhost:8082/auth/actuator/health
```

#### 9.2.3 User Service

**Purpose**: User & Account Management

**Container**: `ib-user-service`  
**Port**: 8084  
**Technology**: Spring Boot

**Responsibilities**:
- User profile management
- Account information retrieval
- Sub-user management
- User permissions
- Customer data management

**Health Check**:
```bash
curl http://localhost:8084/user/actuator/health
```

#### 9.2.4 Payment Service

**Purpose**: Payment Processing & Transactions

**Container**: `ib-payment-service`  
**Port**: 8083  
**Technology**: Spring Boot

**Responsibilities**:
- Internal transfers
- External transfers (SWIFT, ACH)
- Bill payments
- Bulk payments
- Payment validation and approval workflows

**Health Check**:
```bash
curl http://localhost:8083/payment/actuator/health
```

#### 9.2.5 Transaction Service

**Purpose**: Transaction History & Reporting

**Container**: `ib-transaction-service`  
**Port**: 8081  
**Technology**: Spring Boot

**Responsibilities**:
- Transaction history retrieval
- Account statements
- Transaction search and filtering
- Export functionality (PDF, Excel, CSV)

**Health Check**:
```bash
curl http://localhost:8081/transaction/actuator/health
```

#### 9.2.6 ABS Adapter Service

**Purpose**: Core Banking System Integration

**Container**: `ib-abs-adapter-service`  
**Port**: 8085  
**Technology**: Spring Boot

**Responsibilities**:
- CBS middleware communication
- Account balance inquiries
- Transaction posting to CBS
- Customer data synchronization
- CBS response translation

**Configuration**:
- `ABS_URI`: CBS endpoint
- `ABS_SIGNATURE_KEY`: Security key
- `CBS_TYPE`: System type (ABS, T24, etc.)

**Health Check**:
```bash
curl http://localhost:8085/adapter/actuator/health
```

#### 9.2.7 Blobpath Service

**Purpose**: File Upload/Download Management

**Container**: `ib-blobpath-service`  
**Port**: 8086  
**Technology**: Spring Boot + MinIO Client

**Responsibilities**:
- Document uploads
- File downloads
- Storage management
- File type validation
- Virus scanning integration (optional)

**Supported Storage Backends**:
- MinIO (default)
- Azure Blob Storage
- Local filesystem

**Health Check**:
```bash
curl http://localhost:8086/blobpath/actuator/health
```

#### 9.2.8 Settings Service

**Purpose**: System Configuration & Settings

**Container**: `ib-settings-service`  
**Port**: 8888  
**Technology**: Spring Boot

**Responsibilities**:
- System-wide settings
- Feature flags
- Exchange rates
- Bank holidays
- Fee configurations

**Database**: `settings_db` (separate from main DB)

**Health Check**:
```bash
curl http://localhost:8888/settings/actuator/health
```

### 9.3 Frontend Services

#### 9.3.1 Corporate Client

**Purpose**: React-based Corporate Banking Interface

**Container**: `ib-fe-corporate-client`  
**Port**: 3002 (80 internal)  
**Technology**: React 18 + TypeScript

**Features**:
- Account management
- Fund transfers
- Bill payments
- Transaction history
- Card management
- Loan applications
- Multi-user support with approvals

**Build**:
```bash
docker compose -f docker-compose-frontend.yml build fe-corporate-client
```

#### 9.3.2 Main BackOffice

**Purpose**: Vue-based Administrator Dashboard

**Container**: `ib-fe-main-bo-client`  
**Port**: 3004 (80 internal)  
**Technology**: Vue 3 + TypeScript

**Features**:
- User management
- Transaction monitoring
- System configuration
- Report generation
- Audit logs
- Approval workflows
- Customer support tools

**Build**:
```bash
docker compose -f docker-compose-frontend.yml build fe-main-bo-client
```

---

## 10. Operations & Maintenance

### 10.1 Daily Operations

#### 10.1.1 Service Status Checks

```bash
# Check all containers
docker compose ps

# Check specific service
docker compose ps auth-service

# View resource usage
docker stats

# Check disk usage
docker system df
```

#### 10.1.2 Log Management

```bash
# View logs for all services
./logs.sh

# View logs for specific service
./logs.sh auth-service

# Follow logs in real-time
docker compose logs -f auth-service

# View last 100 lines
docker compose logs --tail=100 auth-service

# Save logs to file
docker compose logs auth-service > auth-service.log
```

#### 10.1.3 Service Restart

```bash
# Restart all services
./restart.sh

# Restart specific service
./restart.sh auth-service

# Or using docker compose
docker compose restart auth-service
```

### 10.2 Scaling Services

#### 10.2.1 Horizontal Scaling

```bash
# Scale specific service to 3 replicas
docker compose up -d --scale auth-service=3

# Scale multiple services
docker compose up -d --scale auth-service=3 --scale user-service=2
```

#### 10.2.2 Resource Limits

Add to `docker-compose.yml`:

```yaml
services:
  auth-service:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
```

### 10.3 Updates & Deployments

#### 10.3.1 Rolling Updates

```bash
# Pull latest images
docker compose pull

# Recreate containers with new images
docker compose up -d --no-deps --build auth-service

# Verify new version
docker compose exec auth-service java -jar app.jar --version
```

#### 10.3.2 Blue-Green Deployment

```bash
# Start new version alongside old
docker compose -f docker-compose.yml -f docker-compose-blue.yml up -d

# Test new version
curl http://localhost:8082/auth/actuator/health

# Switch traffic (update NGINX)
# Stop old version
docker compose -f docker-compose-green.yml down
```

### 10.4 Database Maintenance

#### 10.4.1 Database Backup

```bash
# Using backup script
./backup.sh

# Manual PostgreSQL backup
docker compose exec postgres pg_dump -U postgres bank_client_ib_db | gzip > backup_$(date +%Y%m%d).sql.gz

# Backup specific table
docker compose exec postgres pg_dump -U postgres -t users bank_client_ib_db > users_backup.sql
```

#### 10.4.2 Database Restore

```bash
# Restore from backup
gunzip < backup_20260213.sql.gz | docker compose exec -T postgres psql -U postgres bank_client_ib_db

# Restore specific table
docker compose exec -T postgres psql -U postgres bank_client_ib_db < users_backup.sql
```

#### 10.4.3 Database Optimization

```bash
# Connect to database
docker compose exec postgres psql -U postgres bank_client_ib_db

# Run VACUUM
VACUUM ANALYZE;

# Reindex database
REINDEX DATABASE bank_client_ib_db;

# Check table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### 10.5 Monitoring Commands

```bash
# Container health status
docker compose ps

# Resource usage
docker stats --no-stream

# Network inspection
docker network inspect internet-banking-v3_ib-network

# Volume inspection
docker volume ls | grep internet-banking

# Disk usage by containers
docker system df -v

# Remove unused resources
docker system prune -a
```

---

## 11. Security Hardening

### 11.1 SSL/TLS Configuration

#### 11.1.1 Let's Encrypt Setup

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Obtain certificates
sudo certbot --nginx \
    -d api.yourbank.com \
    -d corporate.yourbank.com \
    -d mainbo.yourbank.com

# Auto-renewal
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer

# Test renewal
sudo certbot renew --dry-run
```

#### 11.1.2 Custom SSL Certificates

```bash
# Create SSL directory
mkdir -p nginx/ssl

# Copy certificates
cp your-cert.crt nginx/ssl/api.yourbank.com.crt
cp your-key.key nginx/ssl/api.yourbank.com.key

# Set permissions
chmod 600 nginx/ssl/*.key
chmod 644 nginx/ssl/*.crt
```

### 11.2 Network Security

#### 11.2.1 Firewall Rules

```bash
# Configure UFW
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential ports
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS

# Allow from specific IP (optional)
sudo ufw allow from 10.0.0.0/8 to any port 22

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

#### 11.2.2 Docker Network Isolation

```yaml
# In docker-compose.yml
networks:
  ib-network:
    driver: bridge
    internal: true  # No external access
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### 11.3 Secrets Management

#### 11.3.1 Docker Secrets (Swarm Mode)

```bash
# Create secret
echo "MyStrongPassword123!" | docker secret create postgres_password -

# Use in docker-compose.yml
services:
  postgres:
    secrets:
      - postgres_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password

secrets:
  postgres_password:
    external: true
```

#### 11.3.2 Environment File Protection

```bash
# Restrict .env file permissions
chmod 600 .env

# Exclude from git
echo ".env" >> .gitignore

# Encrypt sensitive files
gpg -c .env  # Creates .env.gpg
```

### 11.4 Application Security

#### 11.4.1 Security Headers (NGINX)

Already configured in `nginx/nginx.conf`:

```nginx
# Security headers
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
add_header Content-Security-Policy "default-src 'self'" always;
```

#### 11.4.2 Rate Limiting

Configured in `nginx/conf.d/api.conf`:

```nginx
# Rate limiting zones
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;

# Apply to locations
location /auth/login {
    limit_req zone=login_limit burst=5 nodelay;
}
```

### 11.5 Access Control

#### 11.5.1 IP Whitelisting (BackOffice)

In `nginx/conf.d/mainbo.conf`:

```nginx
# Allow specific IPs only
allow 10.0.0.0/8;      # Internal network
allow 203.0.113.0/24;  # Office network
deny all;
```

#### 11.5.2 Database Access Restriction

```bash
# Edit PostgreSQL config
docker compose exec postgres sh -c 'echo "host all all 172.20.0.0/16 md5" >> /var/lib/postgresql/data/pg_hba.conf'

# Restart PostgreSQL
docker compose restart postgres
```

---

## 12. Monitoring & Logging

### 12.1 Health Checks

#### 12.1.1 Service Health Endpoints

```bash
# Gateway
curl http://localhost:8080/actuator/health

# Auth Service
curl http://localhost:8082/auth/actuator/health

# User Service
curl http://localhost:8084/user/actuator/health

# All services health check script
for port in 8080 8082 8083 8084 8081 8085 8086 8888; do
    echo "Checking port $port"
    curl -s http://localhost:$port/actuator/health | jq '.status'
done
```

#### 12.1.2 Automated Health Monitoring

```bash
#!/bin/bash
# health-check.sh

SERVICES=("8080:gateway" "8082:auth" "8084:user" "8083:payment")

for service in "${SERVICES[@]}"; do
    PORT="${service%%:*}"
    NAME="${service##*:}"
    
    STATUS=$(curl -s http://localhost:$PORT/actuator/health | jq -r '.status')
    
    if [ "$STATUS" != "UP" ]; then
        echo "ALERT: $NAME service is DOWN"
        # Send notification (email, Slack, etc.)
    fi
done
```

### 12.2 Log Aggregation

#### 12.2.1 Centralized Logging with ELK Stack

Add to deployment:

```yaml
# docker-compose-logging.yml
services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - ib-network

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    networks:
      - ib-network

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    networks:
      - ib-network
```

#### 12.2.2 Log Rotation

Configure in Docker daemon (`/etc/docker/daemon.json`):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5",
    "labels": "production_status",
    "env": "os,customer"
  }
}
```

### 12.3 Performance Monitoring

#### 12.3.1 Prometheus + Grafana

```yaml
# docker-compose-monitoring.yml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - "9090:9090"
    networks:
      - ib-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - ib-network
```

#### 12.3.2 Application Metrics

Spring Boot Actuator exposes metrics:

```bash
# Get metrics
curl http://localhost:8080/actuator/metrics

# Specific metric
curl http://localhost:8080/actuator/metrics/jvm.memory.used
```

### 12.4 Sentry Integration

Configure in `.env`:

```bash
SENTRY_ENABLED=true
SENTRY_DSN_AUTH=https://your-sentry-dsn@sentry.io/project-id
SENTRY_DSN_USER=https://your-sentry-dsn@sentry.io/project-id
SENTRY_DSN_PAYMENT=https://your-sentry-dsn@sentry.io/project-id
```

---

## 13. Backup & Recovery

### 13.1 Automated Backup Strategy

#### 13.1.1 Using the Backup Script

```bash
# Run backup manually
./backup.sh

# Backup location
ls -lh /opt/backups/internet-banking/

# Expected output:
# ib-backup-20260213_143022.tar.gz
```

**What Gets Backed Up:**
1. PostgreSQL databases (full dump)
2. Redis data
3. MinIO object storage
4. Blobpath file storage
5. Environment configuration
6. NGINX configuration

#### 13.1.2 Schedule Automated Backups

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /path/to/internet-banking-v3/backup.sh >> /var/log/ib-backup.log 2>&1

# Add weekly backup on Sunday at 3 AM
0 3 * * 0 /path/to/internet-banking-v3/backup.sh >> /var/log/ib-backup-weekly.log 2>&1
```

### 13.2 Backup Components

#### 13.2.1 Database Backup

```bash
# Full database dump
docker compose exec postgres pg_dumpall -U postgres | gzip > full-backup-$(date +%Y%m%d).sql.gz

# Specific database
docker compose exec postgres pg_dump -U postgres bank_client_ib_db | gzip > db-backup-$(date +%Y%m%d).sql.gz

# Schema only
docker compose exec postgres pg_dump -U postgres --schema-only bank_client_ib_db > schema-backup.sql

# Data only
docker compose exec postgres pg_dump -U postgres --data-only bank_client_ib_db > data-backup.sql
```

#### 13.2.2 Volume Backup

```bash
# Backup PostgreSQL volume
docker run --rm \
    -v internet-banking-v3_postgres-data:/data \
    -v $(pwd)/backups:/backup \
    alpine tar czf /backup/postgres-volume-$(date +%Y%m%d).tar.gz -C /data .

# Backup MinIO volume
docker run --rm \
    -v internet-banking-v3_minio-data:/data \
    -v $(pwd)/backups:/backup \
    alpine tar czf /backup/minio-volume-$(date +%Y%m%d).tar.gz -C /data .

# Backup blobpath storage
docker run --rm \
    -v internet-banking-v3_blobpath-storage:/data \
    -v $(pwd)/backups:/backup \
    alpine tar czf /backup/blobpath-volume-$(date +%Y%m%d).tar.gz -C /data .
```

### 13.3 Recovery Procedures

#### 13.3.1 Full System Recovery

```bash
# 1. Stop all services
./stop.sh

# 2. Extract backup
tar xzf /opt/backups/internet-banking/ib-backup-20260213_143022.tar.gz

# 3. Restore PostgreSQL
gunzip < databases.sql.gz | docker compose exec -T postgres psql -U postgres

# 4. Restore volumes
docker run --rm \
    -v internet-banking-v3_postgres-data:/data \
    -v $(pwd):/backup \
    alpine tar xzf /backup/postgres-data.tar.gz -C /data

# 5. Restore environment
cp env.backup .env

# 6. Start services
./deploy.sh
```

#### 13.3.2 Database Point-in-Time Recovery

```bash
# Enable WAL archiving (add to docker-compose.yml)
services:
  postgres:
    command: postgres -c wal_level=replica -c archive_mode=on -c archive_command='test ! -f /archive/%f && cp %p /archive/%f'
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - postgres-archive:/archive

# Restore to specific point
# 1. Restore base backup
# 2. Apply WAL files up to target time
# 3. Create recovery.conf with target time
```

### 13.4 Disaster Recovery Plan

#### 13.4.1 RTO & RPO Targets

| Metric | Target | Strategy |
|--------|--------|----------|
| **RTO** (Recovery Time Objective) | 4 hours | Automated restore scripts |
| **RPO** (Recovery Point Objective) | 24 hours | Daily backups |
| **Critical RPO** | 1 hour | Continuous WAL archiving |

#### 13.4.2 DR Checklist

**Before Disaster:**
- ✅ Automated daily backups configured
- ✅ Backups stored off-site or in cloud storage
- ✅ DR procedures documented and tested
- ✅ Secondary site identified (if applicable)
- ✅ Team trained on recovery procedures

**During Disaster:**
1. Assess damage and data loss
2. Notify stakeholders
3. Activate DR team
4. Identify latest valid backup
5. Begin recovery procedure

**After Recovery:**
1. Verify all services operational
2. Validate data integrity
3. Monitor for issues
4. Document incident
5. Update DR procedures

### 13.5 Backup Retention Policy

```bash
# Retention policy in backup.sh (already configured)
# Keep last 30 days of daily backups
find /opt/backups/internet-banking -name "ib-backup-*.tar.gz" -mtime +30 -delete

# Custom retention
# Keep last 7 daily backups
find /opt/backups/internet-banking/daily -name "*.tar.gz" -mtime +7 -delete

# Keep last 4 weekly backups
find /opt/backups/internet-banking/weekly -name "*.tar.gz" -mtime +28 -delete

# Keep last 12 monthly backups
find /opt/backups/internet-banking/monthly -name "*.tar.gz" -mtime +365 -delete
```

---

## 14. Troubleshooting

### 14.1 Common Issues

#### 14.1.1 Containers Won't Start

**Symptom**: Containers exit immediately after starting

**Diagnosis**:
```bash
# Check container logs
docker compose logs auth-service

# Check container exit code
docker compose ps
```

**Common Causes**:
1. **Port already in use**
   ```bash
   # Check what's using the port
   sudo lsof -i :8080
   
   # Kill the process
   sudo kill -9 <PID>
   ```

2. **Missing environment variables**
   ```bash
   # Verify .env file exists
   ls -la .env
   
   # Check for required variables
   grep "POSTGRES_PASSWORD" .env
   ```

3. **Database connection failure**
   ```bash
   # Check PostgreSQL is running
   docker compose ps postgres
   
   # Test database connection
   docker compose exec postgres pg_isready
   ```

#### 14.1.2 Cannot Access Application

**Symptom**: Browser shows "Connection refused" or timeout

**Diagnosis**:
```bash
# Check if containers are running
docker compose ps

# Check NGINX logs
docker compose logs nginx-proxy

# Test local connectivity
curl http://localhost:8080/actuator/health
```

**Solutions**:
1. **Firewall blocking access**
   ```bash
   sudo ufw status
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   ```

2. **NGINX not running**
   ```bash
   docker compose restart nginx-proxy
   ```

3. **DNS not configured**
   ```bash
   # Test with hosts file
   sudo echo "SERVER_IP api.yourbank.com" >> /etc/hosts
   ```

#### 14.1.3 Login Fails Silently

**Symptom**: Login button does nothing, no error message

**Diagnosis**:
```bash
# Check browser console for CORS errors
# F12 > Console

# Check auth service logs
docker compose logs -f auth-service

# Verify JWT configuration
docker compose exec auth-service env | grep JWT
```

**Common Causes**:
1. **CORS misconfiguration**
   - Check `REACT_APP_API_BASE_URL` matches actual API domain
   - Verify CORS settings in Gateway service

2. **JWT secret mismatch**
   ```bash
   # Ensure JWT_SECRET_KEY is set and consistent
   grep JWT_SECRET_KEY .env
   ```

3. **Database connection issues**
   ```bash
   # Check auth service can reach database
   docker compose logs auth-service | grep -i "database\|connection"
   ```

#### 14.1.4 Backend Service Unavailable

**Symptom**: "Service Unavailable" error in BackOffice

**Diagnosis**:
```bash
# Check CBS adapter service
docker compose ps abs-adapter-service
docker compose logs abs-adapter-service

# Test CBS connectivity
docker compose exec abs-adapter-service curl -v ${ABS_URI}
```

**Solutions**:
1. **CBS middleware unreachable**
   - Verify `ABS_URI` is correct
   - Check firewall allows outbound connections
   - Verify CBS middleware is running

2. **Signature key mismatch**
   - Verify `ABS_SIGNATURE_KEY` matches CBS configuration

#### 14.1.5 Slow Performance

**Symptom**: Application is slow or unresponsive

**Diagnosis**:
```bash
# Check resource usage
docker stats

# Check database queries
docker compose exec postgres psql -U postgres -d bank_client_ib_db -c "
SELECT pid, now() - query_start AS duration, query 
FROM pg_stat_activity 
WHERE state = 'active' 
ORDER BY duration DESC;
"

# Check Redis memory
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} INFO memory
```

**Solutions**:
1. **Insufficient resources**
   ```bash
   # Increase Docker resources or upgrade server
   # Check available memory
   free -h
   ```

2. **Database needs optimization**
   ```bash
   # Run VACUUM
   docker compose exec postgres psql -U postgres -d bank_client_ib_db -c "VACUUM ANALYZE;"
   ```

3. **Too many logs**
   ```bash
   # Prune old logs
   docker system prune -a
   ```

### 14.2 Debug Mode

#### 14.2.1 Enable Debug Logging

```bash
# Add to .env
LOGGING_LEVEL_ROOT=DEBUG
LOGGING_LEVEL_COM_PKF=DEBUG

# Restart services
docker compose restart
```

#### 14.2.2 Access Container Shell

```bash
# Access backend service
docker compose exec auth-service sh

# Access database
docker compose exec postgres psql -U postgres

# Access Redis
docker compose exec redis redis-cli -a ${REDIS_PASSWORD}
```

### 14.3 Health Check Commands

```bash
# All services health
docker compose ps

# Gateway
curl http://localhost:8080/actuator/health | jq

# Auth
curl http://localhost:8082/auth/actuator/health | jq

# Database connection
docker compose exec postgres pg_isready -U postgres

# Redis connection
docker compose exec redis redis-cli -a ${REDIS_PASSWORD} ping

# MinIO
curl http://localhost:9000/minio/health/live
```

### 14.4 Getting Help

**Log Collection for Support:**
```bash
# Collect all logs
docker compose logs > all-services.log

# Collect system info
docker version > system-info.txt
docker compose version >> system-info.txt
uname -a >> system-info.txt
free -h >> system-info.txt

# Package everything
tar czf support-package-$(date +%Y%m%d).tar.gz all-services.log system-info.txt .env.example
```

---

## 15. Migration from Legacy

### 15.1 Migration Overview

This section guides you through migrating from the legacy JAR-based deployment to Docker.

**Migration Timeline**: 4-8 hours (depending on data volume)

### 15.2 Pre-Migration Checklist

- ✅ Backup all databases
- ✅ Document current systemd service names
- ✅ Export environment variables from legacy .env files
- ✅ Test Docker deployment in staging environment
- ✅ Plan maintenance window
- ✅ Notify users of downtime

### 15.3 Data Migration

#### 15.3.1 Export Legacy Database

```bash
# On legacy system
sudo -u postgres pg_dump bank_client_ib_db | gzip > legacy-db-backup.sql.gz

# Transfer to new system
scp legacy-db-backup.sql.gz user@new-server:/path/to/backups/
```

#### 15.3.2 Import to Docker PostgreSQL

```bash
# On new Docker system
# 1. Start infrastructure
docker compose up -d postgres redis minio

# 2. Wait for PostgreSQL
sleep 15

# 3. Import data
gunzip < legacy-db-backup.sql.gz | docker compose exec -T postgres psql -U postgres -d bank_client_ib_db
```

#### 15.3.3 Migrate File Storage

```bash
# Copy from legacy MinIO/filesystem to Docker volume
docker run --rm \
    -v /legacy/minio/data:/source \
    -v internet-banking-v3_minio-data:/dest \
    alpine sh -c "cp -r /source/* /dest/"
```

### 15.4 Service Migration

#### 15.4.1 Stop Legacy Services

```bash
# On legacy system
sudo systemctl stop gateway-service
sudo systemctl stop auth-service
sudo systemctl stop user-service
sudo systemctl stop payment-service
# ... stop all services
```

#### 15.4.2 Start Docker Services

```bash
# On new system
./deploy.sh
```

#### 15.4.3 Verify Migration

```bash
# Check all services are up
docker compose ps

# Test login
curl -X POST http://localhost:8082/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin@yourbank.com","password":"YourPassword"}'

# Verify data integrity
docker compose exec postgres psql -U postgres -d bank_client_ib_db -c "SELECT COUNT(*) FROM users;"
```

### 15.5 Rollback Plan

If migration fails:

```bash
# 1. Stop Docker services
./stop.sh

# 2. Restart legacy services
sudo systemctl start gateway-service
sudo systemctl start auth-service
# ... start all legacy services

# 3. Restore legacy database if needed
gunzip < legacy-db-backup.sql.gz | sudo -u postgres psql bank_client_ib_db
```

### 15.6 Post-Migration Tasks

- ✅ Update DNS records (if changed)
- ✅ Update SSL certificates
- ✅ Configure monitoring
- ✅ Set up automated backups
- ✅ Test all critical workflows
- ✅ Monitor for 48 hours
- ✅ Decommission legacy services

---

## 16. Production Deployment Checklist

### 16.1 Pre-Deployment

**Infrastructure:**
- [ ] Server meets minimum specifications (32GB RAM, 8 vCPU, 256GB SSD)
- [ ] Ubuntu 24.04/22.04 LTS installed
- [ ] Docker 24.0+ installed and tested
- [ ] Docker Compose 2.20+ installed
- [ ] Public IP address configured
- [ ] Firewall configured (ports 22, 80, 443)

**DNS & Certificates:**
- [ ] DNS records created (api, corporate, mainbo subdomains)
- [ ] DNS propagation verified
- [ ] SSL certificates obtained
- [ ] SSL certificates installed in nginx/ssl/
- [ ] Certificate auto-renewal configured

**External Dependencies:**
- [ ] Core Banking System (CBS) endpoint accessible
- [ ] CBS credentials obtained
- [ ] SMTP server configured (if using email)
- [ ] SMS gateway configured (if using SMS)

### 16.2 Configuration

**Environment Variables:**
- [ ] Copied .env.example to .env
- [ ] Set strong POSTGRES_PASSWORD
- [ ] Set strong REDIS_PASSWORD
- [ ] Set strong MINIO_ROOT_PASSWORD
- [ ] Generated secure JWT_SECRET_KEY (256-bit)
- [ ] Configured ABS_URI (CBS endpoint)
- [ ] Configured ABS_SIGNATURE_KEY
- [ ] Set BANK_NAME
- [ ] Set CBS_DEFAULT_BANK_CODE
- [ ] Set FIRST_SUPER_ADMIN_EMAIL
- [ ] Set FIRST_SUPER_ADMIN_PASSWORD (change after login!)
- [ ] Updated all REACT_APP_* variables
- [ ] Updated all VITE_* variables
- [ ] Configured SMTP settings (if applicable)
- [ ] Reviewed and set feature flags

**Security:**
- [ ] Changed all default passwords
- [ ] Restricted .env file permissions (chmod 600)
- [ ] Configured firewall rules
- [ ] Enabled SSL/TLS
- [ ] Configured rate limiting
- [ ] Set up IP whitelisting for BackOffice (optional)
- [ ] Reviewed security headers in NGINX config

### 16.3 Deployment

- [ ] Made scripts executable (chmod +x *.sh)
- [ ] Ran ./deploy.sh
- [ ] All 15 containers started successfully
- [ ] Database migration completed without errors
- [ ] All health checks passing
- [ ] Reviewed logs for errors or warnings

### 16.4 Verification

**Service Health:**
- [ ] Gateway health: `curl http://localhost:8080/actuator/health`
- [ ] Auth health: `curl http://localhost:8082/auth/actuator/health`
- [ ] User health: `curl http://localhost:8084/user/actuator/health`
- [ ] Payment health: `curl http://localhost:8083/payment/actuator/health`
- [ ] Transaction health: `curl http://localhost:8081/transaction/actuator/health`
- [ ] ABS Adapter health: `curl http://localhost:8085/adapter/actuator/health`

**Frontend Access:**
- [ ] Corporate frontend accessible: https://corporate.yourbank.com
- [ ] BackOffice accessible: https://mainbo.yourbank.com
- [ ] HTTPS redirects working correctly
- [ ] No SSL certificate warnings

**Functional Testing:**
- [ ] Can login with admin credentials
- [ ] Can access user dashboard
- [ ] Can view account balances
- [ ] Can create test transaction
- [ ] Can approve transaction (if applicable)
- [ ] Email notifications working (if configured)
- [ ] SMS notifications working (if configured)

### 16.5 Security Validation

- [ ] Changed default admin password
- [ ] Tested HTTPS enforcement
- [ ] Verified security headers present
- [ ] Tested rate limiting on login endpoint
- [ ] Confirmed database not accessible externally
- [ ] Confirmed Redis not accessible externally
- [ ] Verified file upload restrictions working

### 16.6 Backup & Monitoring

- [ ] Backup script tested successfully
- [ ] Automated backup cron job configured
- [ ] Backup restoration tested
- [ ] Log aggregation configured (optional)
- [ ] Monitoring configured (Sentry/Prometheus)
- [ ] Alert notifications configured
- [ ] Disaster recovery plan documented

### 16.7 Documentation & Training

- [ ] Deployment documentation updated
- [ ] Team trained on Docker commands
- [ ] Runbooks created for common operations
- [ ] Escalation procedures documented
- [ ] Contact list updated

### 16.8 Go-Live

- [ ] Maintenance window scheduled and communicated
- [ ] Users notified of upcoming deployment
- [ ] Rollback plan prepared and tested
- [ ] All stakeholders ready
- [ ] Deploy to production
- [ ] Monitor for first 24-48 hours
- [ ] Conduct post-deployment review

**Sign-off:**

| Role | Name | Signature | Date |
|------|------|-----------|------|
| DevOps Lead | __________ | __________ | __________ |
| IT Manager | __________ | __________ | __________ |
| Project Manager | __________ | __________ | __________ |

---

## 17. Appendices

### 17.1 Complete File Deliverables

#### 17.1.1 Documentation Files (~95 pages)

| File | Description | Size |
|------|-------------|------|
| `COMPREHENSIVE_DEPLOYMENT_GUIDE.md` | This complete guide | 100+ pages |
| `INDEX.md` | Documentation index | 5 pages |
| `README.md` | Quick reference | 5 pages |
| `QUICKSTART.md` | 15-min deployment | 6 pages |
| `DEPLOYMENT_CHECKLIST.md` | Production checklist | 4 pages |

#### 17.1.2 Docker Compose Files

| File | Purpose | Services |
|------|---------|----------|
| `docker-compose.yml` | Infrastructure | PostgreSQL, Redis, MinIO, Schema Migration |
| `docker-compose-backend.yml` | Core backend | Auth, User, Payment |
| `docker-compose-backend-extended.yml` | Extended backend | Transaction, ABS Adapter, Blobpath, Settings, Gateway |
| `docker-compose-frontend.yml` | Frontend | Corporate Client, BackOffice, NGINX |

#### 17.1.3 Operational Scripts

| Script | Purpose |
|--------|---------|
| `deploy.sh` | Automated deployment with health checks |
| `stop.sh` | Stop all services gracefully |
| `restart.sh` | Restart services (all or specific) |
| `logs.sh` | View service logs |
| `backup.sh` | Automated backup with retention |

#### 17.1.4 Configuration Files

| File/Directory | Purpose |
|----------------|---------|
| `.env.example` | Environment variable template (80+ vars) |
| `nginx/nginx.conf` | NGINX main configuration |
| `nginx/conf.d/api.conf` | API Gateway routing |
| `nginx/conf.d/corporate.conf` | Corporate frontend |
| `nginx/conf.d/mainbo.conf` | BackOffice with IP whitelist |
| `init-scripts/01-init-databases.sql` | Database initialization |

### 17.2 Service Versions

| Service | Version | Image |
|---------|---------|-------|
| PostgreSQL | 16 | postgres:16-alpine |
| Redis | 7 | redis:7-alpine |
| MinIO | Latest | minio/minio:latest |
| Gateway | 1.10.0 | Custom (Spring Boot) |
| Auth | 1.42.0 | Custom (Spring Boot) |
| User | 1.67.0 | Custom (Spring Boot) |
| Payment | Latest | Custom (Spring Boot) |
| Transaction | Latest | Custom (Spring Boot) |
| ABS Adapter | 1.29.0 | Custom (Spring Boot) |
| Blobpath | 1.0.0 | Custom (Spring Boot) |
| Settings | 1.2.0 | Custom (Spring Boot) |
| Corporate Client | Latest | Custom (React 18) |
| Main BackOffice | Latest | Custom (Vue 3) |
| NGINX | Latest | nginx:alpine |

### 17.3 Port Reference

**External (Host-accessible):**
- 80 - HTTP (redirects to HTTPS)
- 443 - HTTPS (production traffic)
- 3002 - Corporate Frontend (dev only)
- 3004 - BackOffice (dev only)
- 8080 - API Gateway (dev only)

**Internal (Container network):**
- 5432 - PostgreSQL
- 6379 - Redis
- 9000/9001 - MinIO
- 8082 - Auth Service
- 8084 - User Service
- 8083 - Payment Service
- 8081 - Transaction Service
- 8085 - ABS Adapter
- 8086 - Blobpath Service
- 8888 - Settings Service

### 17.4 Quick Command Reference

```bash
# Deployment
./deploy.sh                  # Full deployment
./stop.sh                    # Stop all services
./restart.sh                 # Restart all
./restart.sh auth-service    # Restart specific service
./logs.sh                    # View all logs
./logs.sh auth-service       # View specific logs
./backup.sh                  # Create backup

# Service Management
docker compose ps            # List containers
docker compose logs -f       # Follow logs
docker compose restart       # Restart services
docker compose down          # Stop and remove
docker compose up -d         # Start in background

# Health Checks
curl http://localhost:8080/actuator/health
curl http://localhost:8082/auth/actuator/health

# Database
docker compose exec postgres psql -U postgres
docker compose exec postgres pg_dump -U postgres bank_client_ib_db > backup.sql

# Redis
docker compose exec redis redis-cli -a ${REDIS_PASSWORD}

# Cleanup
docker system prune -a       # Remove unused resources
docker volume prune          # Remove unused volumes
```

### 17.5 Support Contacts

| Component | Contact | Email |
|-----------|---------|-------|
| Platform Architecture | PKF Research Center | support@pkf-researchcenter.com |
| Docker Infrastructure | DevOps Team | devops@pkf-researchcenter.com |
| Core Banking Integration | CBS Team | cbs-support@pkf-researchcenter.com |

### 17.6 Additional Resources

- **Docker Documentation**: https://docs.docker.com/
- **Docker Compose Reference**: https://docs.docker.com/compose/
- **NGINX Documentation**: https://nginx.org/en/docs/
- **PostgreSQL Manual**: https://www.postgresql.org/docs/
- **Spring Boot Actuator**: https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 3.0 | 2026-02-13 | DevOps Team | Complete Docker transformation |
| 2.0 | 2025-08-25 | DevOps Team | Legacy JAR deployment |
| 1.0 | 2024-06-15 | DevOps Team | Initial deployment guide |

---

**End of Document**

---

**Document Statistics:**
- **Total Sections**: 17 major sections
- **Total Pages**: ~150 pages (Markdown equivalent)
- **Code Examples**: 100+ executable commands
- **Configuration Samples**: 50+ configurations
- **Services Documented**: 15 containerized services
- **Environment Variables**: 80+ documented variables

**For Questions or Support:**  
Contact PKF Research Center DevOps Team  
Email: devops@pkf-researchcenter.com

---

# APPENDIX A: Complete Docker Compose Configurations

## A.1 Infrastructure Services (docker-compose.yml)

This file defines the core infrastructure services: PostgreSQL, Redis, MinIO, and database schema migration.

```yaml
version: '3.8'

networks:
  ib-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  minio-data:
  blobpath-storage:

services:
  # ==============================================
  # INFRASTRUCTURE SERVICES
  # ==============================================
  
  postgres:
    image: postgres:16-alpine
    container_name: ib-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB:-bank_client_ib_db}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - ib-network
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: ib-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    networks:
      - ib-network
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:latest
    container_name: ib-minio
    restart: unless-stopped
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - minio-data:/data
    networks:
      - ib-network
    ports:
      - "9000:9000"
      - "9001:9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # ==============================================
  # DATABASE SCHEMA MIGRATION
  # ==============================================
  
  schema-migration:
    build:
      context: ./pkf_ib_db_schema
      dockerfile: Dockerfile
    container_name: ib-schema-migration
    environment:
      DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-bank_client_ib_db}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      SPRING_PROFILES_ACTIVE: default
      SPRING_LIQUIBASE_ENABLED: "true"
    networks:
      - ib-network
    depends_on:
      postgres:
        condition: service_healthy
    restart: "no"
```

**Usage:**
```bash
# Start infrastructure only
docker compose up -d postgres redis minio

# Run schema migration
docker compose up schema-migration

# View infrastructure logs
docker compose logs -f postgres redis minio
```

---

## A.2 Core Backend Services (docker-compose-backend.yml)

This file defines the core backend services: Auth, User, and Payment services.

```yaml
version: '3.8'

# Backend services configuration
# Use with: docker-compose -f docker-compose.yml -f docker-compose-backend.yml up

services:
  # ==============================================
  # BACKEND SERVICES
  # ==============================================

  auth-service:
    build:
      context: ./pkf-ib-be-auth-service
      dockerfile: Dockerfile
    container_name: ib-auth-service
    restart: unless-stopped
    environment:
      PORT: 8082
      DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-bank_client_ib_db}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      ADAPTER_URL: http://abs-adapter-service:8085
      BLOB_PATH_URI: http://blobpath-service:8083
      SETTINGS_URI: http://settings-service:8888
      USER_URI: http://user-service:8084
      APP_SECURITY_JWT_SECRET_KEY: ${JWT_SECRET_KEY}
      ACCESS_TOKEN_TTL: ${ACCESS_TOKEN_TTL:-1h}
      REFRESH_TOKEN_TTL: ${REFRESH_TOKEN_TTL:-2h}
      OTP_TTL: ${OTP_TTL:-5m}
      OTP_LINK_TTL: ${OTP_LINK_TTL:-24h}
      OTP_GENERATION_ENABLED: ${OTP_GENERATION_ENABLED:-true}
      TOTP_ISSUER: ${TOTP_ISSUER:-InternetBankingV3}
      TOTP_PASSPHRASE_BASE64: ${TOTP_PASSPHRASE_BASE64}
      TOTP_PASSPHRASE_SALT_BASE64: ${TOTP_PASSPHRASE_SALT_BASE64}
      FIRST_SUPER_ADMIN_EMAIL: ${FIRST_SUPER_ADMIN_EMAIL}
      FIRST_SUPER_ADMIN_PASSWORD: ${FIRST_SUPER_ADMIN_PASSWORD:-password123$QWE}
      BANK_NAME: ${BANK_NAME}
      CBS_DEFAULT_BANK_CODE: ${CBS_DEFAULT_BANK_CODE}
      DEFAULT_CURRENCY: ${DEFAULT_CURRENCY}
      DEFAULT_LANGUAGE: ${DEFAULT_LANGUAGE:-en}
      NOTIFICATION_EMAIL_ENABLED: ${NOTIFICATION_EMAIL_ENABLED:-true}
      NOTIFICATION_SMS_ENABLED: ${NOTIFICATION_SMS_ENABLED:-true}
      IB_FO_URL: ${IB_FO_URL}
      IB_ADMIN_URL: ${IB_ADMIN_URL}
      IB_SUPER_ADMIN_URL: ${IB_SUPER_ADMIN_URL}
      SENTRY_ENABLED: ${SENTRY_ENABLED:-false}
      SENTRY_DSN_AUTH: ${SENTRY_DSN_AUTH:-}
    networks:
      - ib-network
    ports:
      - "8082:8082"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8082/auth/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  user-service:
    build:
      context: ./pkf-ib-be-user-service
      dockerfile: Dockerfile
    container_name: ib-user-service
    restart: unless-stopped
    environment:
      PORT: 8084
      DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-bank_client_ib_db}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      ADAPTER_URL: http://abs-adapter-service:8085
      BLOB_PATH_URI: http://blobpath-service:8083
      SETTINGS_URI: http://settings-service:8888
      AUTH_URI: http://auth-service:8082
      PAYMENT_URI: http://payment-service:8083
      BANK_NAME: ${BANK_NAME}
      CBS_DEFAULT_BANK_CODE: ${CBS_DEFAULT_BANK_CODE}
      DEFAULT_CURRENCY: ${DEFAULT_CURRENCY}
      SENTRY_ENABLED: ${SENTRY_ENABLED:-false}
      SENTRY_DSN_USER: ${SENTRY_DSN_USER:-}
    networks:
      - ib-network
    ports:
      - "8084:8084"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8084/user/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  payment-service:
    build:
      context: ./pkf-ib-be-payment
      dockerfile: Dockerfile
    container_name: ib-payment-service
    restart: unless-stopped
    environment:
      PORT: 8083
      DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-bank_client_ib_db}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      ADAPTER_URL: http://abs-adapter-service:8085
      SETTINGS_URI: http://settings-service:8888
      BANK_NAME: ${BANK_NAME}
      CBS_DEFAULT_BANK_CODE: ${CBS_DEFAULT_BANK_CODE}
      DEFAULT_CURRENCY: ${DEFAULT_CURRENCY}
      SENTRY_ENABLED: ${SENTRY_ENABLED:-false}
      SENTRY_DSN_PAYMENT: ${SENTRY_DSN_PAYMENT:-}
    networks:
      - ib-network
    ports:
      - "8083:8083"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8083/payment/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

**Usage:**
```bash
# Start infrastructure + backend
docker compose -f docker-compose.yml -f docker-compose-backend.yml up -d

# View backend logs
docker compose logs -f auth-service user-service payment-service
```

---

## A.3 Extended Backend Services (docker-compose-backend-extended.yml)

This file defines additional backend services: Transaction, ABS Adapter, Blobpath, Settings, and Gateway.

```yaml
version: '3.8'

# Extended backend services configuration
# Use with: docker-compose -f docker-compose.yml -f docker-compose-backend.yml -f docker-compose-backend-extended.yml up

services:
  transaction-service:
    build:
      context: ./pkf-ib-be-transaction-service
      dockerfile: Dockerfile
    container_name: ib-transaction-service
    restart: unless-stopped
    environment:
      PORT: 8081
      DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-bank_client_ib_db}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      SETTINGS_URI: http://settings-service:8888
      BANK_NAME: ${BANK_NAME}
      DEFAULT_CURRENCY: ${DEFAULT_CURRENCY}
      SENTRY_ENABLED: ${SENTRY_ENABLED:-false}
      SENTRY_DSN_TRANSACTION: ${SENTRY_DSN_TRANSACTION:-}
    networks:
      - ib-network
    ports:
      - "8081:8081"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8081/transaction/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  abs-adapter-service:
    build:
      context: ./pkf-ib-be-abs-adapter-service
      dockerfile: Dockerfile
    container_name: ib-abs-adapter-service
    restart: unless-stopped
    environment:
      PORT: 8085
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      SETTINGS_URI: http://settings-service:8888
      ABS_URI: ${ABS_URI}
      ABS_SIGNATURE_KEY: ${ABS_SIGNATURE_KEY}
      CBS_TYPE: ${CBS_TYPE:-ABS}
      BANK_NAME: ${BANK_NAME}
      SENTRY_ENABLED: ${SENTRY_ENABLED:-false}
      SENTRY_DSN_ADAPTER: ${SENTRY_DSN_ADAPTER:-}
    networks:
      - ib-network
    ports:
      - "8085:8085"
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8085/adapter/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  blobpath-service:
    build:
      context: ./pkf-ib-be-blobpath-service
      dockerfile: Dockerfile
    container_name: ib-blobpath-service
    restart: unless-stopped
    environment:
      PORT: 8086
      SPRING_PROFILES_ACTIVE: ${BLOB_STORAGE_TYPE:-minio}
      # MinIO Configuration
      MINIO_ENDPOINT: ${MINIO_ENDPOINT:-http://minio:9000}
      MINIO_ACCESS_KEY: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_SECRET_KEY: ${MINIO_ROOT_PASSWORD}
      MINIO_BUCKET_NAME: ${MINIO_BUCKET_NAME:-ib-documents}
      # Azure Blob Storage Configuration (if using Azure)
      AZURE_STORAGE_ACCOUNT_NAME: ${AZURE_STORAGE_ACCOUNT_NAME:-}
      AZURE_STORAGE_ACCOUNT_KEY: ${AZURE_STORAGE_ACCOUNT_KEY:-}
      AZURE_STORAGE_CONTAINER_NAME: ${AZURE_STORAGE_CONTAINER_NAME:-}
      # File System Configuration (if using local filesystem)
      FILE_STORAGE_BASE_PATH: ${FILE_STORAGE_BASE_PATH:-/app/storage}
      SENTRY_ENABLED: ${SENTRY_ENABLED:-false}
      SENTRY_DSN: ${SENTRY_DSN_BLOBPATH:-}
    volumes:
      - blobpath-storage:/app/storage
    networks:
      - ib-network
    ports:
      - "8086:8086"
    depends_on:
      - minio
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8086/blobpath/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  settings-service:
    build:
      context: ./pkf-ib-be-settings-service
      dockerfile: Dockerfile
    container_name: ib-settings-service
    restart: unless-stopped
    environment:
      PORT: 8888
      DB_URL: jdbc:postgresql://postgres:5432/settings_db
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      USER_URI: http://user-service:8084
      SENTRY_ENABLED: ${SENTRY_ENABLED:-false}
      SENTRY_DSN_SETTINGS: ${SENTRY_DSN_SETTINGS:-}
    networks:
      - ib-network
    ports:
      - "8888:8888"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8888/settings/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  gateway-service:
    build:
      context: ./pkf-ib-be-gateway-service
      dockerfile: Dockerfile
    container_name: ib-gateway-service
    restart: unless-stopped
    environment:
      PORT: 8080
      AUTH_SERVICE_URL: http://auth-service:8082
      USER_SERVICE_URL: http://user-service:8084
      PAYMENT_SERVICE_URL: http://payment-service:8083
      TRANSACTION_SERVICE_URL: http://transaction-service:8081
      ADAPTER_SERVICE_URL: http://abs-adapter-service:8085
      BLOBPATH_SERVICE_URL: http://blobpath-service:8086
      SETTINGS_SERVICE_URL: http://settings-service:8888
      SENTRY_ENABLED: ${SENTRY_ENABLED:-false}
      SENTRY_DSN_GATEWAY: ${SENTRY_DSN_GATEWAY:-}
    networks:
      - ib-network
    ports:
      - "8080:8080"
    depends_on:
      - auth-service
      - user-service
      - payment-service
      - transaction-service
      - abs-adapter-service
      - blobpath-service
      - settings-service
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

**Usage:**
```bash
# Start all backend services
docker compose -f docker-compose.yml \
  -f docker-compose-backend.yml \
  -f docker-compose-backend-extended.yml \
  up -d

# Check gateway routing
curl http://localhost:8080/actuator/health
```

---

## A.4 Frontend Services (docker-compose-frontend.yml)

This file defines the frontend applications and NGINX reverse proxy.

```yaml
version: '3.8'

# Frontend services configuration
# Use with: docker-compose -f docker-compose.yml -f docker-compose-backend.yml -f docker-compose-backend-extended.yml -f docker-compose-frontend.yml up

services:
  # ==============================================
  # FRONTEND SERVICES
  # ==============================================

  fe-corporate-client:
    build:
      context: ./pkf-ib-fe-corporate-client
      dockerfile: Dockerfile
    container_name: ib-fe-corporate-client
    restart: unless-stopped
    environment:
      REACT_APP_API_BASE_URL: ${REACT_APP_API_BASE_URL}
      REACT_APP_WEB_CLIENT_BASE_URL: ${REACT_APP_WEB_CLIENT_BASE_URL}
      REACT_APP_WEB_CORPORATE_BASE_URL: ${REACT_APP_WEB_CORPORATE_BASE_URL}
      REACT_APP_SENTRY_DSN: ${REACT_APP_SENTRY_DSN:-}
      REACT_APP_INTERNAL_BANK: ${REACT_APP_INTERNAL_BANK}
      REACT_APP_DEFAULT_COUNTRY: ${REACT_APP_DEFAULT_COUNTRY:-ci}
      REACT_APP_BANK_PHONE_NUMBER: ${REACT_APP_BANK_PHONE_NUMBER}
      REACT_APP_BANK_WHATSAPP_NUMBER: ${REACT_APP_BANK_WHATSAPP_NUMBER:-}
      REACT_APP_OTP_CODE_LIFE_TIME: ${REACT_APP_OTP_CODE_LIFE_TIME:-300000}
      REACT_APP_FEATURE_NRA_DISABLE: ${REACT_APP_FEATURE_NRA_DISABLE:-false}
      REACT_APP_FEATURE_BANNER_DISABLE: ${REACT_APP_FEATURE_BANNER_DISABLE:-false}
      REACT_APP_FEATURE_CARD_DISABLE: ${REACT_APP_FEATURE_CARD_DISABLE:-false}
      REACT_APP_FEATURE_LOANS_DISABLE: ${REACT_APP_FEATURE_LOANS_DISABLE:-false}
      REACT_APP_FEATURE_EXCHANGES_DISABLE: ${REACT_APP_FEATURE_EXCHANGES_DISABLE:-false}
      REACT_APP_FEATURE_DEPOSITS_DISABLE: ${REACT_APP_FEATURE_DEPOSITS_DISABLE:-false}
      REACT_APP_FEATURE_SWIFT_DISABLE: ${REACT_APP_FEATURE_SWIFT_DISABLE:-false}
      REACT_APP_DEFAULT_CURRENCY: ${REACT_APP_DEFAULT_CURRENCY:-USD}
      REACT_APP_REFERENCE_CURRENCY: ${REACT_APP_REFERENCE_CURRENCY:-USD}
      REACT_APP_ACCOUNT_LENGTH: ${REACT_APP_ACCOUNT_LENGTH:-15}
      REACT_APP_ACCOUNT_LENGTH_FIRST_PART: ${REACT_APP_ACCOUNT_LENGTH_FIRST_PART:-12}
    networks:
      - ib-network
    ports:
      - "3002:80"
    depends_on:
      - gateway-service

  fe-main-bo-client:
    build:
      context: ./pkf-ib-fe-main-bo-client
      dockerfile: Dockerfile
    container_name: ib-fe-main-bo-client
    restart: unless-stopped
    environment:
      VITE_API_BASE_URL: ${VITE_API_BASE_URL}
      VITE_SENTRY_DSN: ${VITE_SENTRY_DSN:-}
      VITE_INTERNAL_BANK: ${VITE_INTERNAL_BANK}
      VITE_DEFAULT_COUNTRY: ${VITE_DEFAULT_COUNTRY:-ci}
      VITE_BANK_PHONE_NUMBER: ${VITE_BANK_PHONE_NUMBER}
      VITE_DEFAULT_CURRENCY: ${VITE_DEFAULT_CURRENCY:-USD}
      VITE_REFERENCE_CURRENCY: ${VITE_REFERENCE_CURRENCY:-USD}
    networks:
      - ib-network
    ports:
      - "3004:80"
    depends_on:
      - gateway-service

  # ==============================================
  # NGINX REVERSE PROXY
  # ==============================================

  nginx-proxy:
    image: nginx:alpine
    container_name: ib-nginx-proxy
    restart: unless-stopped
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    networks:
      - ib-network
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - gateway-service
      - fe-corporate-client
      - fe-main-bo-client
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

**Usage:**
```bash
# Start complete stack
docker compose -f docker-compose.yml \
  -f docker-compose-backend.yml \
  -f docker-compose-backend-extended.yml \
  -f docker-compose-frontend.yml \
  up -d

Tool call argument 'replace' pruned from message history.

