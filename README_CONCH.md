# Conch - Immutable Data Management System

A sophisticated full-stack system for managing immutable, versioned data with provenance tracking, authentication, and real-time subscriptions.

## Project Structure

```
conch-monorepo/
├── backend/                    # Rust Axum backend
│   ├── src/
│   │   ├── main.rs            # Server entry point
│   │   ├── models.rs          # Conch core model + types
│   │   ├── db.rs              # Database queries
│   │   ├── handlers/          # API route handlers
│   │   ├── auth.rs            # JWT auth + permission engine
│   │   ├── ws.rs              # WebSocket handlers
│   │   ├── events.rs          # Event emitter integration
│   │   └── error.rs           # Error handling
│   ├── migrations/            # SQL migrations
│   ├── Cargo.toml
│   ├── Dockerfile
│   └── .env.example
├── app/                       # Next.js frontend
├── components/
├── docker-compose.yml         # Full stack orchestration
└── README_CONCH.md           # This file

## Tech Stack

- **Backend**: Rust + Axum framework
- **Database**: PostgreSQL with SQLx
- **Event Streaming**: Redis (pub/sub)
- **Frontend**: Next.js 16 + Tailwind CSS
- **Authentication**: JWT-based with custom permission engine
- **Real-time**: WebSocket subscriptions
- **Deployment**: Docker + Docker Compose

## Quick Start - Local Development

### Prerequisites

- Docker and Docker Compose installed
- Rust 1.75+ (if building backend locally)
- Node.js 18+ (if running frontend locally)
- pnpm (for frontend)

### Option 1: Full Stack with Docker Compose

```bash
# Clone or navigate to project
cd conch-monorepo

# Copy environment file
cp backend/.env.example backend/.env

# Start all services
docker-compose up -d

# Check logs
docker-compose logs -f backend
docker-compose logs -f postgres
docker-compose logs -f redis
```

The system will be available at:
- Frontend: http://localhost:3000
- Backend API: http://localhost:3001
- Health check: http://localhost:3001/health

### Option 2: Local Backend Development

```bash
# Navigate to backend
cd backend

# Install Rust (if not already installed)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Copy .env file
cp .env.example .env

# Start PostgreSQL and Redis with Docker
docker-compose up postgres redis -d

# Run migrations (they run automatically on server start)
cargo build

# Run the backend
cargo run
```

The backend API will be available at http://localhost:3001

### Option 3: Local Frontend Development

```bash
# Navigate to project root
cd ..

# Install dependencies
pnpm install

# Create .env.local
echo "NEXT_PUBLIC_API_URL=http://localhost:3001" > .env.local

# Start development server
pnpm dev
```

The frontend will be available at http://localhost:3000

## API Endpoints

### Authentication

```bash
# Register new user
POST /auth/register
{
  "username": "user123",
  "email": "user@example.com",
  "password": "secure_password"
}

# Login
POST /auth/login
{
  "email": "user@example.com",
  "password": "secure_password"
}
```

### Conch Management

```bash
# Create a conch
POST /conches
{
  "state": { "name": "My Conch", "data": "..." }
}

# List all conches
GET /conches

# Get specific conch
GET /conches/:id

# Update conch (increments era, appends lineage)
PUT /conches/:id
{
  "state": { "name": "Updated Conch", "data": "..." }
}

# Delete conch
DELETE /conches/:id

# Get conch lineage history
GET /conches/:id/lineage
```

### Graph Links

```bash
# Create link between conches
POST /links/:source_id
{
  "target_id": "uuid",
  "link_type": "references",
  "metadata": { "strength": "strong" }
}

# Get neighborhood (graph query)
GET /conches/:id/neighborhood
```

### Real-time

```bash
# WebSocket subscription
WS /ws/:conch_id

# Emits StateChangeEvent on updates:
{
  "conch_id": "uuid",
  "era": 1,
  "old_state": { ... },
  "new_state": { ... },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## Database Schema

### Conches Table
Stores immutable versioned data with cryptographic signatures

```sql
CREATE TABLE conches (
  id UUID PRIMARY KEY,
  owner_id UUID NOT NULL,
  era BIGINT NOT NULL DEFAULT 0,
  state JSONB NOT NULL,
  signature TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### Conch Lineage
Tracks complete change history

```sql
CREATE TABLE conch_lineage (
  id BIGSERIAL PRIMARY KEY,
  conch_id UUID NOT NULL REFERENCES conches(id),
  era BIGINT NOT NULL,
  state_hash TEXT NOT NULL,
  timestamp TIMESTAMP DEFAULT NOW()
);
```

### Conch Links
Graph relationships between conches

```sql
CREATE TABLE conch_links (
  id UUID PRIMARY KEY,
  source_id UUID NOT NULL REFERENCES conches(id),
  target_id UUID NOT NULL REFERENCES conches(id),
  link_type VARCHAR(50),
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);
```

## Authentication & Permissions

The system uses JWT-based authentication with fine-grained permission control:

- **Owner**: Full access to conch (read, write, delete)
- **Allowlist**: Specific users granted access
- **Scopes**: Read-only or read-write access
- **Roles**: Support for future role-based access control

Permission checks are enforced in middleware for all routes.

## Key Features

### Immutability & Provenance
- Every state change increments the era counter
- Complete lineage of all changes with state hashes
- Cryptographic signatures for integrity verification
- Full audit trail for compliance

### Real-time Subscriptions
- WebSocket connections for live updates
- Redis pub/sub for efficient event distribution
- Automatic connection cleanup on disconnect
- Scalable to thousands of subscribers

### Graph Relationships
- Define links between conches with metadata
- Recursive neighborhood queries with depth control
- Support for complex data relationships
- Efficient CTE-based queries

## Deployment

### Cloud Deployment Options

#### 1. Railway.app (Recommended)
```bash
# Connect your GitHub repository
# Add environment variables in Railway dashboard
# Deploy from UI

Environment variables:
- DATABASE_URL
- REDIS_URL
- JWT_SECRET
```

#### 2. Fly.io
```bash
# Install Fly CLI
curl -L https://fly.io/install.sh | sh

# Login
flyctl auth login

# Deploy
flyctl launch
flyctl deploy
```

#### 3. Heroku
```bash
# Install Heroku CLI
# Login
heroku login

# Create app
heroku create your-app-name

# Add PostgreSQL addon
heroku addons:create heroku-postgresql:standard-0

# Add Redis addon
heroku addons:create heroku-redis:premium-0

# Deploy
git push heroku main
```

### Docker Production Build

```bash
# Build for production
docker build -f backend/Dockerfile -t conch-backend:latest ./backend

# Run with environment variables
docker run -e DATABASE_URL=$DATABASE_URL \
           -e REDIS_URL=$REDIS_URL \
           -e JWT_SECRET=$JWT_SECRET \
           -p 3001:3001 \
           conch-backend:latest
```

## Environment Variables

Required for production:

```
DATABASE_URL=postgresql://user:password@host:5432/conch_db
REDIS_URL=redis://default:password@host:6379
JWT_SECRET=use-strong-random-string-here
RUST_LOG=info
RUST_ENV=production
```

## Development

### Running Tests

```bash
cd backend
cargo test
```

### Code Quality

```bash
# Format code
cargo fmt

# Lint
cargo clippy

# Build docs
cargo doc --open
```

### Database Migrations

Migrations in `backend/migrations/` run automatically on server startup.

To add a new migration:
1. Create file: `backend/migrations/002_your_change.sql`
2. Write SQL
3. Restart server

## Troubleshooting

### PostgreSQL Connection Error
```
Make sure postgres is running:
docker ps | grep postgres

If not running:
docker-compose up postgres -d
```

### Redis Connection Error
```
Check Redis status:
redis-cli ping

Should return: PONG
```

### Port Already in Use
```
Change ports in docker-compose.yml or:
lsof -i :3001
kill -9 <PID>
```

### Build Errors in Backend

```bash
# Clean build
cargo clean
cargo build

# Update dependencies
cargo update
```

## Security Considerations

- All passwords hashed with bcrypt
- JWT tokens expire after 24 hours (configurable)
- Row-level security via permission checks
- SQL injection prevention via SQLx compile-time checks
- CORS configured for frontend origin
- HTTPS enforced in production (add in nginx/reverse proxy)

## Performance Optimization

- Connection pooling for database (max 5 connections)
- Redis caching for frequently accessed conches
- Indexed queries on owner_id, source_id, target_id
- Efficient lineage storage with era-based queries
- WebSocket direct connections minimize latency

## License

MIT

## Support

For issues and questions, please open an issue in the repository.

## Roadmap

- [ ] Step 4-6: Full authentication and CRUD implementation
- [ ] Step 7-8: Graph queries and real-time subscriptions
- [ ] Step 9-10: SDK generation and frontend portal
- [ ] Step 11: Production deployment configuration
- [ ] [ ] GraphQL API support
- [ ] [ ] Full-text search
- [ ] [ ] Time-travel queries
- [ ] [ ] Batch operations
