---
name: Fase 8 - Production Deployment
overview: "Descomposición simplificada de la Fase 8 del MVP enfocada exclusivamente en el despliegue a producción. Incluye containerización con Docker, CI/CD básico, SSL/TLS, gestión de environments y backups de base de datos."
phase: 8
parent_phase: "009-plan-implementacion-detallado.md"
total_issues: 6
---

# Fase 8: Production Deployment

Descomposición en issues enfocados exclusivamente en desplegar el MVP a un environment productivo funcional y seguro.

> **Nota:** Los issues de monitoring, observabilidad, alerting, performance optimization y dashboards se movieron al documento [017-improvements-monitoring-observability.md](./017-improvements-monitoring-observability.md) para implementación post-deployment.

---

## Issue 8.1: Dockerize Backend Application

**Prioridad:** Alta  
**Dependencias:** Ninguna  
**Estimación:** 6 horas

### Descripción

Crear Dockerfiles optimizados para el backend con multi-stage builds, configuración de environment variables y mejores prácticas de seguridad.

### Acceptance Criteria

- [ ] Dockerfile multi-stage para backend
- [ ] Imagen optimizada (< 500MB)
- [ ] No ejecuta como root
- [ ] Health check configurado
- [ ] Environment variables desde .env
- [ ] .dockerignore configurado
- [ ] Documentación de build y run
- [ ] Tests de imagen Docker

### Files to Create

```
context-ai-api/Dockerfile                     # Dockerfile principal
context-ai-api/.dockerignore                  # Ignore patterns
context-ai-api/docker-compose.prod.yml        # Compose para prod
context-ai-api/scripts/docker-build.sh        # Script de build
```

### Technical Notes

```dockerfile
# Dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

# Install pnpm
RUN npm install -g pnpm

# Copy package files
COPY package.json pnpm-lock.yaml ./

# Install dependencies
RUN pnpm install --frozen-lockfile

# Copy source code
COPY . .

# Build application
RUN pnpm build

# Stage 2: Production
FROM node:20-alpine AS production

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

# Install pnpm
RUN npm install -g pnpm

# Copy package files
COPY package.json pnpm-lock.yaml ./

# Install production dependencies only
RUN pnpm install --prod --frozen-lockfile

# Copy built application from builder
COPY --from=builder /app/dist ./dist

# Change ownership
RUN chown -R nestjs:nodejs /app

# Switch to non-root user
USER nestjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "dist/main.js"]
```

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - GCP_PROJECT_ID=${GCP_PROJECT_ID}  # Vertex AI usa ADC del service account
      - AUTH0_DOMAIN=${AUTH0_DOMAIN}
      - AUTH0_AUDIENCE=${AUTH0_AUDIENCE}
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:3000/health']
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${DB_USER}']
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local
```

---

## Issue 8.2: Dockerize Frontend Application

**Prioridad:** Alta  
**Dependencias:** Ninguna  
**Estimación:** 5 horas

### Descripción

Crear Dockerfile para Next.js con standalone output optimizado para producción y configuración de nginx reverse proxy.

### Acceptance Criteria

- [ ] Dockerfile multi-stage para Next.js
- [ ] Standalone output configurado
- [ ] Imagen optimizada (< 300MB)
- [ ] Environment variables en runtime
- [ ] nginx configurado como reverse proxy
- [ ] Compresión gzip habilitada
- [ ] Documentación de deployment
- [ ] Tests de imagen Docker

### Files to Create

```
context-ai-front/Dockerfile                   # Dockerfile principal
context-ai-front/.dockerignore                # Ignore patterns
context-ai-front/nginx.conf                   # Configuración nginx
context-ai-front/next.config.js               # Config (actualizar)
```

### Technical Notes

```dockerfile
# Dockerfile
FROM node:20-alpine AS base

# Install pnpm
RUN npm install -g pnpm

# Stage 1: Dependencies
FROM base AS deps

WORKDIR /app

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# Stage 2: Builder
FROM base AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Set environment variables for build
ENV NEXT_TELEMETRY_DISABLED 1

# Build application
RUN pnpm build

# Stage 3: Runner
FROM base AS runner

WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

# Create non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy necessary files
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

```javascript
// next.config.js
module.exports = {
  output: 'standalone',
  // ... other config
};
```

```nginx
# nginx.conf
upstream nextjs {
    server frontend:3000;
}

server {
    listen 80;
    server_name contextai.com www.contextai.com;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    
    location / {
        proxy_pass http://nextjs;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## Issue 8.3: Configure CI/CD Pipeline for Deployment (Simplificado)

**Prioridad:** Alta  
**Dependencias:** 8.1, 8.2  
**Estimación:** 4 horas

### Descripción

Configurar CI/CD pipeline básico con GitHub Actions para build, test y deployment automático a producción. Versión simplificada sin security scanning, staging environment ni notificaciones.

### Acceptance Criteria

- [ ] Pipeline de CI/CD con GitHub Actions
- [ ] Build automático en push a main
- [ ] Tests automáticos (lint, unit, build)
- [ ] Build de imágenes Docker
- [ ] Push a GitHub Container Registry
- [ ] Deployment a producción (manual trigger)

### Files to Create

```
.github/workflows/deploy-production.yml       # Deploy a prod
scripts/deploy.sh                             # Script de deployment
```

### Technical Notes

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  workflow_dispatch:  # Manual trigger only
    inputs:
      version:
        description: 'Version to deploy'
        required: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 9
      
      - name: Run Backend Tests
        run: |
          cd context-ai-api
          pnpm install
          pnpm lint
          pnpm test
          pnpm build
      
      - name: Run Frontend Tests
        run: |
          cd context-ai-front
          pnpm install
          pnpm lint
          pnpm build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment:
      name: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and Push Docker Images
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          
          docker build -t contextai/api:${{ github.event.inputs.version }} ./context-ai-api
          docker push contextai/api:${{ github.event.inputs.version }}
          
          docker build -t contextai/frontend:${{ github.event.inputs.version }} ./context-ai-front
          docker push contextai/frontend:${{ github.event.inputs.version }}
      
      - name: Deploy to Production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /opt/contextai
            export VERSION=${{ github.event.inputs.version }}
            docker-compose pull
            docker-compose up -d
            docker-compose ps
```

---

## Issue 8.4: Configure SSL/TLS and HTTPS

**Prioridad:** Alta  
**Dependencias:** 8.3  
**Estimación:** 4 horas

> _Renumerado desde 8.9 original_

### Descripción

Configurar SSL/TLS certificates con Let's Encrypt, HTTPS redirect y security headers para producción.

### Acceptance Criteria

- [ ] SSL certificates configurados con Let's Encrypt
- [ ] Auto-renewal configurado con certbot
- [ ] HTTP to HTTPS redirect
- [ ] Security headers configurados (HSTS, CSP, etc.)
- [ ] Nginx configurado como reverse proxy
- [ ] Documentación de renovación de certs

### Files to Create

```
config/nginx/nginx-ssl.conf                   # Nginx con SSL
scripts/setup-ssl.sh                          # Setup SSL
scripts/renew-ssl.sh                          # Renovación SSL
```

### Technical Notes

```nginx
# nginx-ssl.conf
server {
    listen 80;
    server_name contextai.com www.contextai.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name contextai.com www.contextai.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/contextai.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/contextai.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" always;

    # Gzip Compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    location / {
        proxy_pass http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    location /api {
        proxy_pass http://api:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# scripts/setup-ssl.sh
#!/bin/bash

# Install certbot
apt-get update
apt-get install -y certbot python3-certbot-nginx

# Obtain certificate
certbot --nginx -d contextai.com -d www.contextai.com \
  --email admin@contextai.com \
  --agree-tos \
  --non-interactive

# Setup auto-renewal
echo "0 0 * * * root certbot renew --quiet" >> /etc/crontab

# Test renewal
certbot renew --dry-run

echo "✅ SSL setup complete!"
```

---

## Issue 8.5: Configure Environment Management

**Prioridad:** Alta  
**Dependencias:** Ninguna  
**Estimación:** 5 horas

> _Renumerado desde 8.11 original_

### Descripción

Configurar gestión de environments (dev, staging, production) con variables de entorno seguras y secrets management.

### Acceptance Criteria

- [ ] Configuración separada por environment
- [ ] Secrets management con GitHub Secrets
- [ ] .env files no committeados en git
- [ ] .env.example templates actualizados
- [ ] Variables de entorno documentadas
- [ ] Validación de variables requeridas al inicio

### Files to Create

```
.env.example                                  # Template (actualizar)
src/config/env-validation.ts                  # Validación
```

### Technical Notes

```typescript
// env-validation.ts
import { plainToClass } from 'class-transformer';
import { IsString, IsNumber, IsUrl, validateSync } from 'class-validator';

class EnvironmentVariables {
  @IsString()
  NODE_ENV: string;

  @IsNumber()
  PORT: number;

  @IsUrl({ require_tld: false })
  DATABASE_URL: string;

  @IsString()
  GCP_PROJECT_ID: string;  // Vertex AI usa ADC — no requiere API key

  @IsString()
  AUTH0_DOMAIN: string;

  @IsString()
  AUTH0_AUDIENCE: string;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToClass(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );

  const errors = validateSync(validatedConfig, {
    skipMissingProperties: false,
  });

  if (errors.length > 0) {
    throw new Error(`Environment validation failed:\n${errors.toString()}`);
  }

  return validatedConfig;
}
```

```bash
# .env.example
# Application
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/contextai
DB_HOST=localhost
DB_PORT=5432
DB_NAME=contextai
DB_USER=user
DB_PASSWORD=password

# Google Cloud / Vertex AI (usa ADC — no requiere API key)
# Local: gcloud auth application-default login
GCP_PROJECT_ID=your-gcp-project-id

# Auth0
AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_AUDIENCE=https://api.contextai.com
AUTH0_ISSUER_BASE_URL=https://your-tenant.auth0.com/

# Logging
LOG_LEVEL=info
```

---

## Issue 8.6: Configure Database Backups (Simplificado)

**Prioridad:** Alta  
**Dependencias:** 8.1  
**Estimación:** 2 horas

> _Renumerado y simplificado desde 8.8 original_

### Descripción

Configurar backup básico de PostgreSQL con script manual y cron job diario. Versión simplificada sin encriptación, S3 ni tests automatizados de restauración.

### Acceptance Criteria

- [ ] Script de backup con pg_dump + gzip
- [ ] Cron job para backup diario automático
- [ ] Script de restauración documentado
- [ ] Retención básica (últimos 7 días local)

### Files to Create

```
scripts/backup-database.sh                    # Script de backup
scripts/restore-database.sh                   # Script de restauración
```

### Technical Notes

```bash
#!/bin/bash
# scripts/backup-database.sh

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/postgresql"
BACKUP_FILE="$BACKUP_DIR/backup_$TIMESTAMP.sql.gz"

# Create backup directory
mkdir -p $BACKUP_DIR

# Perform backup
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME | gzip > $BACKUP_FILE

# Cleanup old backups (keep last 7 days)
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete

# Verify backup
if [ -f "$BACKUP_FILE" ]; then
  echo "✅ Backup successful: $BACKUP_FILE ($(du -h $BACKUP_FILE | cut -f1))"
else
  echo "❌ Backup failed!"
  exit 1
fi
```

```bash
#!/bin/bash
# scripts/restore-database.sh

set -e

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: ./restore-database.sh <backup_file.sql.gz>"
  exit 1
fi

echo "⚠️  Restoring database from: $BACKUP_FILE"
echo "This will overwrite the current database. Press Ctrl+C to cancel..."
sleep 5

gunzip -c $BACKUP_FILE | psql -h $DB_HOST -U $DB_USER -d $DB_NAME

echo "✅ Database restored successfully!"
```

```bash
# Cron job (agregar al servidor)
# Backup diario a las 2 AM
0 2 * * * /opt/contextai/scripts/backup-database.sh >> /var/log/db-backup.log 2>&1
```

---

## Resumen y Orden de Implementación

### Paso 1: Preparación (Paralelo)
1. **Issue 8.5:** Configure Environment Management
2. **Issue 8.1:** Dockerize Backend Application
3. **Issue 8.2:** Dockerize Frontend Application

### Paso 2: Deployment (Secuencial)
4. **Issue 8.4:** Configure SSL/TLS and HTTPS
5. **Issue 8.3:** Configure CI/CD Pipeline (Simplificado)

### Paso 3: Reliability
6. **Issue 8.6:** Configure Database Backups (Simplificado)

---

## Estimación Total

**Total de horas estimadas:** ~26 horas  
**Total de sprints (2 semanas c/u):** ~1 sprint  
**Desarrolladores recomendados:** 1 Full-Stack

---

## Stack Tecnológico (Deployment)

### Infrastructure
- **Containerization:** Docker, Docker Compose
- **CI/CD:** GitHub Actions
- **Cloud Provider:** AWS/GCP/DigitalOcean (flexible)
- **Web Server:** Nginx

### Security
- **SSL/TLS:** Let's Encrypt
- **Secrets Management:** GitHub Secrets
- **Security Headers:** Configured in Nginx

### Database
- **Primary DB (relacional):** PostgreSQL (sin extensión vector; vectores en Pinecone)
- **Vector Store:** Pinecone (externo)
- **Backups:** Local con pg_dump + cron

---

## Dependencias Externas

### Backend Packages

```bash
# Health checks (si no está instalado)
pnpm add @nestjs/terminus
```

### Infrastructure Tools

```bash
# Docker
docker --version
docker-compose --version

# SSL
certbot --version
```

---

## Post-Deployment Checklist

```bash
# 1. Verify deployment
curl https://api.contextai.com/health

# 2. Check logs
docker-compose logs -f api

# 3. Test critical paths
# - User login
# - Document upload
# - Chat query

# 4. Verify SSL
# - Check https:// works
# - Check http:// redirects to https://

# 5. Verify backups
./scripts/backup-database.sh
```

---

## Validación de Completitud

La Fase 8 se considera completa cuando:

- [ ] Todos los 6 issues están completados
- [ ] Aplicaciones dockerizadas y funcionando
- [ ] CI/CD pipeline operativo
- [ ] SSL/TLS configurado en producción
- [ ] Environment variables gestionados de forma segura
- [ ] Backups de base de datos funcionando
- [ ] Critical paths verificados manualmente

---

## Next Steps Post-Fase 8

Una vez completada la Fase 8:

1. **Production Launch:** Sistema en producción y operativo
2. **Monitoring & Observability:** Implementar issues del documento [017-improvements-monitoring-observability.md](./017-improvements-monitoring-observability.md)
3. **User Onboarding:** Comenzar onboarding de usuarios reales
4. **Iteration:** Recopilar feedback y planear mejoras

---

## Documentation Links

- [Docker Documentation](https://docs.docker.com/)
- [GitHub Actions](https://docs.github.com/en/actions)
- [Nginx](https://nginx.org/en/docs/)
- [Let's Encrypt](https://letsencrypt.org/docs/)
