---
trigger: model_decision
scope: llm_ai_features
priority: critical
---

# LLM Integration Rules

> Regles specifiques pour les appels a des modeles de langage (Claude, GPT, Mistral, Gemini, etc.).
> Applique a toute feature qui utilise un LLM : chat, audit, generation, analyse, agents.

---

## 1. Principe fondamental

Les appels LLM sont **couteux, lents, non-deterministes, et faillibles**.
Chaque appel doit etre **protege, logue, budget-aware, et recupérable**.

---

## 2. Architecture d'un appel LLM

```
┌─────────┐     ┌──────────────┐     ┌───────────┐     ┌──────────┐
│ Service  │────>│ LLM Gateway  │────>│  Provider  │────>│  Model   │
│ metier   │     │ (wrapper)    │     │  (API)     │     │          │
│          │<────│              │<────│            │<────│          │
└─────────┘     └──────────────┘     └───────────┘     └──────────┘
                      │
                 ┌────┴────┐
                 │ Guards  │
                 │- Timeout│
                 │- Circuit│
                 │- Rate   │
                 │- Budget │
                 │- Log    │
                 └─────────┘
```

**Regle : ne jamais appeler un LLM directement depuis le service metier.**
Toujours passer par un wrapper/gateway interne qui applique les guardrails.

---

## 3. Separation des prompts (obligatoire)

### System prompt vs User prompt

```typescript
// OBLIGATOIRE — separation claire
const response = await llm.call({
    systemPrompt: AUDIT_SYSTEM_PROMPT,   // Regles, role, format de sortie
    userPrompt: userInput,                // Donnees de l'utilisateur
    // ...
});

// INTERDIT — tout melanger
const response = await llm.call({
    prompt: `Tu es un auditeur. ${userInput}. Reponds en JSON.`
});
```

### Structure d'un system prompt

```
1. ROLE       — Qui est l'IA dans ce contexte
2. OBJECTIF   — Quelle tache precise elle doit accomplir
3. REGLES     — Contraintes non-negociables
4. FORMAT     — Format de sortie attendu (JSON schema, structure)
5. EXEMPLES   — 1-2 exemples de input/output attendus (few-shot)
6. LIMITES    — Ce que l'IA ne doit PAS faire
```

### Stockage des prompts

```
src/features/<feature>/prompts/
├── <feature>-system.prompt.ts     # System prompt comme constante exportee
├── <feature>-examples.prompt.ts   # Exemples few-shot
└── <feature>-schemas.prompt.ts    # JSON schemas pour le format de sortie
```

**Regle : les prompts sont du code. Ils sont versionnes, testes, et revues.**

---

## 4. JSON Mode obligatoire pour les agents structurels

Quand le LLM doit retourner des donnees structurees :

```typescript
// OBLIGATOIRE — JSON mode + schema de validation
const response = await llm.call({
    systemPrompt: '...',
    userPrompt: '...',
    responseFormat: { type: 'json_object' },  // Force le JSON
});

// Toujours valider la reponse avec Zod
const parsed = AuditResultSchema.safeParse(JSON.parse(response.content));
if (!parsed.success) {
    logger.error({ event: 'llm_invalid_response', errors: parsed.error.issues });
    throw new AppError('LLM_INVALID_RESPONSE', 'AI returned unexpected format', 502, true);
}
```

**INTERDIT :**
- Faire confiance au LLM pour retourner du JSON valide sans `response_format`
- Parser la reponse sans validation Zod
- Utiliser du texte brut quand la reponse doit etre structuree

---

## 5. Guardrails (tous obligatoires)

### 5.1 Timeout

```typescript
const LLM_TIMEOUTS = {
    fast: 30_000,      // Completion simple, classification
    standard: 60_000,  // Generation standard, chat
    heavy: 120_000,    // Audit long, analyse de document, generation complexe
};
```

### 5.2 Circuit Breaker

```typescript
const llmCircuitBreaker = new CircuitBreaker({
    failureThreshold: 3,       // Ouvre apres 3 echecs
    resetTimeoutMs: 60_000,    // Re-essaie apres 1 minute
    monitorWindowMs: 120_000,  // Fenetre de monitoring 2 min
});
```

### 5.3 Concurrency Limiter

```typescript
const llmConcurrency = new ConcurrencyLimiter({
    maxConcurrent: 5,          // Max 5 appels simultanes
    queueSize: 20,             // Max 20 en file d'attente
    queueTimeoutMs: 30_000,    // Timeout de la file
});
```

### 5.4 Rate Limiter

```typescript
const llmRateLimiter = new RateLimiter({
    maxRequestsPerMinute: 60,  // Par instance de service
    maxTokensPerMinute: 100_000,
});
```

### 5.5 Budget Guard

```typescript
const llmBudget = new BudgetGuard({
    maxCostPerRequest: 0.50,      // $0.50 max par requete
    maxCostPerDay: 50.00,         // $50 max par jour
    alertThresholdPercent: 80,     // Alerte a 80% du budget
});
```

---

## 6. Logging des appels LLM (obligatoire)

Chaque appel LLM doit etre logue avec :

```typescript
logger.info({
    event: 'llm_call_completed',
    provider: 'anthropic',          // Provider utilise
    model: 'claude-sonnet-4-6',     // Modele exact
    feature: 'audit',               // Feature appelante
    operation: 'generate_audit',    // Operation specifique
    inputTokens: 1500,              // Tokens en entree
    outputTokens: 800,              // Tokens en sortie
    totalTokens: 2300,              // Total
    durationMs: 3200,               // Duree de l'appel
    estimatedCost: 0.023,           // Cout estime
    success: true,                  // Succes ou echec
    requestId: 'req-abc-123',       // Correlation ID
    cached: false,                  // Reponse depuis le cache ?
});
```

**Pour les echecs :**

```typescript
logger.error({
    event: 'llm_call_failed',
    provider: 'anthropic',
    model: 'claude-sonnet-4-6',
    feature: 'audit',
    operation: 'generate_audit',
    errorCode: 'RATE_LIMITED',
    errorMessage: 'Rate limit exceeded',
    durationMs: 150,
    requestId: 'req-abc-123',
    retriesRemaining: 2,
});
```

---

## 7. Selection de modele

### Strategie de selection

| Tache | Modele recommande | Justification |
|-------|------------------|---------------|
| Chat simple, Q&A | Modele rapide/petit (Haiku, Mistral Small, GPT-4o-mini) | Latence faible, cout minimal |
| Audit complexe, raisonnement | Modele fort (Opus, Mistral Large, GPT-4o) | Qualite de raisonnement |
| Classification, extraction | Modele rapide/petit | Tache simple, volume eleve |
| Generation de code | Modele fort | Precision critique |
| Summarization | Modele moyen (Sonnet, Mistral Medium) | Bon ratio qualite/cout |

### Pattern de fallback entre modeles

```typescript
async function callWithFallback(request: LLMRequest): Promise<LLMResponse> {
    const models = [
        { provider: 'anthropic', model: 'claude-sonnet-4-6' },   // Primaire
        { provider: 'openai', model: 'gpt-4o' },                  // Fallback 1
        { provider: 'mistral', model: 'mistral-large-latest' },   // Fallback 2
    ];

    for (const config of models) {
        try {
            return await llmGateway.call({ ...request, ...config });
        } catch (error) {
            logger.warn({
                event: 'llm_fallback',
                failedModel: config.model,
                nextModel: models[models.indexOf(config) + 1]?.model ?? 'none',
                error: error.message,
            });
            if (models.indexOf(config) === models.length - 1) throw error;
        }
    }
    throw new AppError('LLM_ALL_PROVIDERS_FAILED', 'All AI providers unavailable', 503, true);
}
```

---

## 8. Cache des reponses LLM

### Quand cacher

| Scenario | Cache ? | TTL |
|----------|---------|-----|
| Meme prompt exact + memes parametres | Oui | 1h - 24h |
| Chat conversationnel | Non | — |
| Classification/extraction (meme input) | Oui | 24h |
| Generation creative | Non | — |
| Audit d'un document (meme hash) | Oui | 7 jours |

### Implementation

```typescript
async function callWithCache(request: LLMRequest): Promise<LLMResponse> {
    const cacheKey = hashLLMRequest(request);  // Hash du prompt + params
    const cached = await cache.get(cacheKey);
    if (cached) {
        logger.info({ event: 'llm_cache_hit', cacheKey, feature: request.feature });
        return cached;
    }
    const response = await llmGateway.call(request);
    await cache.set(cacheKey, response, request.cacheTTL ?? 3600);
    return response;
}
```

---

## 9. Conditions d'arret (obligatoires pour les boucles agent)

Toute boucle agentic (agent qui s'appelle recursivement) DOIT avoir :

```typescript
const AGENT_LIMITS = {
    maxIterations: 10,          // Max 10 iterations
    maxTotalTokens: 50_000,     // Max 50k tokens total
    maxDurationMs: 300_000,     // Max 5 minutes
    maxCost: 2.00,              // Max $2 par execution
};
```

**INTERDIT :**
- Boucle agent sans `maxIterations`
- Boucle agent sans timeout global
- Boucle agent sans condition d'arret explicite dans le prompt
- Laisser un agent generer du cout illimite

---

## 10. Streaming

### Quand streamer

| Scenario | Streaming ? |
|----------|-------------|
| Chat interactif | Oui — UX critique |
| Generation longue (> 5s attendu) | Oui |
| Traitement backend (audit, batch) | Non |
| Classification rapide | Non |

### Regles de streaming

- Le frontend doit afficher un indicateur de "generation en cours"
- Le streaming doit avoir un timeout (meme si des tokens arrivent)
- L'annulation par l'utilisateur doit etre supportee
- Les erreurs mid-stream doivent etre gerees gracefully

---

## 11. Anti-Patterns

- Appeler un LLM directement sans wrapper/guardrails
- Pas de timeout sur un appel LLM
- Pas de circuit breaker sur le provider LLM
- Pas de logging des tokens/cout
- Faire confiance a la sortie JSON sans validation Zod
- Melanger system prompt et user prompt
- Boucle agent sans condition d'arret
- Pas de fallback quand le provider primaire tombe
- Stocker les prompts en dur dans le code metier (les isoler dans `/prompts/`)
- Pas de cache quand le meme input produit toujours le meme output
- Ignorer le cout des appels (pas de budget guard)
