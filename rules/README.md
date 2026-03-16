# True Rules — AI Development Standards

> Ensemble de regles universelles pour le developpement d'applications IA.
> Comprehensible par tous les agents IA (Claude, GPT, Gemini, Mistral, etc.).
> Objectif : code production-ready, ultra-maintainable par IA, zero oversight humain necessaire.

---

## Hierarchie de priorite

Quand les regles entrent en conflit, suivre cet ordre :

**Correctness > Security > Operability > Maintainability > Performance > Convenience**

---

## Structure des fichiers

### Regles fondamentales (toujours actives)

| # | Fichier | Role | Trigger |
|---|---------|------|---------|
| 00 | `00-engineering-constitution.md` | Constitution engineering — principes, PLAN>ACT>VERIFY>DOCUMENT, AI-maintainability patterns | **Toujours** |
| 01 | `01-reasoning-framework.md` | Framework de raisonnement IA — planification, diagnostic de bugs, decision sous incertitude | **Toujours** |
| 10 | `10-naming-conventions.md` | **Conventions de nommage strictes** — navigation predictive, noms de fichiers/fonctions/routes/modeles | **Toujours** |

### Regles par domaine (declenchees selon le contexte)

| # | Fichier | Role | Quand l'appliquer |
|---|---------|------|-------------------|
| 02 | `02-feature-architecture.md` | Architecture features — structure, modules, workflow full-stack, manifest | **Creation/modification** de feature |
| 03 | `03-api-contracts-gateway.md` | API contracts, gateway, communication, format d'erreur standard, retry policies | **Appels API**, routes, contracts, gateway |
| 04 | `04-database-workflow.md` | Migrations Prisma, tenant isolation, indexation, patterns de requetes performants | **Schema**, migrations, donnees tenant |
| 05 | `05-ui-ux-design-system.md` | Design system, Shadcn, Tailwind, state management, error boundaries, accessibilite | **Frontend/UI** creation ou modification |
| 06 | `06-security-audit.md` | Audit securite OWASP Top 10, analyse de surface d'attaque, reporting vulnerabilites | **Revue de securite** |
| 07 | `07-quality-gates.md` | Testing pyramid, baseline, ADR, documentation, definition of done | **Implementation non-triviale** |
| 08 | `08-security-dependencies-startup.md` | Dependencies, secrets, startup, guardrails LLM, monitoring/alerting | **Deps**, auth, admin, startup, LLM |
| 09 | `09-deployment-workflow.md` | Docker Compose, startup cascade, post-deploy verification, rollback | **Deploiement** |

### Regles specifiques IA

| # | Fichier | Role | Quand l'appliquer |
|---|---------|------|-------------------|
| 11 | `11-error-handling.md` | Format d'erreur standard, error boundaries, retry, circuit breaker, idempotence, graceful degradation | **Toute operation** qui peut echouer |
| 12 | `12-llm-integration.md` | Appels LLM : prompts, guardrails, JSON mode, fallback, cache, budget, logging | **Features IA** utilisant des LLM |
| 13 | `13-ai-maintenance-protocol.md` | Protocole de maintenance IA : diagnostic, modification, refactoring, depreciation, manifest | **Maintenance** de code existant (80% du travail) |

### Prompts

| Fichier | Role |
|---------|------|
| `prompts/CREATE_FEATURE.md` | Template strict pour la creation d'une nouvelle feature |

---

## Comment utiliser ces regles

### Pour un agent IA (protocole de session)

```
1. BOOT — Lire obligatoirement :
   - 00-engineering-constitution.md (principes)
   - 01-reasoning-framework.md (comment raisonner)
   - 10-naming-conventions.md (comment nommer)
   - 13-ai-maintenance-protocol.md (comment maintenir)
   - FEATURES.md du projet (carte des features)

2. CONTEXTE — Identifier les regles applicables :
   - Creation de feature ? → 02 + 05 + prompts/CREATE_FEATURE.md
   - Appels API ? → 03 + 11
   - Base de donnees ? → 04
   - UI/Frontend ? → 05
   - Integration LLM ? → 12 + 08
   - Bug fix ? → 01 (diagnostic) + 13 (maintenance)
   - Security review ? → 06

3. EXECUTION — Suivre PLAN > ACT > VERIFY > DOCUMENT

4. VERIFICATION — Baseline verte + tests + documentation
```

### Pour un developpeur humain

1. Lire la constitution (`00`) pour comprendre les principes
2. Lire les conventions de nommage (`10`) — c'est la cle de la maintenabilite
3. Consulter les regles specifiques selon la tache
4. Utiliser les checklists avant de soumettre du code

---

## Principes cles

### Production-ready by design
- **Contract-first** : les schemas Zod definissent les frontieres
- **Error handling standard** : meme format d'erreur partout, retry/circuit breaker inclus
- **Observabilite** : logs structures, health checks, request tracing
- **Resilience** : circuit breakers, graceful degradation, timeouts partout

### AI-maintainable by design
- **Nommage predictif** : un agent predit tous les fichiers depuis le nom de la feature
- **Feature manifest** : `FEATURES.md` donne la carte complete du projet en secondes
- **Zero magic** : pas de comportement implicite, pas de code "clever"
- **Operations idempotentes** : safe a re-executer par erreur
- **Protocols de maintenance** : diagnostic, modification, refactoring, depreciation

### Development at scale
- **Feature isolation** : 1 feature = 1 dossier = 1 point d'entree public
- **Backend-first** : pour les features full-stack, commencer par les contracts
- **Gateway obligatoire** : jamais d'appel direct au backend
- **Tests obligatoires** : pas de code sans tests, pas de feature sans smoke test
- **Migrations obligatoires** : tout changement de schema = fichier de migration
- **Baseline verte** : jamais declarer "done" avec un baseline rouge
- **ADR pour les decisions** : documenter le "pourquoi", pas le "quoi"

---

## Cartographie des cross-references

```
00-constitution ←── reference par tous les fichiers
    ├── 10-naming ←── reference par 02, 13
    ├── 11-error-handling ←── reference par 03, 05, 08, 12
    ├── 12-llm-integration ←── reference par 08
    └── 13-ai-maintenance ←── reference par 02

02-feature-architecture
    ├── depends: 10-naming (conventions de nommage)
    ├── depends: 03-api-contracts (communication)
    ├── depends: 04-database (si full-stack)
    └── depends: 05-ui-ux (si frontend)

03-api-contracts ←→ 11-error-handling (format d'erreur)
04-database ←→ 08-security (tenant isolation)
05-ui-ux ←→ 11-error-handling (error boundaries)
08-security ←→ 12-llm-integration (guardrails)
```
