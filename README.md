# MagicBlock Console - System Architecture & Workflow

A comprehensive guide to the MagicBlock Console architecture, designed to provide a seamless developer experience for deploying and managing Solana ephemeral rollups.

## 🌟 System Overview

The MagicBlock Console is a **developer platform aimed at simplifying the use of Ephemeral Rollups on Solana**. It abstracts away the complexity of infrastructure management, providing a unified interface for:

1.  **Project Management**: Create, configure, and monitor projects.
2.  **Rollup Orchestration**: One-click provisioning of dedicated ephemeral rollups.
3.  **SDK Integration**: Auto-generated code snippets for easy integration.
4.  **Real-Time Monitoring**: Live transaction feeds and analytics.

---

## 🏗️ Architecture Diagram

```mermaid
graph TD
    subgraph "Developer's App"
        App[Game / dApp]
        SDK["@magicblock-console/sdk"]
        App -- Uses --> SDK
    end

    subgraph "MagicBlock Console Platform"
        FE[Console Dashboard<br/>(Vite + React + shadcn)]
        API[Control Plane API<br/>(Go + Gin)]
        DB[(PostgreSQL<br/>User & Project Data)]
        Redis[(Redis<br/>Cache & PubSub)]
        
        FE -- HTTP/WS --> API
        API -- Read/Write --> DB
        API -- PubSub --> Redis
    end

    subgraph "Infrastructure Layer"
        MB_API[MagicBlock Cloud API]
        Rollup[Ephemeral Rollup Instance]
        Solana[Solana L1]

        API -- Provisions --> MB_API
        MB_API -- Spawns --> Rollup
        SDK -- Sends Txs --> Rollup
        Rollup -- Settles on --> Solana
        Rollup -- Streams Logs --> API
    end
```

---

## 🔄 Detailed Workflow

### 1. **User Onboarding (Authentication)**
*   **Action**: User connects their Solana wallet (Phantom, Backpack, etc.) on the landing page.
*   **Mechanism**:
    1.  Frontend requests a signature for a nonce.
    2.  `POST /auth/wallet` sends signature to Go backend.
    3.  Backend verifies signature against public key.
    4.  If valid, issues a **JWT** (JSON Web Token) for session management.
    5.  User is redirected to `/dashboard`.

### 2. **Project Creation & Configuration**
*   **Action**: User clicks "New Project", names it (e.g., "MySolanaGame"), and selects a region.
*   **Mechanism**:
    1.  `POST /projects` creates a record in PostgreSQL.
    2.  Backend generates a unique **API Key** for this project.
    3.  Backend initiates a request to MagicBlock's infrastructure to **provision a dedicated rollup**.
    4.  Initial status is set to `PROVISIONING`.

### 3. **SDK Integration**
*   **Action**: User copies the generated code snippet from the console.
*   **Code Example**:
    ```typescript
    import { MagicBlockConsole } from '@magicblock-console/sdk';

    const magic = new MagicBlockConsole({
      apiKey: "MB_pk_12345...",
      environment: "devnet" 
    });
    ```
*   **Mechanism**:
    *   The SDK initializes and fetches the specific rollup endpoint for the project from the Control Plane API.

### 4. **Runtime Execution (The "Magic")**
*   **Action**: User's app sends a transaction using the SDK.
*   **Mechanism**:
    1.  **Delegate**: The SDK handles the "delegation" of specific accounts (e.g., a game state PDA) from Solana L1 to the Ephemeral Rollup.
    2.  **Execute**: App sends transactions directly to the high-speed Rollup RPC. These are gasless and instant.
    3.  **Monitor**: The SDK emits events locally, and the Control Plane API listens for these actions via webhook/websocket from the Rollup.

### 5. **Real-Time Monitoring**
*   **Action**: User watches the "Transactions" tab in the Console.
*   **Mechanism**:
    1.  The Rollup pushes transaction logs to the Control Plane API.
    2.  The API processes and stores them in PostgreSQL.
    3.  The API broadcasts updates via WebSocket (`/ws/transactions`) to the frontend.
    4.  The Console UI updates instantly to show "Transaction Confirmed".

### 6. **Settlement (Commit)**
*   **Action**: The session ends or a periodic commit triggers.
*   **Mechanism**:
    *   The Rollup computes the state difference (diff).
    *   A "Commit" transaction is sent to Solana L1, updating the master state.
    *   Accounts are "undelegated" and released back to L1 control if desired.

---

## 🛠️ Technology Stack Breakdown

| Component | Technology | Reasoning |
| :--- | :--- | :--- |
| **Frontend** | **Vite + React** | Fast build times, modern developer experience. |
| **UI Library** | **shadcn/ui + Tailwind** | Accessible, copy-paste components, highly customizable. |
| **Backend** | **Go (Golang)** | High performance, excellent concurrency for handling WebSocket streams. |
| **Web Framework** | **Gin** | Lightweight, fast HTTP web framework for Go. |
| **Database** | **PostgreSQL** | Robust relational data integrity for user/project data. |
| **ORM** | **GORM** | Developer-friendly abstraction for database operations in Go. |
| **Infrastructure** | **Docker Compose** | Simple, reproducible local development environment. |
| **Live Updates** | **WebSockets (Gorilla)** | Real-time push for transaction logs and status. |

---

## 📂 Project Structure (Monorepo)

```
/
├── apps/
│   ├── console/              # The Frontend Dashboard
│   │   ├── src/components/ui # shadcn components
│   │   └── ...
│   │
│   └── api/                  # The Backend Control Plane
│       ├── cmd/server        # Entry point
│       ├── internal/models   # GORM Database Models
│       ├── internal/handlers # HTTP Route Controllers
│       └── ...
│
├── packages/
│   └── sdk/                  # The Developer SDK
│       └── ...
│
├── docker-compose.yml        # DB & Redis orchestration
└── Makefile                  # Automation scripts
```
