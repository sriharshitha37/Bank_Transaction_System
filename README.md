# Bank Transaction System

![Node.js](https://img.shields.io/badge/Node.js-18+-339933?logo=node.js&logoColor=white)
![Express](https://img.shields.io/badge/Express-5.x-000000?logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose-47A248?logo=mongodb&logoColor=white)
![JWT](https://img.shields.io/badge/Auth-JWT-black?logo=jsonwebtokens)
![Deployed on Render](https://img.shields.io/badge/Deployed-Render-46E3B7?logo=render&logoColor=white)

A backend system for managing user accounts, ledger-based transactions, and balance tracking — built with Node.js, Express, and MongoDB. Includes JWT-based authentication with server-side token revocation, and automated email notifications via Gmail OAuth2.

**Live API:** [https://bank-transaction-system-ctgt.onrender.com](https://bank-transaction-system-ctgt.onrender.com)

> Note: Hosted on Render's free tier, so the first request after inactivity may take 30–60 seconds to respond while the service spins back up.

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [API Reference](#api-reference)
- [Authentication Flow](#authentication-flow)
- [Notes on Design](#notes-on-design)
- [Future Improvements](#future-improvements)

## Features

- **Authentication** — User registration and login secured with JWT and hashed passwords.
- **Account Management** — APIs to create and maintain user account data.
- **Transactions** — Credit and debit operations with:
  - **Idempotency validation** to prevent duplicate transaction processing.
  - **Status tracking** (e.g. transactions move through a `Pending` state before completion).
  - **Ledger-based recording** for consistency and auditability of all account activity.
- **Balance Inquiry** — Fetch a user's current account balance.
- **Initial Funding** — Mechanism to credit starting funds to new accounts, handled through a dedicated `System User` role for administrative oversight.
- **Email Notifications** — Automated emails (via Nodemailer + Gmail OAuth2) for:
  - Registration confirmation
  - Successful transactions
  - Failed transactions

## Architecture

<img width="1698" height="721" alt="diagram-export-30-06-2026-14_42_12" src="https://github.com/user-attachments/assets/bb4185d0-494e-41ae-b39e-d03264f11f0c" />

Requests flow through a standard layered structure: routes handle URL/method matching, middleware handles auth/authorization, controllers contain business logic, and models handle persistence via Mongoose. Two cross-cutting concerns sit outside this main flow — the blacklist check (logout revocation) and the email service (registration/transaction notifications) — keeping auth and notifications decoupled from core business logic.

## Tech Stack

- **Runtime:** Node.js
- **Framework:** Express
- **Database:** MongoDB (Mongoose)
- **Auth:** JSON Web Tokens (JWT)
- **Email:** Nodemailer with Gmail OAuth2
- **Deployment:** Render

## Project Structure

```
src/
├── config/
│   └── db.js                    # MongoDB connection setup
├── controllers/
│   ├── account.controller.js
│   ├── auth.controller.js
│   └── transaction.controller.js
├── middleware/
│   └── auth.middleware.js       # JWT verification / route protection
├── models/
│   ├── account.model.js
│   ├── blackList.model.js       # Invalidated/blacklisted JWTs (logout handling)
│   ├── ledger.model.js
│   ├── transaction.model.js
│   └── user.model.js
├── routes/
│   ├── account.routes.js
│   ├── auth.routes.js
│   └── transaction.routes.js
├── services/
│   └── email.service.js         # Nodemailer + Gmail OAuth2 integration
└── app.js                       # Express app setup
server.js                        # Entry point
```

## Getting Started

### Prerequisites
- Node.js (v18+ recommended)
- A MongoDB instance (local or Atlas)
- A Gmail account configured for OAuth2 (Client ID, Client Secret, Refresh Token)

### Installation

```bash
git clone https://github.com/sriharshitha37/Bank_Transaction_System.git
cd Bank_Transaction_System
npm install
```

### Environment Variables

Create a `.env` file in the root directory with the following:

```
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret
CLIENT_ID=your_google_oauth_client_id
CLIENT_SECRET=your_google_oauth_client_secret
REFRESH_TOKEN=your_google_oauth_refresh_token
EMAIL_USER=your_gmail_address
```

### Run Locally

```bash
npm start
```

The server will start on the configured port (check `server.js` for the default).

## API Reference

Base URL (local): `http://localhost:<PORT>/api`
Base URL (live): `https://bank-transaction-system-ctgt.onrender.com/api`

### Auth — `/api/auth`

| Method | Endpoint | Description | Protected |
|---|---|---|---|
| POST | `/register` | Register a new user | No |
| POST | `/login` | Log in and receive a JWT | No |
| POST | `/logout` | Log out — invalidates the current JWT via blacklist | Yes |

### Accounts — `/api/accounts`

| Method | Endpoint | Description | Protected |
|---|---|---|---|
| POST | `/` | Create a new account for the logged-in user | Yes |
| GET | `/` | Get all accounts belonging to the logged-in user | Yes |
| GET | `/balance/:accountId` | Get the current balance of a specific account | Yes |

### Transactions — `/api/transactions`

| Method | Endpoint | Description | Protected |
|---|---|---|---|
| POST | `/` | Create a new credit/debit transaction | Yes (user) |
| POST | `/system/initial-funds` | Credit initial funds to an account, restricted to the System User role | Yes (system user) |

> "Protected" routes require a valid JWT, sent via cookie (`token`) and verified by `auth.middleware.js`. Routes marked "system user" use a separate `authSystemUserMiddleware`, restricting access to an administrative System User role.

## Authentication Flow

Authentication uses JWTs with a database-backed blacklist for logout, rather than relying on short token expiry alone:

1. **Register / Login** — On successful login, the server signs a JWT (`userId` payload) and sends it to the client as an HTTP-only cookie.
2. **Protected requests** — `auth.middleware.js` reads the token from the cookie, verifies it against `JWT_SECRET`, and checks it against the `blackList` collection. If the token is valid and not blacklisted, the request proceeds with the decoded user attached to `req`.
3. **Logout** — Instead of just clearing the client-side cookie (which doesn't actually invalidate a still-valid JWT), the token is written to the `blackList` model in MongoDB. Any subsequent request using that same token is rejected by the middleware, even if it hasn't expired yet.
4. **System User role** — A separate `authSystemUserMiddleware` gates administrative-only routes (like crediting initial funds), checking for a system-level role rather than just a valid session.

This blacklist approach solves a common problem with stateless JWTs: normally, once a token is issued, there's no way to revoke it before it expires. Storing logged-out tokens server-side closes that gap, at the cost of an extra DB lookup per authenticated request — a deliberate consistency-over-performance tradeoff for a financial system.

## Notes on Design

- **Idempotency** is enforced on transaction requests to avoid double-processing in case of retries or network failures — a common real-world banking system requirement.
- **Ledger model** keeps a consistent, auditable record of all balance-affecting operations rather than mutating account balances directly.
- **System User role** allows administrative crediting of initial funds without requiring a separate funding service.

## Future Improvements
- **Transaction history endpoint** — Currently there's no dedicated route to list a user's past transactions; the ledger model supports it, but it isn't exposed yet.
- **API documentation** — Generate interactive docs (e.g. Swagger/OpenAPI) instead of relying solely on this README.
- **Dockerization** — Add a `Dockerfile` and `docker-compose.yml` for easier local setup without manually configuring MongoDB.
