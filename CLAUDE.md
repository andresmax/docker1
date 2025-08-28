# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Rails 8.0.2 application configured for Docker deployment using Kamal. The application uses:
- PostgreSQL as the database with multiple database configurations for production (primary, cache, queue, cable)
- Tailwind CSS for styling
- Hotwire (Turbo + Stimulus) for interactive frontend behavior
- Solid Cache, Solid Queue, and Solid Cable for Rails infrastructure
- Kamal for containerized deployment

## Common Commands

### Development (Local)
```bash
# Start development servers
bin/dev                    # Runs both web server and Tailwind CSS watcher
bin/rails server           # Rails server only
bin/rails tailwindcss:watch  # Tailwind CSS watcher only

# Database operations
bin/rails db:create
bin/rails db:migrate
bin/rails db:seed
bin/rails db:reset

# Console access
bin/rails console
bin/rails dbconsole
```

### Development (Docker)
```bash
# Start all services (Rails + PostgreSQL)
docker-compose up

# Start in background
docker-compose up -d

# Stop services
docker-compose down

# View logs
docker-compose logs -f web    # Rails app logs
docker-compose logs -f db     # PostgreSQL logs

# Run commands in Rails container
docker-compose exec web bin/rails console
docker-compose exec web bin/rails db:migrate
docker-compose exec web bin/rails test

# Rebuild containers after Gemfile changes
docker-compose build web
docker-compose up --build

# Clean up (remove volumes and containers)
docker-compose down -v
```

### Testing
```bash
# Run all tests
bin/rails test

# Run specific test types
bin/rails test:controllers
bin/rails test:models
bin/rails test:system
bin/rails test:integration

# Run specific test file
bin/rails test test/models/example_test.rb

# Run tests in parallel (default configuration)
bin/rails test
```

### Code Quality
```bash
# Lint Ruby code
bin/rubocop

# Security scan
bin/brakeman

# Auto-fix linting issues
bin/rubocop -a
```

### Deployment (Kamal)
```bash
# Deploy application
bin/kamal deploy

# Useful Kamal aliases (defined in config/deploy.yml)
bin/kamal console    # Interactive Rails console
bin/kamal shell      # Shell access
bin/kamal logs       # Tail application logs
bin/kamal dbc        # Database console
```

## Architecture

### Rails Structure
- Standard Rails 8.0 MVC architecture
- Uses Importmap for JavaScript management (no Node.js/npm required)
- Tailwind CSS integrated via `tailwindcss-rails` gem
- Hotwire for SPA-like behavior without heavy JavaScript

### Database Configuration
- Development/test: Local PostgreSQL (`docker1_development`, `docker1_test`)
- Production: Multi-database setup with separate databases for:
  - Primary application data
  - Cache storage (Solid Cache)
  - Background jobs (Solid Queue) 
  - Action Cable connections (Solid Cable)

### Deployment
- Docker-based deployment using Kamal
- Configured for single web server with SSL via Let's Encrypt
- Background job processing runs within Puma process (`SOLID_QUEUE_IN_PUMA: true`)
- Persistent storage volume mounted for Active Storage files

### Code Style
- Uses `rubocop-rails-omakase` for Ruby styling
- Configuration in `.rubocop.yml` inherits Omakase defaults

## Development Notes

- Tests run in parallel by default using all available processors
- Use `bin/dev` to start both Rails server and Tailwind watcher simultaneously
- The application is configured to use Solid Queue for background jobs within the web process in development
- Kamal deployment configuration is ready but requires updating server IPs and registry credentials in `config/deploy.yml`