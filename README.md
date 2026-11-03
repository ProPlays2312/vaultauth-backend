# VaultAuth SSO — Backend & Identity Service

> **High-Assurance Zero-Trust Authentication API with Argon2id Hashing, Multi-Role PostgreSQL, Email 2FA OTP, and 1-Click Session Revocation.**

[![Project Status](https://img.shields.io/badge/Project%20Status-Active%20Development-blue.svg)](https://github.com/ABhardwaj28/vaultauth-backend)
[![PRD Version](https://img.shields.io/badge/PRD-v2.0%20(Optimized)-success.svg)](docs/PRD_v2.0.md)
[![Track](https://img.shields.io/badge/Track-PST__PD__154%20Product%20Development-orange.svg)]()
[![Batch](https://img.shields.io/badge/Batch-2026%20Level%204%20(Stretch)-purple.svg)]()
[![Frontend Client](https://img.shields.io/badge/Frontend%20Client-vaultauth--frontend-blueviolet.svg)](https://github.com/ABhardwaj28/vaultauth-frontend)
[![Deployment](https://img.shields.io/badge/Deployed%20on-Render%20Free%20Tier-46E3B7.svg)]()

---

## 1. Executive Summary

**VaultAuth SSO (Backend)** is a hardened identity and session management microservice developed in Python with **FastAPI** and **PostgreSQL**. Designed for enterprise-grade Single Sign-On (SSO) without the bloat of legacy OAuth brokers, it enforces:
- **Argon2id Server-Side Password Hashing:** Memory-hard cryptographic hashing via `argon2-cffi` ensuring passwords are never stored or logged in plaintext.
- **Out-of-Band 2FA OTP Delivery:** Challenge-based 6-digit numeric passcodes dispatched via **Resend** (with 10-minute expiry and 3-attempt invalidation).
- **Opaque 256-Bit Bearer Session Tokens:** Cryptographically random session identifiers (only their SHA-256 hash is persisted in the database).
- **Instant Session Revocation:** 1-click token invalidation evaluated per-request in PostgreSQL (< 30s end-to-end rejection).
- **Brute-Force Lockout Defense:** 3 consecutive failed login attempts trigger an atomic 5-minute account lock.
- **Principle of Least Privilege Database Roles:** Private schema `vaultauth` segregated across `vault_app` (DML only), `vault_migrator` (DDL/owner), and `vault_backup` (read-only).

---

## 2. Team & Project Roles

- **Backend & DevSecOps Lead:** Shaurya Jain ([@ProPlays2312](https://github.com/ProPlays2312))
- **Frontend & Client Security Lead:** Apoorva Bhardwaj ([@ABhardwaj28](https://github.com/ABhardwaj28))
- **Course Mentor:** Mahati Ma'am
- **Program:** Polaris School of Technology — On the Job Training (OJT), Semester 1

---

## 3. Product Requirements Document (PRD v2.0)

This backend implements the approved **VaultAuth SSO PRD v2.0**:
- 📄 **[Read PRD v2.0 (Markdown)](docs/PRD_v2.0.md)**
- 📥 **[Download PRD v2.0 (Compiled PDF)](docs/VaultAuth_SSO_PRD_v2.0.pdf)**

---

## 4. System Architecture & Topology

```text
+----------------------------------------------------------------------------------------+
|                                    VaultAuth SSO                                       |
|                                                                                        |
|   +------------------------------------+      +------------------------------------+   |
|   |         vaultauth-frontend         |      |           Expo Mobile ID           |   |
|   |    Static SPA on Vercel (CSP)      |      |     Corporate Digital ID Card      |   |
|   +------------------------------------+      +------------------------------------+   |
|                     |                                           |                      |
|                     +---------------------+---------------------+                      |
|                                           | HTTPS (Bearer Token)                       |
|                                           v                                            |
|                        +-------------------------------------+                         |
|                        |          FastAPI Backend            |                         |
|                        |       Container on Render Free      |                         |
|                        |  • Argon2id Hashing                 |                         |
|                        |  • 2FA Challenge State Engine       |                         |
|                        |  • Atomic Lockout Serializer        |                         |
|                        +-------------------------------------+                         |
|                                  /               \                                     |
|              vault_app Role     /                 \  Transactional Email               |
|            Transaction Pooler  /                   \                                   |
|                               v                     v                                  |
|               +-----------------------+     +-------------------+                      |
|               |  Supabase PostgreSQL  |     |    Resend API     |                      |
|               |  Private 'vaultauth'  |     |  Email 6-digit    |                      |
|               |  Schema               |     |  Verification/OTP |                      |
|               +-----------------------+     +-------------------+                      |
|                           |                                                            |
|          Nightly Sync     | pg_dump (vault_backup) -> psql (vault_migrator)            |
|                           v                                                            |
|               +-----------------------+                                                |
|               |     Neon Standby      |                                                |
|               |   Disaster Recovery   |                                                |
|               +-----------------------+                                                |
+----------------------------------------------------------------------------------------+
```

---

## 5. API & Service Contract

Base Path: `/api/v1`. All endpoints accept and return JSON.

```text
POST   /api/v1/auth/register             # Public: Register email + password (account set to PENDING)
POST   /api/v1/auth/verify-email         # Public: Confirm 6-digit registration code -> ACTIVE
POST   /api/v1/auth/verify-email/resend  # Public: Resend code (capped at 3 per 15 min)
POST   /api/v1/auth/login                # Public: Verify password -> returns {state: "OTP_REQUIRED"}
POST   /api/v1/auth/2fa/verify           # Public: Validate 6-digit code -> mints Bearer token
POST   /api/v1/auth/logout               # Authenticated: Invalidate current session token
GET    /api/v1/me                        # Authenticated: Current user profile
GET    /api/v1/me/sessions               # Authenticated: List current user's active sessions
DELETE /api/v1/sessions/{id}             # Authenticated: Revoke session (user: own, admin: any)
GET    /api/v1/admin/sessions            # Admin Only: List all active sessions across system
GET    /api/v1/admin/audit-logs          # Admin Only: Query immutable security audit events
POST   /api/v1/admin/users/{id}/unlock   # Admin Only: Unlock 3-strike locked account
GET    /api/v1/health                    # Open: Keepalive ping and database connectivity check
```

### Error Contract
All failures follow a standardized envelope:
```json
{
  "error": {
    "code": "ERR_INVALID_CREDENTIALS",
    "message": "Invalid email or password",
    "request_id": "c8f9b1d2-..."
  }
}
```
*Note: In accordance with `BR-05`, login failures return uniform 401 `ERR_INVALID_CREDENTIALS` regardless of whether the email exists, the password is wrong, or the account is locked, preventing user enumeration.*

---

## 6. Database Privileges & Roles

The system operates across three separate PostgreSQL database roles:

1. **`vault_app` (Runtime Service):**
   - Has `SELECT`, `INSERT`, `UPDATE`, `DELETE` on tables in schema `vaultauth`.
   - Explicitly denied `CREATE`, `DROP`, `ALTER`, or `TRUNCATE`.
2. **`vault_migrator` (Migrations & CLI):**
   - Schema owner. Runs Alembic migrations and the administrative `reset.py` CLI utility.
3. **`vault_backup` (Disaster Recovery):**
   - Has `SELECT` rights only. Used by CI/CD to pipe table-scoped `pg_dump` snapshots directly to the Neon standby database.

---

## 7. Project Structure

```text
vaultauth-backend/
├── app/
│   ├── api/
│   │   ├── v1/
│   │   │   ├── auth.py          # Register, verify email, login, 2FA
│   │   │   ├── sessions.py      # Session listing & instant revocation
│   │   │   └── admin.py         # User management & audit log querying
│   │   └── deps.py              # Bearer auth & DB session dependencies
│   ├── core/
│   │   ├── config.py            # Pydantic Settings & environment variables
│   │   └── security.py          # Argon2id hashing & token generation
│   ├── db/
│   │   ├── session.py           # SQLAlchemy async engine & pooler config
│   │   └── base.py              # Declarative base & schema metadata
│   ├── models/                  # SQLAlchemy ORM models (User, Session, Challenge, Audit)
│   ├── schemas/                 # Pydantic request/response schemas
│   ├── services/
│   │   ├── email.py             # Resend email dispatcher
│   │   └── audit.py             # Security audit logger
│   └── main.py                  # FastAPI application entrypoint & middleware
├── alembic/                     # Database migration revisions
├── cli/
│   ├── promote_admin.py         # CLI bootstrap tool to promote admin
│   └── reset.py                 # CLI database reset tool (vault_migrator only)
├── tests/                       # Pytest test suite (critical-path auth tests)
├── Dockerfile                   # Production container definition
├── docs/
│   ├── PRD_v2.0.md              # Product Requirements Document (v2.0)
│   └── VaultAuth_SSO_PRD_v2.0.pdf
└── requirements.txt             # Python dependencies (fastapi, uvicorn, argon2-cffi, etc.)
```

---

## 8. Local Setup & Installation

```bash
# 1. Clone repository
git clone https://github.com/ABhardwaj28/vaultauth-backend.git
cd vaultauth-backend

# 2. Create virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set environment variables
cp .env.example .env

# 5. Run Alembic migrations
alembic upgrade head

# 6. Start development server
uvicorn app.main:app --reload --port 8000
```

---

## 9. Delivery Milestones

- [x] **Week 1:** Repository setup, PRD v2.0 architectural alignment, schema definition, and Dockerfile.
- [ ] **Week 2:** Argon2id hashing, email verification, login endpoint, and out-of-band 2FA OTP flow.
- [ ] **Week 3:** Session revocation engine, admin console routes, 30-day purge, and Render deployment.
- [ ] **Week 4:** Standby database sync verification, CI test suite execution, and end-to-end integration demo.

---

## 10. License

Licensed under the MIT License. Developed for academic demonstration at Polaris School of Technology.
