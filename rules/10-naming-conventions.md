---
trigger: always_on
scope: all_projects
priority: critical
---

# Naming Conventions (Predictive Navigation)

> Ce fichier est le plus important pour la maintenance IA.
> Des conventions de nommage strictes permettent a un agent de **predire** les noms de fichiers,
> fonctions, routes, et modeles a partir du seul nom de la feature — sans jamais chercher.

---

## Principe fondamental

**Un agent IA doit pouvoir naviguer dans un projet de 500 fichiers sans faire un seul `grep`.**

Si la feature s'appelle `invoices`, l'agent sait instantanement :
- Le dossier backend : `src/features/invoices/`
- Le service : `invoices.service.ts`
- Les routes : `invoices.routes.ts`
- Le contract : `invoices.contract.ts`
- Le modele Prisma : `Invoice`
- La page frontend : `InvoicesPage.tsx`
- Le hook API : `useInvoicesApi.ts`
- Le test backend : `invoices.test.ts`
- Le test frontend : `InvoicesPage.test.tsx`
- La route API : `/api/invoices/*`
- La route frontend : `/apps/invoices`

---

## 1. Nom de feature

| Regle | Convention | Exemple |
|-------|-----------|---------|
| Format | `kebab-case`, singulier ou pluriel selon le domaine | `invoices`, `user-profile`, `napkin` |
| Pas d'abbreviations obscures | Nom complet et explicite | `knowledge-base` pas `kb` |
| Pas de prefixes techniques | Le nom decrit le domaine, pas la tech | `invoices` pas `invoice-module` |

Le nom de feature est la **racine** de toutes les autres conventions.

---

## 2. Structure de fichiers backend

```
src/features/<feature>/
├── index.ts                        # Export public unique
├── <feature>.routes.ts             # Routes HTTP (Fastify)
├── <feature>.service.ts            # Logique metier
├── <feature>.repository.ts         # Acces donnees (optionnel, si separation necessaire)
├── <feature>.types.ts              # Types internes (optionnel)
└── __tests__/
    ├── <feature>.service.test.ts   # Tests unitaires service
    └── <feature>.routes.test.ts    # Tests integration routes
```

### Conventions de nommage backend

| Element | Convention | Exemple pour `invoices` |
|---------|-----------|------------------------|
| Dossier feature | `kebab-case` | `src/features/invoices/` |
| Fichier routes | `<feature>.routes.ts` | `invoices.routes.ts` |
| Fichier service | `<feature>.service.ts` | `invoices.service.ts` |
| Fichier repository | `<feature>.repository.ts` | `invoices.repository.ts` |
| Fichier contract | dans `src/contracts/<feature>.contract.ts` | `invoices.contract.ts` |
| Export du service | `invoicesService` (camelCase) | `export const invoicesService = { ... }` |
| Fonction service | verbe + nom | `createInvoice()`, `getInvoiceById()`, `listInvoices()` |
| Fonction route register | `register<Feature>Routes` | `registerInvoicesRoutes(fastify)` |

### Nommage des fonctions service (verbes standard)

| Operation | Prefixe | Exemple |
|-----------|---------|---------|
| Lire un element | `get<Entity>ById` | `getInvoiceById(id)` |
| Lister des elements | `list<Entities>` | `listInvoices(filters)` |
| Creer | `create<Entity>` | `createInvoice(data)` |
| Mettre a jour | `update<Entity>` | `updateInvoice(id, data)` |
| Supprimer | `delete<Entity>` | `deleteInvoice(id)` |
| Rechercher | `search<Entities>` | `searchInvoices(query)` |
| Compter | `count<Entities>` | `countInvoices(filters)` |
| Verifier existence | `<entity>Exists` | `invoiceExists(id)` |

---

## 3. Structure de fichiers frontend

```
src/features/<feature>/
├── index.ts                        # Export public unique
├── README.md                       # Documentation feature
├── ui/
│   ├── <Feature>Page.tsx           # Page principale (PascalCase)
│   ├── <Feature>List.tsx           # Liste d'elements
│   ├── <Feature>Detail.tsx         # Detail d'un element
│   ├── <Feature>Form.tsx           # Formulaire creation/edition
│   ├── <Feature>Card.tsx           # Carte element
│   └── <Feature>Dialog.tsx         # Dialog/modal
├── hooks/
│   ├── use<Feature>.ts             # Hook orchestration principal
│   ├── use<Feature>Api.ts          # Hook API calls
│   └── use<Feature>Filters.ts     # Hook filtres/recherche (optionnel)
├── services/
│   └── <feature>.service.ts        # Appels API bruts
├── types/
│   └── <feature>.types.ts          # Types frontend
└── tests/
    ├── <Feature>Page.test.tsx      # Test smoke UI
    └── <feature>.service.test.ts   # Test service
```

### Conventions de nommage frontend

| Element | Convention | Exemple pour `invoices` |
|---------|-----------|------------------------|
| Dossier feature | `kebab-case` | `src/features/invoices/` |
| Composant page | `<Feature>Page.tsx` (PascalCase) | `InvoicesPage.tsx` |
| Composant liste | `<Feature>List.tsx` | `InvoicesList.tsx` |
| Composant detail | `<Feature>Detail.tsx` | `InvoiceDetail.tsx` |
| Composant formulaire | `<Feature>Form.tsx` | `InvoiceForm.tsx` |
| Composant carte | `<Feature>Card.tsx` | `InvoiceCard.tsx` |
| Hook principal | `use<Feature>.ts` | `useInvoices.ts` |
| Hook API | `use<Feature>Api.ts` | `useInvoicesApi.ts` |
| Service | `<feature>.service.ts` | `invoices.service.ts` |
| Types | `<feature>.types.ts` | `invoices.types.ts` |

### Nommage des composants React

| Pattern | Convention | Exemple |
|---------|-----------|---------|
| Page racine | `<Feature>Page` | `InvoicesPage` |
| Liste | `<Feature>List` | `InvoicesList` |
| Item de liste | `<Feature>ListItem` | `InvoiceListItem` |
| Detail | `<Feature>Detail` | `InvoiceDetail` |
| Formulaire | `<Feature>Form` | `InvoiceForm` |
| Dialog | `<Feature>Dialog` | `InvoiceDialog` |
| Widget carte | `<Feature>Card` | `InvoiceCard` |
| Squelette | `<Feature>Skeleton` | `InvoiceSkeleton` |
| Etat vide | `<Feature>EmptyState` | `InvoiceEmptyState` |
| Barre d'actions | `<Feature>Actions` | `InvoiceActions` |

---

## 4. Contracts (Zod schemas)

```
src/contracts/<feature>.contract.ts
```

### Conventions de nommage des schemas

| Element | Convention | Exemple |
|---------|-----------|---------|
| Fichier | `<feature>.contract.ts` | `invoices.contract.ts` |
| Schema entite | `<Entity>Schema` | `InvoiceSchema` |
| Schema creation | `Create<Entity>RequestSchema` | `CreateInvoiceRequestSchema` |
| Schema mise a jour | `Update<Entity>RequestSchema` | `UpdateInvoiceRequestSchema` |
| Schema reponse | `<Entity>ResponseSchema` | `InvoiceResponseSchema` |
| Schema liste reponse | `<Entities>ListResponseSchema` | `InvoicesListResponseSchema` |
| Type infere | `type <Entity> = z.infer<typeof <Entity>Schema>` | `type Invoice = z.infer<typeof InvoiceSchema>` |

### Exemple complet

```typescript
// invoices.contract.ts
import { z } from 'zod';

export const InvoiceSchema = z.object({
    id: z.string().uuid(),
    number: z.string(),
    amount: z.number(),
    status: z.enum(['draft', 'sent', 'paid', 'overdue']),
    tenantId: z.string().uuid(),
    createdAt: z.string().datetime(),
});

export const CreateInvoiceRequestSchema = z.object({
    number: z.string().min(1),
    amount: z.number().positive(),
});

export const UpdateInvoiceRequestSchema = CreateInvoiceRequestSchema.partial();

export const InvoicesListResponseSchema = z.array(InvoiceSchema);

export type Invoice = z.infer<typeof InvoiceSchema>;
export type CreateInvoiceRequest = z.infer<typeof CreateInvoiceRequestSchema>;
export type UpdateInvoiceRequest = z.infer<typeof UpdateInvoiceRequestSchema>;
```

---

## 5. Routes API

| Operation | Methode | Route | Exemple |
|-----------|---------|-------|---------|
| Lister | `GET` | `/<feature>` | `GET /invoices` |
| Lire un element | `GET` | `/<feature>/:id` | `GET /invoices/:id` |
| Creer | `POST` | `/<feature>` | `POST /invoices` |
| Mettre a jour | `PATCH` | `/<feature>/:id` | `PATCH /invoices/:id` |
| Remplacer | `PUT` | `/<feature>/:id` | `PUT /invoices/:id` |
| Supprimer | `DELETE` | `/<feature>/:id` | `DELETE /invoices/:id` |
| Action specifique | `POST` | `/<feature>/:id/<action>` | `POST /invoices/:id/send` |
| Sous-ressource | `GET` | `/<feature>/:id/<sub>` | `GET /invoices/:id/items` |

### Conventions de routes

- Pluriel pour les collections : `/invoices` (pas `/invoice`)
- Kebab-case pour les noms composes : `/knowledge-bases` (pas `/knowledgeBases`)
- Pas de verbe dans l'URL (le verbe HTTP suffit) : `DELETE /invoices/:id` (pas `POST /invoices/:id/delete`)
- Pas de `/api` dans les routes backend (le gateway le gere)

---

## 6. Base de donnees (Prisma)

| Element | Convention | Exemple |
|---------|-----------|---------|
| Nom du modele | `PascalCase`, singulier | `Invoice` |
| Nom de table | gere par Prisma (snake_case auto) | `invoice` |
| Champs | `camelCase` | `createdAt`, `tenantId`, `invoiceNumber` |
| Relations | nom du modele en camelCase | `invoice.items`, `user.invoices` |
| Enums | `PascalCase` | `InvoiceStatus` |
| Valeurs d'enum | `UPPER_SNAKE_CASE` ou `lowercase` selon convention Prisma | `DRAFT`, `SENT`, `PAID` |
| Migration | `kebab-case` descriptif | `add-invoice-table`, `add-status-to-invoice` |

---

## 7. Variables d'environnement

| Convention | Exemple |
|-----------|---------|
| `UPPER_SNAKE_CASE` | `DATABASE_URL` |
| Prefixe par service si ambigu | `BACKEND_PORT`, `GATEWAY_PORT` |
| Prefixe `VITE_` pour les variables frontend | `VITE_API_BASE_PATH` |
| Prefixe `SECRET_` pour les secrets sensibles | `SECRET_JWT_KEY` |
| Suffixe `_URL` pour les URLs | `DATABASE_URL`, `BACKEND_URL` |
| Suffixe `_KEY` ou `_TOKEN` pour les credentials | `API_KEY`, `JWT_TOKEN` |

---

## 8. Tests

| Element | Convention | Exemple |
|---------|-----------|---------|
| Test unitaire backend | `<feature>.service.test.ts` | `invoices.service.test.ts` |
| Test integration backend | `<feature>.routes.test.ts` | `invoices.routes.test.ts` |
| Test smoke frontend | `<Feature>Page.test.tsx` | `InvoicesPage.test.tsx` |
| Test composant | `<Component>.test.tsx` | `InvoiceCard.test.tsx` |
| Test contract | `<feature>.contract.test.ts` | `invoices.contract.test.ts` |
| Bloc describe | nom du module/fonction | `describe('invoicesService')` |
| Bloc it/test | phrase en anglais, comportement attendu | `it('should return 404 when invoice not found')` |

### Pattern de nommage des tests

```typescript
describe('<feature>Service', () => {
    describe('create<Entity>', () => {
        it('should create a valid <entity>', () => { ... });
        it('should reject invalid input', () => { ... });
        it('should enforce tenant isolation', () => { ... });
    });

    describe('get<Entity>ById', () => {
        it('should return <entity> when found', () => { ... });
        it('should return null when not found', () => { ... });
        it('should not return <entity> from another tenant', () => { ... });
    });
});
```

---

## 9. Commits et branches

### Branches
| Type | Convention | Exemple |
|------|-----------|---------|
| Feature | `feat/<feature-name>` | `feat/invoices` |
| Fix | `fix/<feature-name>-<description>` | `fix/invoices-missing-tenant` |
| Refactor | `refactor/<scope>` | `refactor/invoices-service` |
| Chore | `chore/<scope>` | `chore/update-deps` |

### Commits (Conventional Commits)
```
feat(invoices): add invoice creation endpoint
fix(invoices): enforce tenant isolation on list query
refactor(invoices): extract validation to contract
test(invoices): add negative path tests for create
docs(invoices): add ADR for invoice status workflow
```

Le scope entre parentheses = le nom de la feature (kebab-case).

---

## 10. Regles de verification

Un agent IA doit pouvoir repondre **OUI** a toutes ces questions :

- [ ] En lisant le nom du fichier, je sais ce qu'il contient ?
- [ ] En lisant le nom de la feature, je peux predire tous les noms de fichiers ?
- [ ] En lisant le nom d'une fonction, je sais ce qu'elle fait ?
- [ ] En lisant le nom d'une route, je sais quelle operation elle effectue ?
- [ ] Il n'y a aucune abbreviation ambigue ?
- [ ] Tous les fichiers du meme type suivent le meme pattern ?
