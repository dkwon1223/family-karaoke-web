# Family Karaoke 노래방 — Frontend

React + TypeScript (Vite) frontend for a Korean-style karaoke bar in Aurora, CO.
Deployed on Vercel. Consumes the Django REST API.

## Stack
- React + TypeScript (strict mode), Vite
- TanStack Router (file-based, type-safe routing)
- TanStack Query v5 (React Query) for server state
- Axios with JWT interceptors (`/src/lib/axios.ts`)
- **Tailwind CSS v3** for utility-first styling
- **shadcn/ui** for accessible, unstyled-by-default components (owned in codebase under `src/components/ui/`)
- GSAP (`useGSAP` hook) + React Three Fiber / Three.js r128 for 3D/animation
- Stripe: `@stripe/stripe-js` + `@stripe/react-stripe-js`
- Zod for runtime API response validation at service boundaries
- Vitest + React Testing Library for tests

## Project Structure
```
src/
  features/
    reservations/     # Booking flow, calendar, types.ts
    waitlist/         # QR entry, owner dashboard, types.ts
    payments/         # Stripe checkout, types.ts
    menu/             # Menu display, types.ts
    rooms/            # Room profiles, types.ts
    auth/             # Login, token management, types.ts
  routes/             # TanStack Router file-based routes (auto-generates routeTree.gen.ts)
  services/           # All Axios calls (reservationService.ts, etc.)
  hooks/              # Custom React Query hooks (useReservations, etc.)
  lib/
    axios.ts          # Axios instance, JWT interceptors, error normalization
    utils.ts          # cn() utility (clsx + tailwind-merge) — import from here everywhere
  components/
    ui/               # shadcn/ui generated components — never edit these manually after generation
  3d/                 # R3F scenes (lazy loaded, never imported on unrelated routes)
```

## Code Conventions

### TypeScript
- Strict mode — no `any` unless justified with a comment explaining why
- Types co-located with feature domain: `src/features/<domain>/types.ts`
- Use Stripe's published types from `@stripe/stripe-js` — never redefine them

### Styling
- **Tailwind CSS v3** — utility classes only; no custom CSS unless Tailwind genuinely can't express it
- **`cn()` is the only way to compose classNames** — always import from `@/lib/utils`, never concatenate strings manually
  ```typescript
  import { cn } from "@/lib/utils";
  // correct
  <div className={cn("base-class", isActive && "active-class", className)} />
  // never do this
  <div className={`base-class ${isActive ? "active-class" : ""}`} />
  ```
- **shadcn/ui** for all common UI primitives: Button, Dialog, Calendar, Select, Input, Form, Toast, Badge, Card, Table
  - Add components via `npx shadcn@latest add <component>` — they are copied into `src/components/ui/` and owned by this codebase
  - You may edit generated shadcn components to fit brand needs — they are not a black-box dependency
  - Never install a separate component library (MUI, Chakra, Ant Design) alongside shadcn
- **CSS variables drive theming** — all brand colors are defined as HSL variables in `src/index.css` under `:root` and `.dark`
  - To change the primary brand color, update `--primary` in `index.css` — do not hardcode color values in Tailwind classes
  - Dark mode is class-based (`darkMode: ["class"]` in `tailwind.config.js`)
- **shadcn Form + react-hook-form + Zod** is the standard pattern for all forms (reservation booking, waitlist entry, etc.)
  ```typescript
  const schema = z.object({ ... });
  const form = useForm({ resolver: zodResolver(schema) });
  ```
- **GSAP and Tailwind coexist fine** — GSAP manipulates inline styles/transforms directly, Tailwind handles static layout and color. Never use CSS-in-JS (styled-components, Emotion) — it conflicts with GSAP's DOM manipulation

### API Layer
- **Never** inline Axios calls in components — all calls go through `/src/services/`
- **Never** call `useQuery` directly in components with raw query functions — wrap in custom hooks in `/src/hooks/`
- Validate critical API responses (reservations, payments, waitlist) with Zod at the service boundary

### React Query
- Custom hooks: `useReservations()`, `useMenuItems()`, `useRoomAvailability()`, `useWaitlist()`
- Aggressive caching for static-ish data: `staleTime: 1000 * 60 * 10` for menu/room profiles
- Waitlist owner dashboard polls at `refetchInterval: 15000`

### Auth
- JWT access token stored in memory (NOT localStorage)
- Refresh token in `httpOnly` cookie
- Axios interceptor attaches access token and handles 401 → refresh → retry

### Stripe
- `VITE_STRIPE_PUBLISHABLE_KEY` is the only Stripe key on the frontend
- Never expose `STRIPE_SECRET_KEY` — backend-only
- Use Stripe's React components (`<Elements>`, `<PaymentElement>`) rather than custom card inputs

### Routing
- File-based routing via TanStack Router — routes live in `src/routes/`
- Route tree is auto-generated into `routeTree.gen.ts` — never edit this file manually
- Search params validated with Zod schemas defined in the route file using `validateSearch`
- Use route `loader` to prefetch into React Query cache; components still consume data via custom hooks
- Never store reservation filter state (dates, room, view mode) in `useState` — use search params so URLs are shareable and bookmarkable
- Authenticated routes protected via `beforeLoad` — redirect to `/login` if no valid token in context

### GSAP + R3F
- GSAP animations use `useGSAP()` hook — always use the returned cleanup to avoid memory leaks
- R3F scenes wrapped in `<Suspense>` — never block initial page paint
- Heavy 3D assets loaded via `useLoader` + `React.lazy` — code-split at route level
- **Never** mount `<Canvas>` on routes that don't need it

## Commands
```bash
npm run dev       # Dev server (localhost:5173) — also runs tsr watch in parallel
npm run build     # Runs tsr generate then vite build
npm run typecheck # tsc --noEmit
npm run test      # Vitest
npm run lint      # ESLint
```

## Environment Variables
See `.env.example`. Only `VITE_` prefixed vars are exposed to the browser.
Required: `VITE_API_BASE_URL`, `VITE_STRIPE_PUBLISHABLE_KEY`

## Testing Priorities
1. Reservation conflict/availability UI logic
2. Payment state transitions and Stripe redirect handling
3. Waitlist entry and queue state
4. Auth interceptor (token attach, refresh, logout on failure)

## Footguns
- Vercel previews get a new URL per deploy — update CORS on the backend for preview URLs or use a wildcard pattern in staging
- R3F `<Canvas>` is expensive — lazy load aggressively, gate behind route-level `React.lazy`
- GSAP ScrollTrigger requires `ScrollTrigger.refresh()` after layout changes — easy to miss after dynamic content loads
- `useGSAP` cleanup: if you don't return the cleanup, animations leak across route navigations
- TanStack Router requires `tsr generate` (or `tsr watch` in dev) to regenerate `routeTree.gen.ts` after adding/renaming route files — if route types look wrong, this is almost always why
- Never confirm payment on frontend redirect (`?payment_intent=...`) — wait for the reservation status to update via polling/refetch after the backend processes the webhook
- shadcn components are installed into `src/components/ui/` via CLI — after adding a new component, commit the generated file immediately so it's tracked in git
- Tailwind v3 is pinned intentionally — shadcn/ui is not yet fully compatible with Tailwind v4; do not upgrade without verifying shadcn compatibility first
- `cn()` from `@/lib/utils` uses `tailwind-merge` under the hood — this correctly resolves conflicting Tailwind classes (e.g. `cn("p-4", "p-8")` → `"p-8"`). Raw string concatenation does not do this and will produce specificity bugs
