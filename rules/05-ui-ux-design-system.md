---
trigger: model_decision
scope: frontend_ui
---

# UI/UX Design System and Frontend Standards

> Rules for building consistent, accessible, and premium frontend interfaces.
> Applies whenever creating, editing, or reviewing frontend UI.

---

## 1. Design Philosophy

- **Premium and clean** — modern, professional, polished interfaces
- **Consistent** — systematic reuse of components, tokens, spacing. No ad-hoc styles
- **Accessible** — keyboard navigable, screen reader compatible. A11y is not optional
- **Component-first** — think in reusable, autonomous components, not monolithic pages
- **Responsive** — must work on desktop and tablet minimum; mobile simplified but not broken

---

## 2. Technical Stack

| Layer | Technology |
|-------|-----------|
| Framework | React 18+ (TypeScript) |
| Styling | Tailwind CSS (utility-first) |
| Components | Shadcn/ui (Radix UI based) |
| Icons | Lucide React |
| Animations | CSS transitions (Tailwind) or Framer Motion (complex interactions) |

---

## 3. Component Library: Shadcn UI First

**Golden rule:** Never recreate a standard component if a Shadcn equivalent exists.

Before creating any UI element, check `src/components/ui/` for existing primitives:

| Category | Components |
|----------|-----------|
| Actions | `Button` (default, secondary, ghost, outline, destructive) |
| Forms | `Input`, `Textarea`, `Select`, `Label`, `Form` |
| Feedback | `Toast`, `Alert`, `Badge`, `Skeleton` |
| Overlay | `Dialog` (modals), `Sheet` (sidebars), `Popover` |
| Layout | `Card`, `ScrollArea`, `Separator`, `Tabs` |
| Data | `Table`, `Avatar` |

### Adding a new Shadcn component
```bash
npx shadcn@latest add <component-name>
```

---

## 4. Styling Rules

### Tailwind only — no custom CSS files

- **FORBIDDEN:** feature-specific `.css` or `.scss` files
- **FORBIDDEN:** raw hex colors in components (e.g., `#3B82F6`)
- **MANDATORY:** use theme tokens from CSS variables via Tailwind classes
- **MANDATORY:** use `cn()` utility for conditional class composition

```tsx
import { cn } from "@/lib/utils";

<div className={cn(
    "p-4 border rounded-lg",
    isActive && "bg-primary text-primary-foreground"
)}>
```

### Theme Tokens

| Purpose | Tokens |
|---------|--------|
| Backgrounds | `bg-background`, `bg-card`, `bg-popover`, `bg-muted` |
| Text | `text-foreground`, `text-muted-foreground`, `text-primary` |
| Borders | `border-border`, `border-input` |
| Actions | `bg-primary`, `text-primary-foreground` |
| Danger | `bg-destructive`, `text-destructive-foreground` |

---

## 5. Platform Geometry

Standard dimensions for visual consistency across all modules:

| Element | Value |
|---------|-------|
| App shell header | `h-16` |
| Main content width | `max-w-7xl` |
| Horizontal page padding | `px-4 sm:px-6 lg:px-8` |
| Standard card padding | `p-6` |
| Compact card padding | `p-4` |
| Layout gaps | `gap-4` or `gap-6` |
| Border radius | `rounded-lg`, `rounded-xl`, `rounded-2xl` |
| Surface treatment | subtle `border` + `shadow-sm` |

Do not invent a different spacing or radius scale inside one module.

---

## 6. Visual Language

- **Primary action color:** indigo
- **Neutrals:** slate palette for backgrounds and borders
- **Surfaces:** white cards over pale slate page background
- **Header:** `border-b` + blur for sticky header
- **Shadows:** restrained, subtle
- **Icons:** Lucide, concise usage
- **Transitions:** subtle, short, functional (`transition-colors`, `transition-all`)

New modules should **extend** this language, not replace it.

---

## 7. Feedback is Mandatory

Every user action must provide feedback:

| State | Implementation |
|-------|---------------|
| **Success** | `useToast()` with success message |
| **Error** | `useToast({ variant: 'destructive' })` with user-readable message |
| **Loading** | `Skeleton` components for async views |
| **Empty** | Intentional empty state with call-to-action (not blank space) |

```tsx
// Mutation feedback
toast({ title: 'Success', description: 'Item saved.' });
toast({ variant: 'destructive', title: 'Error', description: 'Save failed.' });
```

**FORBIDDEN:**
- `alert()` — always use toast
- Silent failures — always show error feedback
- Ad hoc floating message boxes

---

## 8. Accessibility (A11y) — Mandatory

| Requirement | Implementation |
|-------------|---------------|
| Interactive elements | Real `<button>` or `<a>` elements |
| Icon-only buttons | Must have `aria-label` |
| Form inputs | Must have `<Label htmlFor="...">` or `aria-label` |
| Keyboard navigation | Must work on all interactive elements |
| Focus states | Must remain visible |
| Non-semantic clickables | Must include `role`, `tabIndex`, keyboard handlers (rare, avoid when possible) |
| Color contrast | WCAG compliant ratios |

---

## 9. Responsive Design

- Desktop and tablet are **mandatory** — must work correctly
- Mobile is **recommended** — simplified but not broken
- Do not assume fixed widths
- App Bar and content containers must handle longer labels, empty data, smaller screens

---

## 10. Page Composition

**Do not build monolithic page components.**

Prefer this structure:
- **Page/layout components** in `ui/`
- **Orchestration hooks** in `hooks/`
- **API calls** in `services/`
- **Dialogs/panels/widgets** as dedicated components

### Standard page shell
```tsx
<div className="bg-slate-50 text-slate-900 font-sans flex flex-col h-full">
  <main className="flex-1 w-full max-w-7xl mx-auto px-6 py-12">
    {/* page content */}
  </main>
</div>
```

### Standard card assembly
```tsx
<Card className="border border-border shadow-sm">
  <CardHeader>
    <CardTitle>Title</CardTitle>
  </CardHeader>
  <CardContent className="space-y-4">
    {/* content */}
  </CardContent>
</Card>
```

---

## 11. Mobile/Touch Design Principles

When targeting touch interfaces:

| Principle | Guideline |
|-----------|-----------|
| Touch targets | Minimum 44x44pt (iOS) / 48x48dp (Android) |
| Spacing | Adequate spacing between interactive elements |
| Navigation | Tab bars for top-level, back buttons/gestures critical |
| Typography | Minimum 11pt/sp, clear hierarchy |
| Loading | Skeletons over spinners |
| Forms | Correct keyboard types, auto-fill support |
| Gestures | Intuitive (swipe to delete/back) |

---

## 12. Shared Shell Rules

- Authenticated screens live under `MainLayout`
- Do **not** build module-specific headers or toolbars — use the shared AppBar and SecondaryNavBar
- Module navigation and actions go through `ModuleMenuContext`
- Do not put complex local React state into the static module registry

---

## 13. Anti-Patterns

- Creating local headers that duplicate AppBar behavior
- Using raw `<input>`, `<button>` when Shadcn primitives exist
- Adding feature CSS files or inline hex colors
- Inventing a new color palette per module
- Mixing business logic and UI flow into one large page file
- Using `alert()` instead of the shared toast system
- Shipping interactive elements that are mouse-only
- Copying older inconsistent screens as style reference for new work
- Using raw HTML controls instead of Shadcn components

---

## 14. Review Checklist

Before considering UI work done:
- [ ] Uses the shared shell correctly
- [ ] Uses shared UI primitives where expected
- [ ] Uses theme tokens, not ad hoc colors
- [ ] Has loading, empty, success, and error feedback
- [ ] Is keyboard-usable
- [ ] Visually aligned with existing modules
- [ ] Does not introduce a new local visual language
- [ ] Responsive at desktop and tablet minimum
- [ ] Error Boundary wraps the module
- [ ] State management follows project conventions

---

## 15. State Management Strategy

### Decision tree

| State type | Where to manage | Tool |
|-----------|----------------|------|
| **Server state** (API data) | Feature hooks with cache | React Query / SWR / custom hook with `useEffect` |
| **UI state** (tabs, modals, filters) | Local component state | `useState`, `useReducer` |
| **Form state** | Form library | React Hook Form + Zod validation |
| **Auth state** | App-level context | `AuthContext` (single source of truth) |
| **Theme/layout** | App-level context | `ThemeContext` |
| **Cross-feature shared state** | Minimal, justified context | Dedicated context with ADR justification |

### Rules
- **Server state** is not UI state — do not store API responses in `useState`. Use a data-fetching layer with caching and invalidation
- **Minimize global state** — most state should be local to the feature
- **No prop drilling beyond 2 levels** — if passing props through 3+ components, extract to context or hook
- **Derived state is computed, not stored** — if a value can be computed from existing state, compute it
- **Auth state** lives in `AuthContext` only — never read `localStorage` directly in components

### Data fetching pattern

```tsx
// Feature hook encapsulates data fetching + state
export const useInvoices = (filters: InvoiceFilters) => {
    const api = useInvoicesApi();
    const [invoices, setInvoices] = useState<Invoice[]>([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);

    useEffect(() => {
        let cancelled = false;
        setLoading(true);
        api.listInvoices(filters)
            .then(data => { if (!cancelled) setInvoices(data); })
            .catch(err => { if (!cancelled) setError(err.message); })
            .finally(() => { if (!cancelled) setLoading(false); });
        return () => { cancelled = true; };
    }, [filters]);

    return { invoices, loading, error };
};
```

---

## 16. Error Boundaries (mandatory)

### Placement

```tsx
// App.tsx — every module wrapped in its own Error Boundary
<ErrorBoundary fallback={<ModuleErrorFallback />}>
    <Route path="/apps/invoices/*" element={<InvoicesPage />} />
</ErrorBoundary>
```

### Rules
- **App-level Error Boundary** — catches unhandled errors, shows full-page fallback
- **Module-level Error Boundary** — isolates module crashes, other modules keep working
- **Never** let a component crash propagate to the entire app
- Error fallback must offer a "retry" or "go back" action
- Log errors to structured logging (see `11-error-handling.md`)
