---
trigger: model_decision
scope: error_handling
priority: critical
---

# Error Handling, Resilience, and Recovery

> Format d'erreur standard, error boundaries, retry policies, circuit breakers, graceful degradation.
> Applique a toute operation qui peut echouer : API, DB, LLM, file IO, external services.

---

## 1. Format d'erreur standard (obligatoire)

**Toute** reponse d'erreur API doit suivre ce format :

```typescript
interface ApiError {
    error: true;                          // Toujours true pour les erreurs
    code: string;                         // Machine-readable, UPPER_SNAKE_CASE
    message: string;                      // Human-readable, pour l'UI
    statusCode: number;                   // HTTP status code
    requestId: string;                    // Correlation ID pour tracing
    details?: Record<string, unknown>;    // Validation errors, context additionnel
    retryable?: boolean;                  // Indique si le client peut retry
}
```

### Exemple concret

```typescript
// 404 — Ressource introuvable
{
    error: true,
    code: "INVOICE_NOT_FOUND",
    message: "Invoice not found",
    statusCode: 404,
    requestId: "req-abc-123",
    retryable: false
}

// 400 — Validation echouee
{
    error: true,
    code: "VALIDATION_ERROR",
    message: "Invalid request data",
    statusCode: 400,
    requestId: "req-abc-123",
    retryable: false,
    details: {
        fields: {
            amount: "Must be a positive number",
            number: "Required field"
        }
    }
}

// 503 — Service temporairement indisponible
{
    error: true,
    code: "LLM_SERVICE_UNAVAILABLE",
    message: "AI service is temporarily unavailable",
    statusCode: 503,
    requestId: "req-abc-123",
    retryable: true
}
```

---

## 2. Taxonomie des codes d'erreur

### Convention de nommage

Format : `<DOMAIN>_<DESCRIPTION>` en `UPPER_SNAKE_CASE`

### Codes standard (reutiliser en priorite)

| Code | Status | Signification |
|------|--------|---------------|
| `VALIDATION_ERROR` | 400 | Donnees d'entree invalides |
| `UNAUTHORIZED` | 401 | Token manquant, expire, ou invalide |
| `FORBIDDEN` | 403 | Authentifie mais pas autorise |
| `RESOURCE_NOT_FOUND` | 404 | Ressource demandee inexistante |
| `CONFLICT` | 409 | Conflit d'etat (doublon, version) |
| `RATE_LIMITED` | 429 | Trop de requetes |
| `INTERNAL_ERROR` | 500 | Erreur serveur inattendue |
| `SERVICE_UNAVAILABLE` | 503 | Dependance externe indisponible |
| `TIMEOUT` | 504 | Timeout sur une dependance |

### Codes specifiques au domaine

```
<FEATURE>_NOT_FOUND          — ex: INVOICE_NOT_FOUND
<FEATURE>_ALREADY_EXISTS     — ex: INVOICE_ALREADY_EXISTS
<FEATURE>_INVALID_STATUS     — ex: INVOICE_INVALID_STATUS
<FEATURE>_<ACTION>_FAILED    — ex: INVOICE_SEND_FAILED
TENANT_MISMATCH              — tentative d'acces cross-tenant
LLM_CALL_FAILED              — appel LLM echoue
LLM_TIMEOUT                  — timeout LLM
LLM_RATE_LIMITED             — rate limit LLM
```

---

## 3. Backend : Error Handler centralisé

### Middleware d'erreur global

```typescript
// src/shared/errors/app-error.ts
export class AppError extends Error {
    constructor(
        public readonly code: string,
        public readonly message: string,
        public readonly statusCode: number,
        public readonly retryable: boolean = false,
        public readonly details?: Record<string, unknown>
    ) {
        super(message);
        this.name = 'AppError';
    }
}

// Factories pour les erreurs courantes
export const notFound = (resource: string, id: string) =>
    new AppError(`${resource.toUpperCase()}_NOT_FOUND`, `${resource} not found`, 404, false, { id });

export const validationError = (fields: Record<string, string>) =>
    new AppError('VALIDATION_ERROR', 'Invalid request data', 400, false, { fields });

export const unauthorized = (reason?: string) =>
    new AppError('UNAUTHORIZED', reason ?? 'Authentication required', 401, false);

export const serviceUnavailable = (service: string) =>
    new AppError(`${service.toUpperCase()}_SERVICE_UNAVAILABLE`, `${service} is temporarily unavailable`, 503, true);
```

### Error handler Fastify

```typescript
// src/shared/errors/error-handler.ts
fastify.setErrorHandler((error, request, reply) => {
    const requestId = request.id;

    if (error instanceof AppError) {
        logger.warn({
            event: 'app_error',
            code: error.code,
            statusCode: error.statusCode,
            requestId,
            path: request.url,
        });
        return reply.status(error.statusCode).send({
            error: true,
            code: error.code,
            message: error.message,
            statusCode: error.statusCode,
            requestId,
            retryable: error.retryable,
            details: error.details,
        });
    }

    // Erreur inattendue — ne jamais exposer les details au client
    logger.error({
        event: 'unhandled_error',
        message: error.message,
        stack: error.stack,
        requestId,
        path: request.url,
    });
    return reply.status(500).send({
        error: true,
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred',
        statusCode: 500,
        requestId,
        retryable: false,
    });
});
```

---

## 4. Frontend : Error Handling systématique

### Error Boundary React (obligatoire a la racine)

```tsx
// src/shared/ui/ErrorBoundary.tsx
class ErrorBoundary extends React.Component<Props, State> {
    state = { hasError: false, error: null };

    static getDerivedStateFromError(error: Error) {
        return { hasError: true, error };
    }

    componentDidCatch(error: Error, info: React.ErrorInfo) {
        logger.error({ event: 'react_error_boundary', error: error.message, componentStack: info.componentStack });
    }

    render() {
        if (this.state.hasError) {
            return <ErrorFallback error={this.state.error} onReset={() => this.setState({ hasError: false })} />;
        }
        return this.props.children;
    }
}
```

### Placement des Error Boundaries

```tsx
// App.tsx — structure d'error boundaries
<ErrorBoundary>                          {/* Niveau app : catch-all */}
  <AuthProvider>
    <Routes>
      <Route element={<MainLayout />}>
        <ErrorBoundary>                  {/* Niveau module : isole les crashes */}
          <Route path="/apps/invoices/*" element={<InvoicesPage />} />
        </ErrorBoundary>
      </Route>
    </Routes>
  </AuthProvider>
</ErrorBoundary>
```

### Hook d'erreur API standardise

```typescript
// src/shared/hooks/useApiError.ts
export const useApiError = () => {
    const { toast } = useToast();
    const { logout } = useAuth();

    const handleError = (error: unknown) => {
        if (error instanceof ApiError) {
            if (error.statusCode === 401) {
                logout();
                return;
            }
            toast({
                variant: 'destructive',
                title: errorTitleMap[error.code] ?? 'Error',
                description: error.message,
            });
            return;
        }
        // Erreur inconnue
        toast({
            variant: 'destructive',
            title: 'Unexpected error',
            description: 'Something went wrong. Please try again.',
        });
    };

    return { handleError };
};
```

---

## 5. Retry Policy (obligatoire pour les appels externes)

### Regles de retry

| Type d'erreur | Retry ? | Strategie |
|---------------|---------|-----------|
| `400` Bad Request | Non | Corriger l'input |
| `401` Unauthorized | Non | Re-authentifier |
| `403` Forbidden | Non | Pas de retry |
| `404` Not Found | Non | Pas de retry |
| `409` Conflict | Parfois | Retry apres re-fetch de l'etat |
| `429` Rate Limited | Oui | Backoff exponentiel + respect du `Retry-After` |
| `500` Server Error | Oui | 2-3 retries avec backoff |
| `502/503` Unavailable | Oui | Backoff exponentiel |
| `504` Timeout | Oui | Retry avec timeout augmente |
| Network Error | Oui | Backoff exponentiel |

### Implementation standard

```typescript
interface RetryConfig {
    maxRetries: number;       // Defaut: 3
    baseDelayMs: number;      // Defaut: 1000
    maxDelayMs: number;       // Defaut: 30000
    backoffMultiplier: number; // Defaut: 2
}

async function withRetry<T>(
    operation: () => Promise<T>,
    config: RetryConfig = { maxRetries: 3, baseDelayMs: 1000, maxDelayMs: 30000, backoffMultiplier: 2 }
): Promise<T> {
    let lastError: Error;
    for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
        try {
            return await operation();
        } catch (error) {
            lastError = error as Error;
            if (!isRetryable(error) || attempt === config.maxRetries) throw error;
            const delay = Math.min(
                config.baseDelayMs * Math.pow(config.backoffMultiplier, attempt),
                config.maxDelayMs
            );
            await sleep(delay + Math.random() * 200); // Jitter pour eviter thundering herd
        }
    }
    throw lastError!;
}
```

---

## 6. Circuit Breaker (obligatoire pour les dependances externes)

### Quand utiliser un circuit breaker

- Appels a des APIs externes (LLM, services tiers)
- Appels a des services internes non-critiques
- Operations couteuses en resources

### Etats du circuit

```
CLOSED (normal) → erreurs repetees → OPEN (refuse les appels)
OPEN → apres cooldown → HALF_OPEN (laisse passer 1 appel test)
HALF_OPEN → succes → CLOSED | echec → OPEN
```

### Configuration standard

```typescript
interface CircuitBreakerConfig {
    failureThreshold: number;   // Nombre d'echecs avant ouverture (defaut: 5)
    resetTimeoutMs: number;     // Temps avant tentative half-open (defaut: 30000)
    monitorWindowMs: number;    // Fenetre de comptage d'erreurs (defaut: 60000)
}
```

---

## 7. Timeouts standard

| Operation | Timeout | Justification |
|-----------|---------|---------------|
| API call interne (gateway → backend) | 30s | Operations CRUD standard |
| Requete DB simple | 5s | Lecture/ecriture standard |
| Requete DB complexe (report, agregation) | 30s | Requetes lourdes |
| Appel LLM (generation courte) | 60s | Reponse LLM standard |
| Appel LLM (generation longue) | 120s | Audit, analyse, generation de documents |
| Upload fichier | 120s | Fichiers potentiellement lourds |
| Health check | 5s | Doit etre rapide |
| Startup dependency check | 5s par tentative | Avec retry toutes les 5s |

**Regle : chaque appel externe DOIT avoir un timeout. Aucun appel sans limite.**

---

## 8. Graceful Degradation

### Principe

Quand un service non-critique tombe, l'application continue de fonctionner avec des capacites reduites au lieu de crasher.

### Patterns

| Situation | Degradation |
|-----------|-------------|
| LLM indisponible | Afficher message "AI temporairement indisponible", desactiver les features AI, les autres features marchent |
| Service de recherche down | Fallback sur filtrage basique en DB |
| Service de notification down | Queue les notifications pour envoi ulterieur |
| Service d'analytics down | Ignorer silencieusement (non-critique) |

### Regle d'implementation

```typescript
// Pattern: try the enhanced path, fallback to basic
async function getItemsWithSearch(query: string): Promise<Item[]> {
    try {
        return await searchService.search(query);  // Service avance
    } catch (error) {
        logger.warn({ event: 'search_degraded', error: error.message });
        return await itemsRepository.findByFilter(query);  // Fallback basique
    }
}
```

---

## 9. Idempotence (obligatoire pour les mutations)

### Principe

Une operation idempotente peut etre executee plusieurs fois avec le meme resultat. C'est **critique** pour la maintenance IA : si un agent relance une operation par erreur, le systeme doit etre safe.

### Patterns d'idempotence

| Operation | Pattern |
|-----------|---------|
| `POST /items` (creation) | Header `Idempotency-Key`, deduplication en DB |
| `PUT /items/:id` (remplacement) | Naturellement idempotent |
| `PATCH /items/:id` (update) | Naturellement idempotent si pas de compteurs relatifs |
| `DELETE /items/:id` | Retourner 200 meme si deja supprime (pas 404) |
| Envoi d'email/notification | Deduplication par cle unique |
| Appel LLM | Cache par hash du prompt + parametres |

### Implementation

```typescript
// Idempotency key pour les creations
fastify.post('/invoices', async (request, reply) => {
    const idempotencyKey = request.headers['idempotency-key'];
    if (idempotencyKey) {
        const existing = await invoicesService.findByIdempotencyKey(idempotencyKey);
        if (existing) return reply.status(200).send(existing); // Deja cree
    }
    const created = await invoicesService.create({ ...body, idempotencyKey });
    return reply.status(201).send(created);
});
```

---

## 10. Anti-Patterns

- Formats d'erreur inconsistants entre features
- Exposer des stack traces ou details internes au client
- Pas de timeout sur les appels externes
- Retry infini sans backoff
- Pas de circuit breaker sur les services externes
- `catch (e) { /* silent */ }` — ne jamais avaler une erreur silencieusement
- Pas d'Error Boundary React — un crash dans un composant casse toute l'app
- Retourner 404 sur un DELETE d'element deja supprime (non-idempotent)
- Log l'erreur mais ne pas la remonter (l'UI reste en loading infini)
