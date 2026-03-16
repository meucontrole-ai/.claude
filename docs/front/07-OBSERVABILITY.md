# Skill: Observability (Logging, Error Tracking, Analytics, Web Vitals)

## Quando Ativar
- Implementar error tracking (Sentry)
- Implementar analytics (Plausible, PostHog, GA4)
- Monitorar Web Vitals (LCP, FID, CLS, INP, TTFB)
- Implementar logging estruturado no frontend
- Configurar source maps para debugging em producao
- Implementar feature flags

## Quando NAO Ativar
- Logica de negocio sem necessidade de monitoramento
- Criar componentes de UI (use Components)
- Testes (use Testing)

---

## Error Tracking (Sentry)

### Setup

```typescript
// shared/lib/sentry.ts
import * as Sentry from '@sentry/react'

function initSentry() {
  Sentry.init({
    dsn: import.meta.env.VITE_SENTRY_DSN,
    environment: import.meta.env.VITE_APP_ENV,
    tracesSampleRate: import.meta.env.PROD ? 0.1 : 1.0,
    replaysSessionSampleRate: 0.1,
    replaysOnErrorSampleRate: 1.0,
    integrations: [
      Sentry.browserTracingIntegration(),
      Sentry.replayIntegration({
        maskAllText: true,
        blockAllMedia: true,
      }),
    ],
    beforeSend(event) {
      if (event.user) {
        delete event.user.ip_address
        delete event.user.email
      }
      return event
    },
  })
}

export { initSentry }
```

### Captura de Erros em Componentes

```tsx
// shared/components/feedback/ErrorBoundaryWithTracking.tsx
import * as Sentry from '@sentry/react'
import { Component, type ErrorInfo, type ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback: ReactNode
}

interface State {
  hasError: boolean
}

class ErrorBoundaryWithTracking extends Component<Props, State> {
  constructor(props: Props) {
    super(props)
    this.state = { hasError: false }
  }

  static getDerivedStateFromError(): State {
    return { hasError: true }
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    Sentry.captureException(error, {
      extra: {
        componentStack: errorInfo.componentStack,
      },
    })
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback
    }
    return this.props.children
  }
}

export { ErrorBoundaryWithTracking }
```

### Contexto de Usuario

```typescript
// features/auth/hooks/useAuth.ts (trecho)
import * as Sentry from '@sentry/react'

function setUserContext(user: User) {
  Sentry.setUser({
    id: user.id,
    username: user.name,
  })
}

function clearUserContext() {
  Sentry.setUser(null)
}
```

---

## Web Vitals

### Monitoramento

```typescript
// shared/lib/webVitals.ts
import { onCLS, onINP, onLCP, onFCP, onTTFB, type Metric } from 'web-vitals'

interface VitalsReporter {
  report: (metric: Metric) => void
}

function createVitalsReporter(endpoint: string): VitalsReporter {
  const queue: Metric[] = []
  let timeoutId: ReturnType<typeof setTimeout> | null = null

  function flush() {
    if (queue.length === 0) return
    const body = JSON.stringify(queue.splice(0))
    if (navigator.sendBeacon) {
      navigator.sendBeacon(endpoint, body)
    }
  }

  return {
    report(metric: Metric) {
      queue.push(metric)
      if (timeoutId) clearTimeout(timeoutId)
      timeoutId = setTimeout(flush, 5000)
    },
  }
}

function initWebVitals(endpoint: string) {
  const reporter = createVitalsReporter(endpoint)

  onCLS(reporter.report)
  onINP(reporter.report)
  onLCP(reporter.report)
  onFCP(reporter.report)
  onTTFB(reporter.report)
}

export { initWebVitals }
```

### Thresholds de Web Vitals

| Metrica | Bom | Precisa Melhorar | Ruim |
|---|---|---|---|
| LCP (Largest Contentful Paint) | < 2.5s | 2.5s - 4.0s | > 4.0s |
| INP (Interaction to Next Paint) | < 200ms | 200ms - 500ms | > 500ms |
| CLS (Cumulative Layout Shift) | < 0.1 | 0.1 - 0.25 | > 0.25 |
| FCP (First Contentful Paint) | < 1.8s | 1.8s - 3.0s | > 3.0s |
| TTFB (Time to First Byte) | < 800ms | 800ms - 1800ms | > 1800ms |

---

## Structured Logging

```typescript
// shared/lib/logger.ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error'

interface LogEntry {
  level: LogLevel
  message: string
  timestamp: string
  context?: Record<string, unknown>
}

const LOG_LEVELS: Record<LogLevel, number> = {
  debug: 0,
  info: 1,
  warn: 2,
  error: 3,
}

function createLogger(minLevel: LogLevel = 'info') {
  const minLevelNum = LOG_LEVELS[minLevel]

  function log(level: LogLevel, message: string, context?: Record<string, unknown>) {
    if (LOG_LEVELS[level] < minLevelNum) return

    const entry: LogEntry = {
      level,
      message,
      timestamp: new Date().toISOString(),
      context,
    }

    switch (level) {
      case 'debug':
        console.debug(JSON.stringify(entry))
        break
      case 'info':
        console.info(JSON.stringify(entry))
        break
      case 'warn':
        console.warn(JSON.stringify(entry))
        break
      case 'error':
        console.error(JSON.stringify(entry))
        break
    }
  }

  return {
    debug: (message: string, context?: Record<string, unknown>) => log('debug', message, context),
    info: (message: string, context?: Record<string, unknown>) => log('info', message, context),
    warn: (message: string, context?: Record<string, unknown>) => log('warn', message, context),
    error: (message: string, context?: Record<string, unknown>) => log('error', message, context),
  }
}

const logger = createLogger(
  import.meta.env.DEV ? 'debug' : 'warn'
)

export { logger, createLogger }
```

---

## Analytics

### Event Tracking Pattern

```typescript
// shared/lib/analytics.ts

interface AnalyticsEvent {
  name: string
  properties?: Record<string, string | number | boolean>
}

interface AnalyticsProvider {
  track: (event: AnalyticsEvent) => void
  identify: (userId: string, traits?: Record<string, unknown>) => void
  page: (name: string, properties?: Record<string, unknown>) => void
}

function createAnalytics(): AnalyticsProvider {
  return {
    track({ name, properties }) {
      if (import.meta.env.DEV) {
        console.debug('[Analytics] track:', name, properties)
        return
      }
      // Integrar com PostHog, Plausible, etc.
    },

    identify(userId, traits) {
      if (import.meta.env.DEV) {
        console.debug('[Analytics] identify:', userId, traits)
        return
      }
    },

    page(name, properties) {
      if (import.meta.env.DEV) {
        console.debug('[Analytics] page:', name, properties)
        return
      }
    },
  }
}

const analytics = createAnalytics()

export { analytics }
export type { AnalyticsEvent, AnalyticsProvider }
```

### Hook de Analytics

```typescript
// shared/hooks/useAnalytics.ts
import { useCallback } from 'react'
import { analytics } from '@/shared/lib/analytics'

function useAnalytics() {
  const trackEvent = useCallback(
    (name: string, properties?: Record<string, string | number | boolean>) => {
      analytics.track({ name, properties })
    },
    []
  )

  return { trackEvent }
}

export { useAnalytics }
```

---

## Regras de Observabilidade

1. **NUNCA** logue PII (email completo, CPF, senha, token)
2. **SEMPRE** mascare dados sensiveis antes de enviar ao Sentry
3. **SEMPRE** configure `beforeSend` no Sentry para filtrar PII
4. **SEMPRE** monitore Web Vitals em producao
5. **NUNCA** use `console.log` em producao -- use logger estruturado
6. **SEMPRE** inclua contexto relevante nos logs (userId, action, page)
7. **USE** sampling para tracing e replays (nao capture 100% em producao)

---

## Checklist de Validacao

1. Sentry configurado com DSN e environment?
2. Source maps enviados ao Sentry para debugging?
3. PII removida de eventos do Sentry (beforeSend)?
4. Web Vitals monitorados (LCP, INP, CLS)?
5. Logger estruturado (nao console.log direto)?
6. Analytics events padronizados com nome + properties?
7. Error boundaries com tracking em subarvores criticas?
8. Sampling configurado para tracing e replays?
9. Contexto de usuario setado no Sentry apos login?
10. Debug logs desligados em producao?
