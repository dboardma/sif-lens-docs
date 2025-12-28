![TypeScript](https://img.shields.io/badge/TypeScript-Strict-blue)
![License](https://img.shields.io/badge/License-Proprietary-red)
# SIF Lens - Full-Stack TypeScript Application

A full-stack application for analyzing SIF API data consistency, with a Node.js/Express backend and React frontend.

## Repository Status and Licensing

This repository is published publicly for **transparency, technical review, and evaluation**.

No rights are granted for commercial use, redistribution, or deployment without a separate written agreement from the author.

This codebase is actively developed and may change without notice.

See the [LICENSE](LICENSE) file for usage and redistribution terms.

## Project Structure

```
SIFDataAnalysis/
├── server/              # Backend (Node.js/Express/TypeScript)
│   ├── src/
│   │   ├── client/      # ApiClient.ts
│   │   ├── analyzer/    # SchemaAnalyzer.ts
│   │   ├── database/    # DatabaseWriter.ts
│   │   ├── manager/     # ScanManager.ts
│   │   └── Server.ts    # Express server
│   ├── init_database.sql # MySQL database schema
│   └── package.json
└── client/              # Frontend (React/Vite/TypeScript)
    ├── src/
    │   ├── App.tsx
    │   └── App.css
    └── package.json
```

## Backend Architecture

### 1. **ApiClient** (`server/src/client/ApiClient.ts`)
- Handles SIF API authentication with HMAC-SHA256
- Implements pagination with `navigationPage` header
- Streams student records as an async generator
- Supports dynamic solution selection (prod/uat)
- No knowledge of analysis or database logic

### 2. **SchemaAnalyzer** (`server/src/analyzer/SchemaAnalyzer.ts`)
- Pure data processing - no API or database calls
- Recursively traverses JSON structures
- Filters fields starting with uppercase letters
- Records all hierarchy levels
- Uses extensible factory pattern for field extraction
- Collects up to 5 unique sample values per field

#### Field Extractor Pattern
The analyzer uses a factory pattern to handle different field types:
- **StandardFieldExtractor** - Handles normal fields
- **SIFExtendedElementExtractor** - Processes `SIFExtendedElements.SIFExtendedElement` arrays
  - Extracts individual fields from each array element
  - Uses `-Name` attribute as field name
  - Uses `#text` attribute as field value
- **Custom Extractors** - Easy to add new extractors (see `server/src/analyzer/extractors/README.md`)

### 3. **DatabaseWriter** (`server/src/database/DatabaseWriter.ts`)
- Handles MySQL persistence with transactions
- Upserts field statistics
- Stores sample values with source RefIds
- Connection pooling for efficiency

### 4. **ScanManager** (`server/src/manager/ScanManager.ts`)
- Orchestrates the entire scan workflow
- Creates and tracks scan sessions in the database
- Coordinates ApiClient → SchemaAnalyzer → DatabaseWriter
- Handles multiple zone IDs sequentially
- Marks scans as completed or failed
- Optional history preservation (default: delete previous scans)

### 5. **Server** (`server/src/Server.ts`)
- Express REST API
- `POST /api/scan` - Trigger analysis (async)
- `GET /api/results/:zoneId` - Retrieve results

## Setup Instructions

### Docker Setup (Recommended for Local Development)

For local development, use Docker to run the database and server. See [DOCKER.md](DOCKER.md) for complete instructions.

**Quick Start with Docker:**
```bash
# Linux/macOS: Start database and server
cp .env.docker.example .env.docker
# Windows (PowerShell): Copy-Item .env.docker.example .env.docker
# Windows (CMD): copy .env.docker.example .env.docker

# Edit .env.docker with your credentials
docker-compose --env-file .env.docker up -d

# Start client (separate terminal)
cd client
npm install
npm run dev
```

**Windows Users:** See [DOCKER.md](DOCKER.md) for platform-specific commands and WSL2 recommendations.

Access the application at http://localhost:5173

### AWS Production Deployment

For production deployments to AWS:
- **Client**: S3 + CloudFront (static hosting)
- **Server**: ECS Fargate (Docker container)
- **Database**: RDS MySQL

Refer to your deployment tool documentation for AWS deployment instructions.

### Manual Setup

#### Prerequisites
- Node.js 18+ and npm
- MySQL 8.0+
- SIF API credentials

### 1. Database Setup

```bash
# Create database and tables
mysql -u root -p < server/init_database.sql
```

### 2. Backend Setup

```bash
cd server

# Install dependencies
npm install

# Configure environment
# Linux/macOS:
cp .env.example .env
# Windows (PowerShell): Copy-Item .env.example .env
# Windows (CMD): copy .env.example .env
# Edit .env with your actual credentials:
# .env.example contains placeholder values only - you must provide:
#   DB_HOST=localhost
#   DB_USER=root
#   DB_PASSWORD=<your_password>
#   DB_NAME=siflens
#   JWT_SECRET=<generate_secure_random_string>
# Note: SIF API credentials are configured in the app's Settings page

# Run in development mode
npm run dev

# Or build and run in production
npm run build
npm start
```

The server will start on `http://localhost:3001`

### 3. Frontend Setup

```bash
# Linux/macOS: Run the setup script to install dependencies
chmod +x setup-client.sh
./setup-client.sh

# Windows or manual setup:
cd client
npm install

# Start development server
npm run dev
```

The client will start on `http://localhost:5173`

## Usage

### Starting a Scan

1. Open the frontend at `http://localhost:5173`
2. Select the solution (Production or UAT) from the dropdown
3. Enter zone IDs in the text area:
   - Full format: `zone1Id` or `zone2Id`
   - Comma-separated for multiple zones
4. (Optional) Check "Keep previous scan history" to preserve historical data
   - **Default**: Previous scans for each zone are deleted
   - **Checked**: Historical scans are preserved
5. Click "Start Scan"
6. The scan runs in the background on the server

### Viewing Results

1. Enter a zone ID in the "View Results" section
2. Click "Fetch Results"
3. View field statistics, data types, and sample values

### API Endpoints

**Start Scan:**
```bash
# Previous scans for a zone-recordtype deleted
curl -X POST http://localhost:3001/api/scan \
  -H "Content-Type: application/json" \
  -d '{"zoneIds": ["zone1Id", "zone2Id"], "solution": "prod"}'
```

**Get Results:**
```bash
curl http://localhost:3001/api/results/zone1Id
```

**Health Check:**
```bash
curl http://localhost:3001/api/health
```
## Features

✅ Multi-solution support (Production and UAT)  
✅ Automatic SAUID to zone ID conversion  
✅ Solution-specific API credentials  
✅ Async streaming of API data (handles large datasets)  
✅ Recursive schema analysis with hierarchy tracking  
✅ Extensible field extractor pattern for custom processing  
✅ Special handling for SIFExtendedElements  
✅ Uppercase field filtering  
✅ Transaction-based database writes  
✅ Clean separation of concerns  
✅ TypeScript throughout

## Testing

The project includes comprehensive test suites for both server and client covering critical security and provisioning functionality.

### Prerequisites for Testing

- **Docker**: Required for server integration tests (uses Testcontainers for MySQL)
- **IMPORTANT**: Ensure Docker Desktop (or Docker daemon) is running before executing server tests
- If Docker is not running, server tests will be skipped (shown as "skipped" in test output)

### Server Tests

Server tests use Vitest + Supertest + Testcontainers for integration testing.

```bash
cd server

# Run all tests
npm test

# Run tests in watch mode
npm run test:watch

# Run tests with UI
npm run test:ui
```

**Test Coverage:**
- Authentication (JWT login, token validation)
- Authorization & Segregation (platform vs tenant access)
- System Organizations (provisioning with admin users)
- Permission Enforcement (records:analyze, org:admin)

**Note:** Server tests automatically start a MySQL container using Testcontainers, run migrations, and clean up after tests complete. First run may take longer while Docker pulls the MySQL image.

### Client Tests

Client tests use Vitest + React Testing Library + MSW for component and integration testing.

```bash
cd client

# Run all tests
npm test

# Run tests in watch mode
npm run test:watch

# Run tests with UI
npm run test:ui
```

**Test Coverage:**
- Navigation Segregation (system admin vs tenant UI)
- System Administration Component (org list and creation)
- Tenant User UI (data analyst and admin navigation)

### Running All Tests

From the project root:

```bash
# Run server tests
cd server && npm test

# Run client tests
cd ../client && npm test
```

## Development

**Lint and type-check:**
```bash
cd server
npm run build  # TypeScript compilation will catch errors
```

## Troubleshooting

**Connection errors:**
- Verify MySQL is running: `mysql.server status`
- Check credentials in `.env`
- Ensure database exists: `SHOW DATABASES;`

**API authentication errors:**
- Verify SIF API credentials are configured in the Settings page
- Check timestamp synchronization
- Ensure you're using the correct solution in the UI

**CORS errors:**
- Ensure server is running on port 3001
- Check `API_BASE_URL` in `client/src/App.tsx`

## Data Handling and Privacy Posture

SIF Lens is designed to analyze **schema structure and metadata**, not to act as a data warehouse.

Key privacy characteristics:
- Full records are processed **in memory only**
- Records are **never persisted**.
- No relational reconstruction of records is possible from stored data
- All API access is explicitly directed by the user

Responsibility for data access authorization remains with the deploying organization.
_Synced from sif-lens commit 
