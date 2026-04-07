# Feature workflow guide

This document outlines the **canonical flow** for Create, Read, Update, and Delete operations for any feature.  
Stick to it, and always stay consistent.

**TanStack Query** is the standard layer for all server state: use **`useQuery`** for reads and **`useMutation`** for writes. Do not call Axios directly from UI components—go through hooks that wrap the API functions in `*.api.ts`.

## 1 — Create & Update (schema-first flow)

| Step                    | Responsibility                                                                                                                | Required files / libs                                      |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **1. Schema**           | Define all Zod objects along side their default values.                                                                       | `src/services/<Feature>/<feature>.schemas.ts`              |
| **2. Component**        | Build a form UI with **React Hook Form** and `zodResolver(schema)`.                                                           | `src/components/<Feature>/<feature-operation><Form>.tsx`   |
| **3. Mutation Hook**    | Implement `use<Verb><Feature>()` using _TanStack Query_’s `useMutation`.                                                      | `src/hooks/<feature>/useCreate<Feature>.ts` & `useUpdate…` |
| **4. API Call**         | Network logic lives in `<feature>.api.ts` (`createFeature`, `updateFeature`).                                                 | `src/services/<Feature>/<feature>.api.ts`                  |
| **5. Submit**           | On form `handleSubmit`, call the dedicated mutation hook.                                                                     | component code                                             |
| **6. Success Handling** | In the mutation `onSuccess`:<br>• `queryClient.invalidateQueries` for relevant lists / details<br>• Display toast / redirect. | mutation hook                                              |

## 2 — Read (query flow)

| Step              | Responsibility                                                    | Required files / libs |
| ----------------- | ----------------------------------------------------------------- | --------------------- |
| **1. API Call**   | Add fetcher (`getFeature`, `listFeatures`) in `<feature>.api.ts`. |
| **2. Query Hook** | Wrap the fetcher with `useQuery` or in `src/hooks/<feature>/`.    |
| **3. Component**  | Consume the query hook; rely on its `data`, `isLoading`, `error`. | UI component          |

_Queries must_ declare a stable `queryKey` that matches the invalidation key used by create/update/delete mutations.

## 3 — Delete (mutation without form)

| Step                    | Responsibility                                          | Required files / libs |
| ----------------------- | ------------------------------------------------------- | --------------------- |
| **1. API Call**         | `deleteFeature(id)` in `<feature>.api.ts`.              |
| **2. Mutation Hook**    | `useDelete<Feature>()` using `useMutation`.             |
| **3. Invocation**       | Call the mutation directly from UI (button, menu item). |
| **4. Success Handling** | `invalidateQueries` the list and/or details cache.      |

## 5 — Toast-based feedback & mutation lifecycle

- **Library:** [`sonner`](https://www.npmjs.com/package/sonner) for non-blocking toasts.
- **Pattern:** supply `onSuccess`, `onError`, and `onSettled` in every mutation.

```ts
const mutation = useCreateUser({
    onSuccess: () => {
        toast.success("User created");
        invalidateUsers();
    },
    onError: (err) => {
        toast.error(err.message ?? "Something went wrong");
    },
    onSettled: () => mutation.reset()
});
```
