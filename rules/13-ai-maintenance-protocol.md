---
trigger: always_on
scope: ai_agents
priority: critical
---

# AI Maintenance Protocol

> Protocole pour la maintenance de code par une IA : diagnostic, modification, depreciation, refactoring.
> Ce fichier couvre les 80% du travail d'un agent IA : maintenir du code existant (pas en creer).

---

## 1. Principe fondamental

**La creation represente 20% du travail. La maintenance represente 80%.**

Un agent IA qui maintient un projet doit :
- Comprendre avant de modifier
- Diagnostiquer avant de corriger
- Mesurer l'impact avant de changer
- Verifier apres chaque modification
- Ne jamais casser ce qui marche

---

## 2. Protocole de diagnostic de bug

### Etape 1 : Comprendre le symptome

```
AVANT de toucher au code :
1. Quel est le comportement observe ? (erreur, comportement incorrect, crash)
2. Quel est le comportement attendu ?
3. Depuis quand ? (commit, deploy, changement recent)
4. Est-ce reproductible ? (toujours, parfois, conditions specifiques)
5. Quel est l'impact ? (bloquant, degradé, cosmetique)
```

### Etape 2 : Localiser la source

```
STRATEGIE DE LOCALISATION (dans cet ordre) :
1. LOGS     — Chercher les erreurs dans les logs structures (requestId, event, error)
2. TRACE    — Suivre le flux de donnees du point d'entree au point de sortie
3. DIFF     — Comparer avec le dernier etat fonctionnel (git log, git diff)
4. ISOLATE  — Reproduire avec le minimum de code/config necessaire
5. BISECT   — Si l'etape 3 ne suffit pas, git bisect pour trouver le commit fautif
```

### Etape 3 : Categoriser le bug

| Categorie | Exemples | Strategie de fix |
|-----------|----------|-----------------|
| **Data** | Requete mal formee, schema invalide, tenant leak | Fix au niveau du contract/validation |
| **Logic** | Condition inversee, off-by-one, race condition | Fix dans le service/hook |
| **Integration** | Route cassee, header manquant, gateway misconfigured | Fix au niveau de la couche d'integration |
| **State** | UI desynchronisee, cache stale, session corrompue | Fix dans la gestion d'etat |
| **Config** | Variable env manquante, URL incorrecte, feature flag | Fix dans la configuration |
| **Regression** | Un changement recent a casse un comportement existant | Revert ou fix cible + test de non-regression |

### Etape 4 : Corriger avec precision chirurgicale

```
REGLES DE FIX :
1. Corriger UNIQUEMENT le bug — pas de refactoring opportuniste
2. Ecrire le test QUI AURAIT ATTRAPE LE BUG avant de commencer le fix
3. Le test doit echouer AVANT le fix et passer APRES
4. Verifier que le fix ne casse pas les tests existants
5. Documenter la root cause dans le commit message
```

### Etape 5 : Prevenir la recurrence

```
APRES le fix :
1. Le test de non-regression est-il en place ?
2. La root cause est-elle documentee ? (commit message ou ADR si systemique)
3. Y a-t-il d'autres endroits avec le meme pattern vulnerable ?
4. Faut-il ajouter une regle de lint ou un guard pour prevenir la recurrence ?
```

---

## 3. Protocole de modification d'une feature existante

### Ajouter un champ a une entite

```
CHECKLIST — AJOUT DE CHAMP :
1. [ ] Schema Prisma    → Ajouter le champ (optionnel avec default si possible)
2. [ ] Migration        → npx prisma migrate dev --name add-<field>-to-<entity>
3. [ ] Prisma Generate  → npx prisma generate
4. [ ] Contract backend → Ajouter le champ au schema Zod (optionnel pour compat)
5. [ ] Service backend  → Mettre a jour les queries qui lisent/ecrivent l'entite
6. [ ] Routes backend   → Verifier que le champ est expose si necessaire
7. [ ] Contract frontend→ Miroir du contract backend
8. [ ] Service frontend → Mettre a jour les appels API si necessaire
9. [ ] UI frontend      → Afficher/editer le nouveau champ
10.[ ] Tests            → Mettre a jour les tests existants + ajouter test pour le nouveau champ
11.[ ] Baseline         → npm run baseline → VERT
```

### Modifier un comportement existant

```
CHECKLIST — MODIFICATION DE COMPORTEMENT :
1. [ ] Identifier TOUS les consumers du comportement actuel
2. [ ] Ecrire les tests du nouveau comportement AVANT de modifier le code
3. [ ] Verifier que les tests actuels passent encore (ou les adapter si le changement est voulu)
4. [ ] Modifier le code
5. [ ] Verifier que les nouveaux tests passent
6. [ ] Verifier que les tests des consumers passent
7. [ ] Si le changement est breaking → ADR + migration plan pour les consumers
8. [ ] Baseline → VERT
```

### Ajouter un endpoint a une feature existante

```
CHECKLIST — AJOUT ENDPOINT :
1. [ ] Contract    → Ajouter les schemas Zod (request + response)
2. [ ] Service     → Ajouter la logique metier
3. [ ] Route       → Ajouter la route dans le fichier de routes existant
4. [ ] Test        → Test positif + test negatif + test tenant isolation
5. [ ] Contract FE → Miroir si le frontend consomme
6. [ ] Service FE  → Ajouter l'appel dans le hook API
7. [ ] Baseline    → VERT
```

---

## 4. Protocole de refactoring safe

### Regle d'or du refactoring

**Un refactoring ne change JAMAIS le comportement externe.**
Si le comportement change, ce n'est pas un refactoring, c'est une modification.

### Methode "Strangler Fig"

Pour les gros refactorings, utiliser l'approche incrementale :

```
1. CREER la nouvelle implementation a cote de l'ancienne
2. TESTER la nouvelle implementation isolement
3. ROUTER progressivement le trafic vers la nouvelle implementation
4. VERIFIER que le comportement est identique
5. SUPPRIMER l'ancienne implementation une fois confirmee
```

### Checklist de refactoring

```
AVANT :
1. [ ] Les tests existants couvrent-ils le comportement actuel ? Si non, les ajouter D'ABORD
2. [ ] Le refactoring est-il decoupable en etapes atomiques ?
3. [ ] Chaque etape preserve-t-elle le comportement externe ?

PENDANT :
4. [ ] Une etape = un commit
5. [ ] Baseline verte apres chaque etape
6. [ ] Pas de changement de comportement glisse dans le refactoring

APRES :
7. [ ] Tous les tests existants passent sans modification
8. [ ] Pas de code mort laisse derriere
9. [ ] Les imports/exports sont corrects
```

---

## 5. Protocole de depreciation

### Comment deprecier une feature/API/champ

```
PHASE 1 — SIGNAL (sprint N)
1. Marquer comme deprecie dans le code :
   /** @deprecated Use newMethod() instead. Will be removed in v2.1. */
2. Ajouter un warning dans les logs quand utilise
3. Documenter dans l'ADR
4. Informer les consumers connus

PHASE 2 — REDIRECT (sprint N+1)
1. Ajouter un wrapper qui redirige vers le nouveau code
2. Logger chaque utilisation du chemin deprecie
3. Verifier que tous les consumers connus sont migres

PHASE 3 — REMOVE (sprint N+2 minimum)
1. Verifier que les logs ne montrent plus d'utilisation
2. Supprimer le code deprecie
3. Supprimer les tests associes
4. Mettre a jour la documentation
```

### Regles de depreciation

- **Jamais de suppression sans phase de depreciation** (sauf code interne non-expose)
- **Minimum 1 version/sprint** entre le signal et la suppression
- **Toujours fournir l'alternative** dans le message de depreciation
- **Logger chaque appel deprecie** pour mesurer l'utilisation restante

---

## 6. Navigation predictive dans le code

### Comment un agent IA doit naviguer

Quand un agent recoit une tache sur la feature `invoices` :

```
ETAPE 1 — LOCALISER (sans grep, par convention)
  Backend:   src/features/invoices/
  Frontend:  src/features/invoices/
  Contract:  src/contracts/invoices.contract.ts
  Schema DB: prisma/schema.prisma → model Invoice
  Tests:     src/features/invoices/__tests__/
  Routes:    /api/invoices/*

ETAPE 2 — COMPRENDRE (lire dans cet ordre)
  1. README.md de la feature
  2. index.ts (surface publique)
  3. <feature>.contract.ts (forme des donnees)
  4. <feature>.service.ts (logique metier)
  5. <feature>.routes.ts (points d'entree HTTP)
  6. Tests (comportement attendu)

ETAPE 3 — IDENTIFIER L'IMPACT
  1. Qui importe depuis features/invoices/index.ts ?
  2. Quels autres services appellent invoicesService ?
  3. Quels composants frontend consomment l'API ?
  4. Y a-t-il des jobs/cron/webhooks qui touchent cette feature ?
```

---

## 7. Impact Analysis avant modification

Avant toute modification, l'agent doit evaluer l'impact :

### Matrice d'impact

| Ce que je modifie | Impact potentiel | Verification necessaire |
|-------------------|-----------------|------------------------|
| Schema Prisma | DB, backend service, contract, frontend | Migration + full stack test |
| Contract Zod | Backend routes, frontend services | Tests contract + consumers |
| Service backend | Routes qui l'utilisent | Tests service + integration |
| Route backend | Gateway proxy, frontend calls | Test e2e du flux |
| Composant frontend | Pages qui l'utilisent | Test UI + smoke test |
| Hook frontend | Composants qui l'utilisent | Test du hook + composants |
| CSS/Tailwind tokens | Tous les composants | Visual review |
| Variables d'env | Startup, config, deploy | Test startup + deploy |

### Regles d'impact

```
MODIFICATION A FAIBLE IMPACT (peut etre faite rapidement) :
- Ajouter un champ optionnel a un contract
- Modifier la logique interne d'un service (meme inputs/outputs)
- Ajouter un composant dans une feature existante
- Ajouter un test

MODIFICATION A FORT IMPACT (necessite plan + verification etendue) :
- Modifier un contract existant (meme additivement)
- Modifier un schema Prisma
- Modifier une route (path, method, auth)
- Modifier le comportement du gateway
- Modifier la gestion d'auth
```

---

## 8. Feature Manifest (carte du projet)

Chaque projet devrait maintenir un fichier `FEATURES.md` a la racine :

```markdown
# Features Manifest

| Feature | Status | Backend | Frontend | DB Model | API Routes | Owner |
|---------|--------|---------|----------|----------|------------|-------|
| invoices | active | src/features/invoices/ | src/features/invoices/ | Invoice | /api/invoices/* | team-billing |
| napkin | active | src/features/napkin/ | src/features/napkin/ | Board, Cell | /api/napkin/* | team-collab |
| auth | active | src/features/auth/ | src/features/auth/ | User, Session | /api/login, /api/me | team-platform |
| roadmap | beta | src/features/roadmap/ | src/features/roadmap/ | Initiative | /api/roadmap/* | team-product |
```

**Ce manifest permet a un agent IA de comprendre tout le projet en 10 secondes.**

---

## 9. Pattern de code auto-descriptif

### Le code doit se documenter par sa structure

```typescript
// BIEN — la structure raconte l'histoire
export const invoicesService = {
    // Queries (lecture)
    getInvoiceById,
    listInvoices,
    searchInvoices,
    countInvoices,

    // Commands (ecriture)
    createInvoice,
    updateInvoice,
    deleteInvoice,

    // Actions metier
    sendInvoice,
    markAsPaid,
    generatePdf,
};

// MAL — liste plate sans organisation
export { getInvoice, create, update, del, send, paid, pdf, list, search, count };
```

### Conventions de code auto-descriptif

| Regle | Exemple |
|-------|---------|
| Noms de fonctions = verbe + nom | `createInvoice()` pas `invoice()` |
| Noms de booleens = question | `isActive`, `hasPermission`, `canDelete` |
| Noms de listes = pluriel | `invoices`, `items` pas `invoiceList` |
| Noms de maps/dictionnaires = `<key>By<value>` | `invoiceById`, `usersByTenantId` |
| Noms d'erreurs = descriptif | `InvoiceNotFoundError` pas `Error404` |
| Noms de constantes = intent | `MAX_RETRY_ATTEMPTS` pas `NUM_3` |

---

## 10. Checklist de maintenance quotidienne

Quand un agent IA commence une session de maintenance :

```
1. [ ] Lire le FEATURES.md / manifest pour comprendre le projet
2. [ ] Lire les rules applicables a la tache
3. [ ] Identifier la feature concernee par convention de nommage
4. [ ] Lire le README de la feature
5. [ ] Lire les tests existants (comportement attendu)
6. [ ] Evaluer l'impact de la modification
7. [ ] Appliquer le protocole adapte (bug fix / modification / refactoring)
8. [ ] Ecrire/mettre a jour les tests
9. [ ] Verifier le baseline
10.[ ] Documenter si necessaire (commit, ADR)
```

---

## 11. Anti-Patterns de maintenance

- Modifier du code sans lire les tests existants d'abord
- Faire un "quick fix" sans comprendre la root cause
- Refactorer en meme temps qu'un bug fix (2 problemes = 2 commits)
- Supprimer du code "inutile" sans verifier qui l'appelle
- Ne pas ecrire le test de non-regression apres un bug fix
- Modifier un contract sans verifier les consumers
- Faire confiance a la documentation quand elle contredit le code
- Changer le comportement externe dans un commit intitule "refactor"
- Ne pas mettre a jour le manifest/README apres une modification significative
- Ignorer les warnings de depreciation dans les logs
