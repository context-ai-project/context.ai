---
name: Improvements - Monitoring y Observabilidad
overview: "Issues de monitoring, observabilidad, alerting, performance optimization y dashboards extraídos de la Fase 8. Se implementarán post-deployment como mejoras continuas del sistema en producción."
priority: Post-MVP
parent_phase: "013-fase-8-deployment-monitoring-issues.md"
total_issues: 8
---

# Improvements: Monitoring y Observabilidad

Issues extraídos de la [Fase 8 - Production Deployment](./013-fase-8-deployment-monitoring-issues.md) para implementación post-deployment. Estas mejoras se priorizan una vez que el sistema esté funcionando en producción.

---

## IMP-1: Implement Structured Logging

**Prioridad:** Alta  
**Dependencias:** Fase 8 completada  
**Estimación:** 6 horas  
**Origen:** Issue 8.4 original

### Descripción

Implementar logging estructurado con Winston o Pino para facilitar debugging, monitoring y análisis de logs en producción.

### Acceptance Criteria

- [ ] Logger configurado (Winston o Pino)
- [ ] Logs estructurados en formato JSON
- [ ] Niveles de log configurables por environment
- [ ] Context y correlation IDs en logs
- [ ] Logs de requests HTTP (middleware)
- [ ] Logs de errores con stack traces
- [ ] Rotación de logs configurada
- [ ] Integration con servicio de logs (opcional)

### Files to Create

```
src/common/logger/logger.module.ts            # Módulo de logger
src/common/logger/logger.service.ts           # Servicio
src/common/middleware/request-logger.middleware.ts  # Middleware
src/common/interceptors/logging.interceptor.ts  # Interceptor
config/logger.config.ts                       # Configuración
```

### Technical Notes

```typescript
// logger.service.ts
import { Injectable, LoggerService } from '@nestjs/common';
import * as winston from 'winston';

@Injectable()
export class CustomLoggerService implements LoggerService {
  private logger: winston.Logger;

  constructor() {
    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json(),
      ),
      defaultMeta: {
        service: 'context-ai-api',
        environment: process.env.NODE_ENV,
      },
      transports: [
        new winston.transports.Console({
          format: winston.format.combine(
            winston.format.colorize(),
            winston.format.simple(),
          ),
        }),
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error',
          maxsize: 5242880, // 5MB
          maxFiles: 5,
        }),
        new winston.transports.File({
          filename: 'logs/combined.log',
          maxsize: 5242880,
          maxFiles: 5,
        }),
      ],
    });
  }

  log(message: string, context?: string, meta?: Record<string, unknown>) {
    this.logger.info(message, { context, ...meta });
  }

  error(message: string, trace?: string, context?: string, meta?: Record<string, unknown>) {
    this.logger.error(message, { trace, context, ...meta });
  }

  warn(message: string, context?: string, meta?: Record<string, unknown>) {
    this.logger.warn(message, { context, ...meta });
  }

  debug(message: string, context?: string, meta?: Record<string, unknown>) {
    this.logger.debug(message, { context, ...meta });
  }
}

// request-logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { CustomLoggerService } from '../logger/logger.service';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestLoggerMiddleware implements NestMiddleware {
  constructor(private logger: CustomLoggerService) {}

  use(req: Request, res: Response, next: NextFunction) {
    const correlationId = uuidv4();
    req['correlationId'] = correlationId;

    const startTime = Date.now();

    res.on('finish', () => {
      const duration = Date.now() - startTime;
      
      this.logger.log('HTTP Request', 'HTTP', {
        correlationId,
        method: req.method,
        url: req.url,
        statusCode: res.statusCode,
        duration,
        userAgent: req.headers['user-agent'],
        ip: req.ip,
      });
    });

    next();
  }
}

// Dependencias
pnpm add winston
pnpm add -D @types/winston
```

---

## IMP-2: Implement APM (Application Performance Monitoring)

**Prioridad:** Media  
**Dependencias:** IMP-1  
**Estimación:** 8 horas  
**Origen:** Issue 8.5 original

### Descripción

Integrar APM tool (New Relic, Datadog o self-hosted) para monitorear performance de la aplicación, detectar cuellos de botella y errores en producción.

### Acceptance Criteria

- [ ] APM agent configurado (New Relic o Datadog)
- [ ] Métricas de performance capturadas
- [ ] Distributed tracing habilitado
- [ ] Error tracking configurado
- [ ] Custom metrics definidas
- [ ] Dashboards configurados
- [ ] Alertas configuradas para métricas críticas
- [ ] Documentación de monitoreo

### Files to Create

```
src/common/apm/apm.module.ts                  # Módulo APM
src/common/apm/apm.service.ts                 # Servicio
src/common/interceptors/performance.interceptor.ts  # Interceptor
config/apm.config.ts                          # Configuración
```

### Technical Notes

```typescript
// apm.service.ts
import { Injectable } from '@nestjs/common';
import * as newrelic from 'newrelic'; // or datadog

@Injectable()
export class ApmService {
  trackCustomMetric(name: string, value: number) {
    newrelic.recordMetric(name, value);
  }

  trackBusinessEvent(eventName: string, attributes: Record<string, unknown>) {
    newrelic.recordCustomEvent(eventName, attributes);
  }

  startTransaction(name: string, type: string = 'web') {
    return newrelic.startWebTransaction(name, () => {
      // Transaction logic
    });
  }

  noticeError(error: Error, customAttributes?: Record<string, unknown>) {
    newrelic.noticeError(error, customAttributes);
  }

  addCustomAttribute(key: string, value: string | number) {
    newrelic.addCustomAttribute(key, value);
  }
}

// performance.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { ApmService } from '../apm/apm.service';

@Injectable()
export class PerformanceInterceptor implements NestInterceptor {
  constructor(private apmService: ApmService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest();
    const endpoint = `${request.method} ${request.route?.path}`;
    
    const startTime = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - startTime;
        
        this.apmService.trackCustomMetric(`endpoint.${endpoint}.duration`, duration);
        
        if (endpoint.includes('/chat/query')) {
          this.apmService.trackBusinessEvent('ChatQuery', {
            duration,
            userId: request.user?.id,
            sectorId: request.body.sectorId,
          });
        }
      }),
    );
  }
}

// Dependencias
// New Relic
pnpm add newrelic

// O Datadog
pnpm add dd-trace
```

---

## IMP-3: Implement Metrics with Prometheus

**Prioridad:** Media  
**Dependencias:** IMP-1  
**Estimación:** 6 horas  
**Origen:** Issue 8.6 original

### Descripción

Configurar Prometheus para recolectar métricas técnicas y de negocio, con exportación de métricas custom y dashboards en Grafana.

### Acceptance Criteria

- [ ] Prometheus client configurado
- [ ] Métricas técnicas (CPU, memoria, requests)
- [ ] Métricas de negocio (queries, users, documents)
- [ ] Endpoint /metrics expuesto
- [ ] Grafana configurado con dashboards
- [ ] Métricas custom por módulo
- [ ] Alerting rules definidas
- [ ] Documentación de métricas

### Files to Create

```
src/common/metrics/metrics.module.ts          # Módulo de métricas
src/common/metrics/metrics.service.ts         # Servicio
config/prometheus.yml                         # Config Prometheus
config/grafana-dashboards/api-dashboard.json  # Dashboard
```

### Technical Notes

```typescript
// metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Counter, Histogram, Gauge, register } from 'prom-client';

@Injectable()
export class MetricsService {
  private httpRequestDuration: Histogram;
  private httpRequestTotal: Counter;
  private activeChatSessions: Gauge;
  private documentsProcessed: Counter;

  constructor() {
    this.httpRequestDuration = new Histogram({
      name: 'http_request_duration_seconds',
      help: 'Duration of HTTP requests in seconds',
      labelNames: ['method', 'route', 'status'],
    });

    this.httpRequestTotal = new Counter({
      name: 'http_requests_total',
      help: 'Total number of HTTP requests',
      labelNames: ['method', 'route', 'status'],
    });

    this.activeChatSessions = new Gauge({
      name: 'active_chat_sessions',
      help: 'Number of active chat sessions',
    });

    this.documentsProcessed = new Counter({
      name: 'documents_processed_total',
      help: 'Total number of documents processed',
      labelNames: ['sector', 'status'],
    });
  }

  recordHttpRequest(method: string, route: string, status: number, duration: number) {
    this.httpRequestDuration.labels(method, route, status.toString()).observe(duration);
    this.httpRequestTotal.labels(method, route, status.toString()).inc();
  }

  incrementActiveSessions() {
    this.activeChatSessions.inc();
  }

  decrementActiveSessions() {
    this.activeChatSessions.dec();
  }

  recordDocumentProcessed(sector: string, status: 'success' | 'failed') {
    this.documentsProcessed.labels(sector, status).inc();
  }

  getMetrics(): Promise<string> {
    return register.metrics();
  }
}

// Dependencias
pnpm add prom-client
```

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - '9090:9090'
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - '3001:3000'
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./config/grafana-dashboards:/etc/grafana/provisioning/dashboards
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

---

## IMP-4: Implement Error Tracking with Sentry

**Prioridad:** Alta  
**Dependencias:** Fase 8 completada  
**Estimación:** 4 horas  
**Origen:** Issue 8.7 original

### Descripción

Integrar Sentry para tracking automático de errores en backend y frontend con contexto completo, sourcemaps y alerting.

### Acceptance Criteria

- [ ] Sentry configurado en backend
- [ ] Sentry configurado en frontend
- [ ] Sourcemaps subidos automáticamente
- [ ] Contexto de usuario capturado
- [ ] Breadcrumbs habilitados
- [ ] Alertas configuradas para errores críticos
- [ ] Performance monitoring habilitado
- [ ] Documentación de error tracking

### Files to Create

```
context-ai-api/src/config/sentry.config.ts    # Config backend
context-ai-front/lib/sentry.config.ts         # Config frontend
.github/workflows/upload-sourcemaps.yml       # Workflow
```

### Technical Notes

```typescript
// Backend - src/config/sentry.config.ts
import * as Sentry from '@sentry/node';
import { ProfilingIntegration } from '@sentry/profiling-node';

export function initSentry() {
  Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV,
    integrations: [
      new ProfilingIntegration(),
    ],
    tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
    profilesSampleRate: 0.1,
    beforeSend(event) {
      if (event.request) {
        delete event.request.cookies;
        delete event.request.headers?.authorization;
      }
      return event;
    },
  });
}

// Frontend - lib/sentry.config.ts
import * as Sentry from '@sentry/nextjs';

export function initSentry() {
  Sentry.init({
    dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
    environment: process.env.NEXT_PUBLIC_ENV,
    tracesSampleRate: 0.1,
    replaysSessionSampleRate: 0.1,
    replaysOnErrorSampleRate: 1.0,
    integrations: [
      new Sentry.Replay({
        maskAllText: false,
        blockAllMedia: false,
      }),
    ],
  });
}

// Dependencias
pnpm add @sentry/node @sentry/profiling-node  # Backend
pnpm add @sentry/nextjs                       # Frontend
```

---

## IMP-5: Implement Alerting System

**Prioridad:** Media  
**Dependencias:** IMP-2, IMP-3  
**Estimación:** 6 horas  
**Origen:** Issue 8.10 original

### Descripción

Configurar sistema de alerting para eventos críticos con múltiples canales (Slack, email) y escalation policies.

### Acceptance Criteria

- [ ] Alertas configuradas para errores críticos
- [ ] Alertas de performance (latencia alta)
- [ ] Alertas de disponibilidad (downtime)
- [ ] Alertas de seguridad (rate limit exceeded)
- [ ] Integración con Slack
- [ ] Integración con email
- [ ] Documentación de alertas

### Files to Create

```
src/common/alerts/alerts.module.ts            # Módulo de alertas
src/common/alerts/alerts.service.ts           # Servicio
config/alert-rules.yml                        # Reglas de alertas
```

### Technical Notes

```typescript
// alerts.service.ts
import { Injectable } from '@nestjs/common';
import axios from 'axios';

export enum AlertSeverity {
  INFO = 'info',
  WARNING = 'warning',
  CRITICAL = 'critical',
}

@Injectable()
export class AlertsService {
  async sendAlert(
    title: string,
    message: string,
    severity: AlertSeverity = AlertSeverity.INFO,
  ) {
    await this.sendToSlack(title, message, severity);

    if (severity === AlertSeverity.CRITICAL) {
      await this.sendEmail(title, message);
    }

    console.log(`[ALERT] ${severity.toUpperCase()}: ${title} - ${message}`);
  }

  private async sendToSlack(title: string, message: string, severity: AlertSeverity) {
    const emoji = severity === AlertSeverity.CRITICAL ? '🚨' : 
                  severity === AlertSeverity.WARNING ? '⚠️' : 'ℹ️';

    await axios.post(process.env.SLACK_WEBHOOK_URL || '', {
      text: `${emoji} *${title}*\n${message}`,
      username: 'Context.AI Alerts',
    });
  }

  private async sendEmail(title: string, message: string) {
    console.log(`Sending email alert: ${title} - ${message}`);
  }
}
```

```yaml
# config/alert-rules.yml
alerts:
  - name: HighErrorRate
    condition: error_rate > 5%
    duration: 5m
    severity: critical
    message: "Error rate is above 5% for the last 5 minutes"

  - name: HighLatency
    condition: p95_latency > 3s
    duration: 10m
    severity: warning
    message: "95th percentile latency is above 3 seconds"

  - name: ServiceDown
    condition: uptime < 99%
    duration: 2m
    severity: critical
    message: "Service availability is below 99%"

  - name: DatabaseConnectionIssue
    condition: db_connection_errors > 10
    duration: 5m
    severity: critical
    message: "Multiple database connection errors detected"
```

---

## IMP-6: Implement Performance Optimization

**Prioridad:** Media  
**Dependencias:** IMP-3  
**Estimación:** 6 horas  
**Origen:** Issue 8.12 original

### Descripción

Optimizar performance de producción con caching (Redis), CDN para assets estáticos, database indexing y query optimization.

### Acceptance Criteria

- [ ] Redis configurado para caching
- [ ] CDN configurado para assets estáticos
- [ ] Database queries optimizadas
- [ ] Índices de BD apropiados
- [ ] Response caching implementado
- [ ] Compression habilitada
- [ ] Performance metrics mejoradas
- [ ] Documentación de optimizaciones

### Files to Create

```
src/common/cache/cache.module.ts              # Módulo de cache
src/common/cache/cache.service.ts             # Servicio
config/redis.config.ts                        # Configuración Redis
scripts/analyze-queries.sql                   # Análisis de queries
```

### Technical Notes

```typescript
// cache.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class CacheService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async get<T>(key: string): Promise<T | undefined> {
    return await this.cacheManager.get<T>(key);
  }

  async set<T>(key: string, value: T, ttl?: number): Promise<void> {
    await this.cacheManager.set(key, value, ttl);
  }

  async del(key: string): Promise<void> {
    await this.cacheManager.del(key);
  }

  async wrap<T>(
    key: string,
    fn: () => Promise<T>,
    ttl?: number,
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached) return cached;

    const result = await fn();
    await this.set(key, result, ttl);
    return result;
  }
}

// app.module.ts
import { CacheModule } from '@nestjs/cache-manager';
import * as redisStore from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.register({
      isGlobal: true,
      store: redisStore,
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT,
      ttl: 600, // 10 minutes default
    }),
  ],
})
export class AppModule {}

// Dependencias
pnpm add @nestjs/cache-manager cache-manager cache-manager-redis-store redis
```

---

## IMP-7: Create Operations Runbook

**Prioridad:** Media  
**Dependencias:** Todas las anteriores  
**Estimación:** 8 horas  
**Origen:** Issue 8.13 original

### Descripción

Crear runbook completo para operaciones incluyendo deployment, troubleshooting, disaster recovery y common issues.

### Acceptance Criteria

- [ ] Deployment procedures documentadas
- [ ] Rollback procedures documentadas
- [ ] Troubleshooting guide completo
- [ ] Common issues y soluciones
- [ ] Disaster recovery plan
- [ ] On-call procedures
- [ ] Architecture diagrams actualizados
- [ ] Contact information actualizada

### Files to Create

```
docs/ops/RUNBOOK.md                           # Runbook principal
docs/ops/DEPLOYMENT.md                        # Deployment guide
docs/ops/TROUBLESHOOTING.md                   # Troubleshooting
docs/ops/DISASTER_RECOVERY.md                 # DR plan
docs/ops/COMMON_ISSUES.md                     # Issues comunes
```

---

## IMP-8: Setup Production Monitoring Dashboard

**Prioridad:** Media  
**Dependencias:** IMP-2, IMP-3  
**Estimación:** 6 horas  
**Origen:** Issue 8.14 original

### Descripción

Crear dashboard centralizado en Grafana con todas las métricas críticas, SLOs y health status visible en tiempo real.

### Acceptance Criteria

- [ ] Dashboard de Grafana con métricas clave
- [ ] Visualización de SLIs/SLOs
- [ ] Alertas visuales en dashboard
- [ ] Uptime tracking
- [ ] Request rate y latency
- [ ] Error rate tracking
- [ ] Business metrics dashboard

### Files to Create

```
config/grafana/main-dashboard.json            # Dashboard principal
config/grafana/business-metrics-dashboard.json # Métricas negocio
config/grafana/datasources.yml                # Datasources
```

### Technical Notes

```json
{
  "dashboard": {
    "title": "Context.AI Production Overview",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [{ "expr": "rate(http_requests_total[5m])" }]
      },
      {
        "title": "P95 Latency",
        "targets": [{ "expr": "histogram_quantile(0.95, http_request_duration_seconds)" }]
      },
      {
        "title": "Error Rate",
        "targets": [{ "expr": "rate(http_requests_total{status=~\"5..\"}[5m])" }]
      },
      {
        "title": "Active Chat Sessions",
        "targets": [{ "expr": "active_chat_sessions" }]
      },
      {
        "title": "Documents Processed (24h)",
        "targets": [{ "expr": "increase(documents_processed_total[24h])" }]
      }
    ]
  }
}
```

---

## Resumen y Orden de Implementación Sugerido

### Prioridad 1: Observabilidad Básica (Post-Deployment inmediato)
1. **IMP-1:** Structured Logging (6h)
2. **IMP-4:** Error Tracking con Sentry (4h)

### Prioridad 2: Monitoring Avanzado
3. **IMP-2:** APM - Application Performance Monitoring (8h)
4. **IMP-3:** Metrics con Prometheus + Grafana (6h)

### Prioridad 3: Alerting y Dashboards
5. **IMP-5:** Alerting System (6h)
6. **IMP-8:** Production Monitoring Dashboard (6h)

### Prioridad 4: Optimization y Documentación
7. **IMP-6:** Performance Optimization con Redis (6h)
8. **IMP-7:** Operations Runbook (8h)

---

## Estimación Total

**Total de horas estimadas:** 50 horas  
**Total de sprints (2 semanas c/u):** ~1.5-2 sprints  
**Desarrolladores recomendados:** 1 Full-Stack + 1 DevOps (opcional)

---

## Stack Tecnológico

### Monitoring & Observability
- **APM:** New Relic o Datadog
- **Metrics:** Prometheus + Grafana
- **Logging:** Winston (structured JSON logs)
- **Error Tracking:** Sentry
- **Health Checks:** @nestjs/terminus

### Caching
- **Cache:** Redis
- **Cache Manager:** @nestjs/cache-manager

---

## Dependencias Externas

```bash
# Logging
pnpm add winston
pnpm add -D @types/winston

# APM (elegir uno)
pnpm add newrelic
# O
pnpm add dd-trace

# Error Tracking
pnpm add @sentry/node @sentry/profiling-node  # Backend
pnpm add @sentry/nextjs                       # Frontend

# Metrics
pnpm add prom-client

# Caching
pnpm add @nestjs/cache-manager cache-manager cache-manager-redis-store redis
```

---

## Service Level Objectives (SLOs) - Referencia

### Availability
- **Target:** 99.9% uptime (43.2 minutes downtime/month)
- **Measurement:** Uptime monitoring con health checks

### Performance
- **API Response Time:** P95 < 500ms, P99 < 1s
- **Chat Query:** P95 < 3s, P99 < 5s
- **Document Processing:** < 30s per document

### Error Rate
- **Target:** < 0.1% error rate
- **Critical Errors:** < 10 per day

### Business Metrics
- **Active Users:** Tracked daily
- **Documents Processed:** Tracked hourly
- **Chat Queries:** Tracked real-time

