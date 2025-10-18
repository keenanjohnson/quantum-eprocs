# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About Quantum

Quantum is an Electronic Procedures Software for space mission operations. It allows mission operators to upload, view, and run procedure instances consisting of sections and steps. Built for Audacy space missions using the MEAN stack (MongoDB, Express, AngularJS, Node.js).

## Development Commands

### Quick Start with Docker Compose (Recommended)
The easiest way to get started is using Docker Compose, which runs both MongoDB and the application:

```bash
# IMPORTANT: Install dependencies locally first (required for volume mount)
npm install

# Start everything (MongoDB + Quantum app)
docker compose up -d

# View logs
docker compose logs -f

# Stop everything
docker compose down

# Stop and remove all data (including database)
docker compose down -v
```

Access the application at:
- `http://localhost:3000/dashboard` - **Main application (recommended)**
- `https://localhost` (HTTPS via nginx - self-signed cert warning expected)
- `http://localhost:80` (HTTP via nginx)

**Note**: You must run `npm install` locally before starting Docker Compose because the volume mount shares your local directory with the container. Without local `node_modules`, the app will fail to start.

### Running Locally (without Docker)
```bash
# Prerequisites: MongoDB must be running locally on port 27017

# Install dependencies
npm install

# Start server (runs on port 3000 by default, or PORT env variable)
npm start

# Stop server
npm stop
```

### Manual Docker Setup (Alternative)
```bash
# Build and run Docker container for development
docker build -t quantum-app .
docker run -d -t --name quantum --cap-add SYS_PTRACE \
  -v /proc:/host/proc:ro -v /sys:/host/sys:ro \
  -v $(pwd):/node/ -p 80:80 -p 443:443 quantum-app
```

### Testing
```bash
# Run all tests (Karma for frontend + Mocha for backend)
npm test

# Run only Karma tests for frontend AngularJS code
karma start karma.conf.js

# Run only Mocha tests for backend API routes
mocha
```

## Architecture Overview

### Backend (Node.js/Express)
- **Entry Point**: [server.js](server.js) - Initializes Express, connects to MongoDB, configures Passport authentication
- **Routes**: [server/routes.js](server/routes.js) - Defines all API endpoints and authentication routes
- **Controllers**: [server/controllers/](server/controllers/)
  - `procedure.controller.js` - Handles procedure upload, retrieval, instances, and operations
  - `user.controller.js` - Manages user roles, missions, and permissions
- **Models**: [server/models/](server/models/)
  - `procedure.js` - MongoDB schema for procedures (procedureID, sections, versions, instances)
  - `user.js` - MongoDB schema for users with Google OAuth integration
- **Authentication**: [server/lib/passport.js](server/lib/passport.js) - Google OAuth configuration (currently commented out in routes)

### Frontend (AngularJS 1.6)
- **Main Module**: [app/js/app.js](app/js/app.js) - Defines routes and initializes 'quantum' Angular module
- **Components**: [app/js/components/](app/js/components/)
  - `procedures/` - Procedure list view and upload functionality
  - `sections/` - Individual procedure sections and steps display
  - `runIndex/` - Running procedure instances management
  - `archivedIndex/` - Archived procedure instances
  - `homepage/` - Landing page components
  - `rightSidebar/` - User role and mission selection
- **Services**: [app/js/services/](app/js/services/) - Shared business logic and API calls
- **Views**: [app/views/](app/views/) - EJS templates for server-rendered pages

### Configuration
- **Environment Config**: [config/config.env.js](config/config.env.js) - Reads from `config.env` or uses defaults
  - Google OAuth credentials (clientID, clientSecret, callbackURL)
  - MongoDB connection string (default: `mongodb://localhost:27017/quindar`)
  - Supports NODE_ENV values: 'development', 'staging', 'production'
- **Roles**: [config/role.js](config/role.js) - Defines available user roles for missions
- **SSL Certificates**: [config/ssl/](config/ssl/) - HTTPS certificates for production

## Key Workflows

### Procedure Upload Flow
1. User uploads Excel file via `/upload` endpoint (using multer middleware)
2. File saved to `/tmp/uploads` directory
3. `procedure.controller.js` parses Excel and creates procedure document
4. Procedure stored in MongoDB with sections array and metadata

### Procedure Instance Execution
1. User creates instance from base procedure via `/saveProcedureInstance`
2. Instance runs with version/revision tracking
3. Steps marked complete, comments added via `/setComments`
4. User status updates tracked via `/setUserStatus`
5. Instance completed via `/setInstanceCompleted` and archived

### User Role Management
1. Users authenticate via Google OAuth (currently disabled - see [server/routes.js:144](server/routes.js#L144))
2. Missions assigned via `/setMissionForUser`
3. Allowed roles configured via `/setAllowedRoles`
4. Active role set via `/setUserRole`
5. Role permissions defined in [config/role.js](config/role.js)

## Database Schema Notes

### Procedure Document
- `procedureID` - Unique identifier
- `sections` - Array of section objects with steps
- `instances` - Array of running/archived instances
- `versions` - Version control for procedure updates

### User Document
- `google` - OAuth token and profile info
- `grid` - User role assignments (array)
- `missions` - Associated missions (array)

## Important Notes

- **Authentication Currently Disabled**: The `isLoggedIn` middleware in [server/routes.js:141-149](server/routes.js#L141-L149) always returns `next()` without checking authentication
- **Database Name**: Default database is "quindar" not "quantum"
- **File Uploads**: Procedures uploaded as Excel files following template in [app/js/components/procedures/](app/js/components/procedures/)
- **Testing Setup**: Karma uses Chrome without sandbox flag for Docker compatibility
- **Port Configuration**: Server defaults to port 3000, but Docker maps to 80/443

## Troubleshooting

### Docker Compose Issues

**Problem**: Container starts but app crashes with "Cannot find module 'express'"
- **Cause**: Volume mount overwrites the container's `/node` directory, removing `node_modules` installed during build
- **Solution**: Run `npm install` on your host machine before starting Docker Compose

**Problem**: "centos:latest: not found" error during build
- **Cause**: CentOS has reached end-of-life and is no longer available on Docker Hub
- **Solution**: Dockerfile has been updated to use `rockylinux:8` instead

**Problem**: Container runs but services don't start (no output in logs)
- **Cause**: Multiple `CMD` statements in Dockerfile - only the last one executes
- **Solution**: Dockerfile has been fixed to use single `CMD` with `pm2-runtime`

**Problem**: OAuth errors in logs
- **Cause**: Google OAuth credentials not configured in [config/config.env.js](config/config.env.js)
- **Impact**: None - authentication is currently disabled, these errors are expected and can be ignored

### Database Connection Issues

**Problem**: "Error connecting Mongo" in logs
- **Solution**: Ensure MongoDB container is healthy before app starts (docker-compose handles this with health checks)
- **Manual Check**: `docker logs quantum-mongodb` should show "waiting for connections on port 27017"
