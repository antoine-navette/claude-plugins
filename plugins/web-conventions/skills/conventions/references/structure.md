# Structure

A React application built with Vite, talking only to its own origin: `/api` is proxied to the API by Vite in
development and by nginx in production. The priority is that the code reads easily, not that it anticipates needs that
don't exist yet.

## Folders

```
src/
├── config/       # http.ts: the HTTP client (openapi-fetch)
├── services/     # one function per API route: auth.ts, transaction.ts...
├── entities/     # the types pages work with: User.ts, Transaction.ts
├── mappers/      # API schema -> entity: UserMapper.ts
├── contexts/     # AuthContext.ts: the context and its useAuth() hook
├── providers/    # AuthProvider.tsx: loads the session, provides it
├── guards/       # RequireGuest, RequireAuth, RequireAdmin
├── pages/        # one per route: LoginPage.tsx, TransactionsPage.tsx, ErrorPage.tsx
├── modals/       # one per modal, opened from a page: UpsertTransactionModal.tsx
├── components/   # pieces several screens reuse: Modal (the frame every modal uses), Navbar
├── hooks/        # reusable technical logic: useGeolocation, useCountdown
├── constants/    # errors.ts: messages for every error code
├── App.tsx       # routes and guards
└── main.tsx      # router, providers, App
```

Folders are named after the role of what they hold, since in React almost everything is a component: a page, a modal,
a guard and a button are all components, and `components/` alone would say nothing about any of them.

Data flows in one short path: a page calls a service, which calls the HTTP client and maps the response into entities.
Pages never see `fetch`, the HTTP client nor the types generated from the API (see `api-access.md`).

## A page holds its own state

```tsx
const TransactionsPage = () => {
    const [state, setState] = useState<
        | { status: 'loading' }
        | { status: 'error'; code: ErrorCode }
        | { status: 'success'; transactions: Transaction[] }
    >({ status: 'loading' });

    useEffect(() => {
        let ignore = false;

        const load = async () => {
            const [transactions, code] = await getUserTransactions();
            // A response arriving after the page moved on (e.g. another month) is dropped
            if (ignore) return;
            if (code) return setState({ status: 'error', code });

            setState({ status: 'success', transactions });
        };

        void load();
        return () => {
            ignore = true;
        };
    }, []);

    if (state.status === 'loading') return <Spinner />;
    if (state.status === 'error') return <ErrorPage code={state.code} />;

    return (/* the page, top to bottom */);
};
```

- **The page holds its state in a `useState` with a `status`**, a union where each status carries exactly the data it
  has: no `isLoading` and `error` booleans that can contradict each other, no data that may be `undefined`. The union
  is written inline in the `useState`, not as a separate type used once.
- **It calls the services directly** and decides what to show, top to bottom, in one file. Understanding a screen means
  opening one file.
- **A page may be long.** A long page that reads top to bottom is better than ten files each used once.

## A modal is a screen too

A modal has its own form, its own submission with its loading and error, its own rules: it is a screen opened from a
page, not a piece of it. So it gets its own file in `modals/`, even when only one page opens it.

```tsx
// In the page: one state for every modal it can open, since only one is open at a time
const [modal, setModal] = useState<
    | { status: 'closed' }
    | { status: 'upsert'; transaction: Transaction | null }
    | { status: 'delete'; transaction: Transaction }
>({ status: 'closed' });

{modal.status === 'upsert' && (
    <UpsertTransactionModal
        transaction={modal.transaction}
        onClose={() => setModal({ status: 'closed' })}
        onUpsert={handleUpsert}
    />
)}
```

- **The page decides which modal is open**, with one union in its state, and passes the modal what it needs plus
  callbacks to report the result.
- **The modal holds its own state** (its form, its submission), in a `useState` with a `status` like a page.
- **It is rendered only while open**, so it mounts on each opening and its form starts from the values it receives: no
  state to reset by hand, no values left over from the previous opening.
- **Its frame is a component**: the backdrop, the box, the title and the close button are shared by every modal, in
  `components/Modal.tsx`.

## Components and hooks

- **A component is extracted only when it is used in several places.** Splitting a page into components used once
  (`AlertsPanel`, `ShopMap`) moves its logic away without making it reusable. This applies to pieces of a screen, not
  to screens: a page or a modal always has its own file.
- **Hooks are for reusable technical logic** (`useGeolocation`, `useCountdown`), never for a screen's state: a
  `useShopAlerts` hook moves the state out of the page, which then no longer reads on its own.
- **Some repetition is accepted** (WET, _Write Everything Twice_): two pages displaying something the same way each
  write their own, as long as the two uses aren't clearly identical. Factor out on the third copy, or when two copies
  must change together. An abstraction made too early freezes a resemblance that is often only on the surface, then
  grows options to cover the cases that diverge, and ends up harder to read than the copies it replaced.

## No global state library by default

The only data shared between screens is the signed-in user, held in a context (see `session-and-errors.md`). Each page
loads its own data. A global store (Redux, Zustand) or a server-state library (TanStack Query) becomes worth it when
several screens share and must keep in sync a lot of server data, which is a decision to make when it happens, not
upfront.

## Naming

- Pages end with `Page` (`LoginPage`), one per route, and modals with `Modal`, named after what they do
  (`UpsertTransactionModal`, `DeleteTransactionModal`). Components, providers, guards, entities and mappers are
  `PascalCase` files named after what they export. Services are named after the resource (`auth.ts`, `transaction.ts`),
  their functions after the API route (`login`, `getUserTransactions`).
- A context file exports both the context and its hook (`AuthContext.ts` exports `AuthContext` and `useAuth()`), which
  throws when used outside its provider.
