# Conch - Deployment Guide

This guide covers deploying the Conch system to various cloud platforms and on-premises environments.

## Local Development

### Prerequisites
- Docker and Docker Compose
- Rust 1.75+ (optional, for local backend development)
- Node.js 18+ (optional, for local frontend development)

### Quick Start with Docker Compose

```bash
# Clone the repository
git clone <your-repo>
cd conch-monorepo

# Copy environment file
cp backend/.env.example backend/.env

# Start all services
docker-compose up -d

# Check services
docker-compose logs -f
```

Services available at:
- Frontend: http://localhost:3000
- Backend API: http://localhost:3001
- PostgreSQL: localhost:5432
- Redis: localhost:6379

## Production Deployment

### Environment Variables

Create `.env.prod` file with:

```
# Database
DATABASE_URL=postgresql://user:password@host:5432/conch_db

# Redis
REDIS_URL=redis://:password@host:6379

# JWT
JWT_SECRET=your-strong-random-secret-key

# Frontend
NEXT_PUBLIC_API_URL=https://api.yourdomain.com

# Other
RUST_LOG=info
RUST_ENV=production
```

### Option 1: Railway.app (Recommended)

Railway is the easiest option for quick deployment:

1. **Create Railway account** at https://railway.app
2. **Connect GitHub repository**
3. **Create services**:
   - PostgreSQL (built-in)
   - Redis (built-in)
   - Custom service for backend
   - Custom service for frontend

4. **Configure environment variables** in Railway dashboard:
   - Set DATABASE_URL, REDIS_URL, JWT_SECRET
   - Set NEXT_PUBLIC_API_URL to your backend domain

5. **Deploy**:
   ```bash
   npm install -g @railway/cli
   railway login
   railway up
   ```

### Option 2: Fly.io

Fly.io provides global edge deployment:

1. **Install Fly CLI**:
   ```bash
   curl -L https://fly.io/install.sh | sh
   ```

2. **Create apps**:
   ```bash
   flyctl auth login
   flyctl apps create conch-backend
   flyctl apps create conch-frontend
   ```

3. **Deploy backend**:
   ```bash
   cd backend
   flyctl deploy
   ```

4. **Deploy frontend**:
   ```bash
   cd ..
   flyctl deploy -a conch-frontend
   ```

5. **Configure PostgreSQL and Redis**:
   ```bash
   flyctl postgres create
   flyctl redis create
   ```

### Option 3: Heroku

Traditional but reliable option:

1. **Install Heroku CLI**:
   ```bash
   npm install -g heroku
   ```

2. **Login and create apps**:
   ```bash
   heroku login
   heroku create conch-backend
   heroku create conch-frontend
   ```

3. **Add PostgreSQL addon**:
   ```bash
   heroku addons:create heroku-postgresql:standard-0 -a conch-backend
   ```

4. **Add Redis addon**:
   ```bash
   heroku addons:create heroku-redis:premium-0 -a conch-backend
   ```

5. **Set environment variables**:
   ```bash
   heroku config:set JWT_SECRET=your-secret -a conch-backend
   heroku config:set NEXT_PUBLIC_API_URL=https://conch-backend.herokuapp.com
   ```

6. **Deploy**:
   ```bash
   git push heroku main
   ```

### Option 4: Docker on VPS (DigitalOcean, Linode, etc.)

For complete control:

1. **SSH into server**:
   ```bash
   ssh root@your-server-ip
   ```

2. **Install Docker and Docker Compose**:
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sh get-docker.sh
   curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   chmod +x /usr/local/bin/docker-compose
   ```

3. **Clone repository**:
   ```bash
   git clone <your-repo> /opt/conch
   cd /opt/conch
   ```

4. **Configure environment**:
   ```bash
   cp backend/.env.example backend/.env
   # Edit backend/.env with production values
   vim backend/.env
   ```

5. **Deploy with Docker Compose**:
   ```bash
   docker-compose -f docker-compose.prod.yml up -d
   ```

6. **Setup nginx reverse proxy**:
   ```bash
   apt-get install nginx
   # Configure nginx with SSL/TLS
   ```

## CI/CD with GitHub Actions

The repository includes a GitHub Actions workflow for automated testing and deployment.

### Workflow Features
- Automated Rust backend testing
- Docker image building and pushing
- Container registry management
- Deployment hooks

### Setup

1. **Enable GitHub Actions** in repository settings
2. **Configure secrets**:
   - `DATABASE_URL`
   - `REDIS_URL`
   - `JWT_SECRET`
   - Deployment platform tokens (Railway, Fly.io, etc.)

3. **Push to main branch** to trigger workflow:
   ```bash
   git add .
   git commit -m "Deploy: Production release"
   git push origin main
   ```

## Monitoring and Maintenance

### Health Checks

Backend health endpoint:
```bash
curl http://your-backend:3001/health
```

Expected response:
```json
{
  "status": "ok",
  "service": "conch-backend"
}
```

### Logs

View logs from Docker:
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f backend
docker-compose logs -f postgres
docker-compose logs -f redis
```

### Database Backups

PostgreSQL backup:
```bash
docker-compose exec postgres pg_dump -U postgres conch_db > backup.sql
```

Restore:
```bash
docker-compose exec -T postgres psql -U postgres conch_db < backup.sql
```

### Database Migrations

Migrations run automatically on server startup. To run manually:
```bash
docker-compose exec backend cargo sqlx migrate run
```

## Performance Optimization

### Database Optimization
- Connection pooling configured to 5 connections
- Indexed queries on common fields
- Regular `ANALYZE` for query planner

### Redis Caching
- Event streaming via Redis pub/sub
- Real-time subscriptions
- Efficient memory usage

### Frontend Optimization
- Next.js built-in optimizations
- Code splitting and lazy loading
- Static generation where possible

## Security Considerations

### Production Checklist
- [ ] Change default JWT_SECRET to strong random value
- [ ] Use HTTPS/TLS for all connections
- [ ] Enable PostgreSQL SSL connections
- [ ] Use strong database passwords
- [ ] Enable Redis authentication
- [ ] Configure CORS appropriately
- [ ] Setup rate limiting
- [ ] Enable security headers
- [ ] Regular security updates
- [ ] Monitor and log all access

### HTTPS Setup

Use Let's Encrypt with nginx:
```bash
apt-get install certbot python3-certbot-nginx
certbot certonly --nginx -d yourdomain.com
```

## Troubleshooting

### Backend Connection Issues

```bash
# Check backend logs
docker-compose logs backend

# Verify database connectivity
docker-compose exec backend psql $DATABASE_URL

# Check Redis connection
docker-compose exec backend redis-cli -u $REDIS_URL ping
```

### Frontend API Issues

```bash
# Verify API_URL environment variable
echo $NEXT_PUBLIC_API_URL

# Test backend endpoint
curl http://backend:3001/health

# Check browser console for CORS errors
```

### Database Migration Issues

```bash
# Check migration status
docker-compose exec backend cargo sqlx migrate info

# Reset database (development only!)
docker-compose down
docker volume rm conch_postgres_data
docker-compose up -d
```

## Scaling Considerations

### Horizontal Scaling
- Load balancer (nginx, HAProxy)
- Multiple backend instances
- Connection pooling
- Redis sentinel for failover

### Vertical Scaling
- Increase database connections
- Allocate more memory to services
- Optimize queries with indexes

## Support and Resources

- README: See README_CONCH.md for API and feature documentation
- Issues: Report bugs and feature requests on GitHub
- Discussions: Community discussions on GitHub

## License

MIT
