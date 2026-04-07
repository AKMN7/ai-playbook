# Feature workflow guide

This document outlines the **canonical flow** for building features in interactive apps.
Stick to it, and always stay consistent.

**TanStack Query** is the standard layer for all server state: use **`useQuery`** for reads and **`useMutation`** for writes. Do not call Axios directly from UI components—go through hooks that wrap the API functions in `*.api.ts`.

## 1 — Screen flow pattern

Interactive apps are **screen-flow** driven. Define the visitor journey as a sequence of screens.

| Step                    | Responsibility                                                                                                                | Required files / libs                                  |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| **1. Screens**          | Each distinct step is a component with a `*Screen.tsx` suffix.                                                                | `src/components/<Feature>/<Step>Screen.tsx`            |
| **2. Navigation**       | Screens receive navigation callbacks (`onNext`, `onBack`, `onReset`) — no direct router coupling.                             | component props                                        |
| **3. Schema**           | Define all Zod objects alongside their default values for any form or data the screen handles.                                | `src/services/<Feature>/<feature>.schemas.ts`          |
| **4. Component**        | Build a form UI with **React Hook Form** and `zodResolver(schema)` when user input is needed.                                 | `src/components/<Feature>/<feature-operation>Form.tsx` |
| **5. Mutation Hook**    | Implement `use<Verb><Feature>()` using _TanStack Query_'s `useMutation`.                                                      | `src/hooks/<feature>/useCreate<Feature>.ts`            |
| **6. API Call**         | Network logic lives in `<feature>.api.ts` (`createFeature`, `updateFeature`).                                                 | `src/services/<Feature>/<feature>.api.ts`              |
| **7. Submit**           | On form `handleSubmit`, call the dedicated mutation hook.                                                                     | component code                                         |
| **8. Success Handling** | In the mutation `onSuccess`: `queryClient.invalidateQueries` for relevant caches, display toast, navigate to the next screen. | mutation hook                                          |

## 2 — Read (query flow)

| Step              | Responsibility                                                    | Required files / libs |
| ----------------- | ----------------------------------------------------------------- | --------------------- |
| **1. API Call**   | Add fetcher (`getFeature`, `listFeatures`) in `<feature>.api.ts`. |                       |
| **2. Query Hook** | Wrap the fetcher with `useQuery` in `src/hooks/<feature>/`.       |                       |
| **3. Component**  | Consume the query hook; rely on its `data`, `isLoading`, `error`. | UI component          |

_Queries must_ declare a stable `queryKey` that matches the invalidation key used by mutations.

## 3 — Delete (mutation without form)

| Step                    | Responsibility                                          | Required files / libs |
| ----------------------- | ------------------------------------------------------- | --------------------- |
| **1. API Call**         | `deleteFeature(id)` in `<feature>.api.ts`.              |                       |
| **2. Mutation Hook**    | `useDelete<Feature>()` using `useMutation`.             |                       |
| **3. Invocation**       | Call the mutation directly from UI (button, menu item). |                       |
| **4. Success Handling** | `invalidateQueries` the list and/or details cache.      |                       |

## 4 — Toast-based feedback & mutation lifecycle

- **Library:** [`sonner`](https://www.npmjs.com/package/sonner) for non-blocking toasts.
- **Pattern:** supply `onSuccess`, `onError`, and `onSettled` in every mutation.

```ts
const mutation = useCreateVote({
    onSuccess: () => {
        toast.success("Vote submitted");
        invalidateVotes();
        onNext();
    },
    onError: (err) => {
        toast.error(err.message ?? "Something went wrong");
    },
    onSettled: () => mutation.reset()
});
```
