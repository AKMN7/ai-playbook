# Feature workflow

How a feature is wired end-to-end: service → hook → component → form → table → loading/empty/error states → where state lives. Read this before adding a new CRUD feature or changing the data flow of an existing one.

**TanStack Query** is the only layer for server state. UI components never call Axios directly; they consume hooks under `src/hooks/<feature>/`, which wrap the functions in `src/services/<feature>/<Feature>.api.ts`.

---

## 1 — Layout of a feature

For a feature called `widgets`, expect:

```
src/services/widgets/
    Widgets.api.ts        # axios calls + Widget type + ENDPOINT const
    Widgets.schemas.ts    # WidgetSchema, WidgetSchemaType, WidgetSchemaInitData
    Widgets.helpers.ts    # (optional) pure transformers / exporters

src/hooks/widgets/
    useGetAllWidgets.ts   # list query
    useWidgetById.ts      # detail query
    useCreateWidget.ts    # create mutation (uses useIdempotentMutation when the endpoint supports it)
    useUpdateWidget.ts    # update mutation (plain useMutation)
    useDeleteWidget.ts    # delete mutation (plain useMutation)

src/components/widgets/
    CreateWidgetDialog.tsx
    UpdateWidgetDialog.tsx
    WidgetsTable.tsx
    WidgetsTableMain.tsx
    WidgetsTableColumnDefs.tsx
    WidgetsTableSearch.tsx
    WidgetsTableFilter.tsx
    WidgetsTableVisibility.tsx
    WidgetsTableExport.tsx
    WidgetsTablePagination.tsx
    WidgetsTableActionCell.tsx
    WidgetsTableDelete.tsx
    WidgetsTableResetSelection.tsx
```

If a feature does not need every file (e.g. no export, no filter, no update dialog), simply omit it. Don't create empty files. Access grants currently use `CreateAccessDialog`, `AccessCard`, and grant update/revoke controls inside each card rather than a separate `UpdateAccessDialog` or table action cell.

---

## 2 — `<Feature>.api.ts`

- Single `ENDPOINT` constant at the top, computed from environment variables (production vs. dev base URL):
    ```ts
    const ENDPOINT = import.meta.env.VITE_ENV === "PROD"
        ? `${import.meta.env.VITE_BE_ENDPOINT}/widget`
        : `${import.meta.env.VITE_BE_ENDPOINT}:3000/widget`;
    ```
- Export the entity TS `type` (`Widget`) — this is the API's response shape, **not** derived from Zod.
- Export each call as a `camelCase async function`. Return the parsed response (`API.GET<Widget[]>(...)`).
- Idempotent creates take an `idempotencyKey: string` parameter and forward it as the `"Idempotency-Key"` header. The hook layer (`useIdempotentMutation`) injects the key.
- Only use that idempotency shape when the backend endpoint supports it. Current idempotent resource creates include events, exclusives, feedback, participants, and workshops. Access grants and endpoints without an idempotency contract use plain `API.POST` and `useMutation`.
- File-uploading endpoints follow the two-step presigned-URL pattern: `API.POST` to get `{ uploadUrl, imgUrl }`, then `API.PUT` the file to the upload URL, then return the resulting `imgUrl` to the caller. Keep the upload helper as a non-exported function at the bottom of `<Feature>.api.ts`.

---

## 3 — `<Feature>.schemas.ts`

- Always export three things together:
    1. The Zod schema → `WidgetSchema` (PascalCase + `Schema`).
    2. The inferred TS type → `export type WidgetSchemaType = z.infer<typeof WidgetSchema>;`.
    3. The default form values → `export const WidgetSchemaInitData: WidgetSchemaType = { ... };`.
- All Zod errors are **i18n keys** in kebab-case strings — never literal English. Example: `z.string({ error: "name-is-required" }).trim().min(1, { error: "name-can-not-be-empty" })`.
- Use `z.email`, `z.iso.date`, `z.iso.time`, `z.discriminatedUnion`, `z.array(...).superRefine(...)` as needed.
- For dynamic Zod schemas built from form-field metadata, use `@/lib/ZodSchemaCreator` and `@/lib/ZodSchemaDefaultValues` if those helpers exist in the project — do not roll your own.

---

## 4 — Read flow (query hook)

```ts
// src/hooks/widgets/useGetAllWidgets.ts
import { getAllWidgets as getAllWidgetsAPI } from "@/services/widgets/Widgets.api";
import { keepPreviousData, useQuery } from "@tanstack/react-query";

export function useGetAllWidgets(parentId: string) {
    const { data: widgets, isPending: loading, isError } = useQuery({
        queryKey: ["widgets", parentId],
        queryFn: () => getAllWidgetsAPI(parentId),
        placeholderData: keepPreviousData,
        retry: false
    });

    return { widgets, loading, isError };
}
```

Rules:

- **Alias the imported API call** with an `API` suffix (`getAllWidgets as getAllWidgetsAPI`) to avoid colliding with the hook's internal binding name.
- **Rename `data`** to the resource (`widgets`, `widget`, `users`) and **rename `isPending`** to `loading`. Always also expose `isError`.
- `queryKey` is an array starting with the kebab-case resource name, followed by any scoping ids.
- Default options on the read side: `placeholderData: keepPreviousData` for lists, `retry: false` everywhere. Global `QueryClient` defaults (typically `staleTime: 3min`, `refetchInterval: 5min`) live in `main.tsx`.
- For lists that should display newest-first regardless of API order, use `select: (data) => data?.reverse()`.

---

## 5 — Create flow (idempotent mutation when supported)

```ts
// src/hooks/widgets/useCreateWidget.ts
import { useTranslation } from "react-i18next";
import { toast } from "sonner";

import { useIdempotentMutation } from "@/hooks/useIdempotentMutation";
import { createWidget as createWidgetAPI } from "@/services/widgets/Widgets.api";
import { useQueryClient } from "@tanstack/react-query";

export function useCreateWidget(successCallBack?: () => void) {
    const queryClient = useQueryClient();
    const { t } = useTranslation();

    const { mutate: createWidget, isPending: loading, isSuccess: success, isError, reset } = useIdempotentMutation({
        mutationFn: (payload: object, idempotencyKey) => createWidgetAPI(payload, idempotencyKey),
        onSuccess: (response) => {
            toast.success(t(response || "widget-created-successfully"), { duration: 5000 });
            queryClient.invalidateQueries({ queryKey: ["widgets"] });
            successCallBack?.();
        },
        onError: (error) => {
            toast.error(t(error.message || "something-went-wrong-please-try-again-later"));
        },
        onSettled: () => {
            reset();
        }
    });

    return { createWidget, loading, success, isError, reset };
}
```

Rules:

- Creates use **`useIdempotentMutation`** (from `@/hooks/useIdempotentMutation`) when the service function accepts an idempotency key and forwards `Idempotency-Key`.
- Creates without a backend idempotency contract, including `useGrantAccess`, use plain `useMutation` while keeping the standard `onSuccess` / `onError` / `onSettled` lifecycle.
- Updates and deletes use plain `useMutation` from `@tanstack/react-query`.
- The hook accepts an optional `successCallBack?: () => void` (camelCase, exactly that spelling — kept for consistency).
- Hook return shape: `{ <verb><Resource>, loading, success?, isError, reset }`. Aliases:
    - `mutate` → action name (`createWidget`, `updateWidget`, `deleteWidget`).
    - `isPending` → `loading`.
    - `isSuccess` → `success` (only when needed; otherwise omit).

---

## 6 — Update flow

```ts
// src/hooks/widgets/useUpdateWidget.ts
const { mutate: updateWidget, isPending: loading, isSuccess: success, isError, reset } = useMutation({
    mutationFn: ({ widgetId, payload }: { widgetId: string; payload: object }) => updateWidgetAPI(widgetId, payload),
    onSuccess: (response) => {
        toast.success(t(response || "widget-updated-successfully"), { duration: 5000 });
        queryClient.invalidateQueries({ queryKey: ["widgets"] });
        queryClient.invalidateQueries({ queryKey: ["widget"] });
        successCallBack?.();
    },
    onError: (error) => toast.error(t(error.message || "something-went-wrong-please-try-again-later")),
    onSettled: () => reset()
});
```

- Pass mutation arguments as a **single object** (`{ widgetId, payload }`), never positional.
- Invalidate **every related query key**: list, detail, and any aggregated stats key.

---

## 7 — Delete flow

```ts
const { mutate: deleteWidgets, isPending: loading, isError, isSuccess, reset } = useMutation({
    mutationFn: ({ parentId, widgetIds }: { parentId: string; widgetIds: string[] }) =>
        deleteWidgetsAPI(parentId, widgetIds)
    /* onSuccess / onError / onSettled per the standard pattern */
});
```

- Bulk-delete endpoints accept `{ targets: ids }` in the request body and the hook signature uses `<resource>Ids: string[]`.
- Single-entity deletes pass the id directly (`deleteWidget(parentId, widgetId)`).

---

## 8 — Submission from a component

```tsx
function CreateWidgetDialog({ onClose }: CreateWidgetDialogProps) {
    const { createWidget, loading } = useCreateWidget(() => onClose());
    const form = useForm<WidgetSchemaType>({ resolver: zodResolver(WidgetSchema), defaultValues: WidgetSchemaInitData });

    function onSubmit(values: WidgetSchemaType) {
        createWidget(values);
    }

    function resetState() {
        form.reset(WidgetSchemaInitData);
    }

    return (
        <Dialog defaultOpen onOpenChange={onClose}>
            <DialogContent onCloseAutoFocus={resetState}>
                <form onSubmit={form.handleSubmit(onSubmit)}>...</form>
            </DialogContent>
        </Dialog>
    );
}
```

- Dialogs reset form state on `onCloseAutoFocus`, not on `onOpenChange`.
- Update dialogs hydrate the form with `useEffect(() => { if (data) form.reset(data, { keepDefaultValues: false }); }, [data, form])`.
- The submit button is **disabled when** `loading || !form.formState.isValid` (creates) and additionally `|| !form.formState.isDirty` (updates).
- See `§9 — Forms` below for the full Field/FieldGroup/FieldError pattern.

---

## 9 — Forms (React Hook Form + Zod + shadcn `Field` primitives)

Every form follows the same skeleton. Mirror it literally.

### 9.1 — The schema-first contract

For each form, the corresponding schema file (`src/services/<feature>/<Feature>.schemas.ts`) **must** export three things:

```ts
export const WidgetSchema = z.object({ /* ... */ });
export type WidgetSchemaType = z.infer<typeof WidgetSchema>;
export const WidgetSchemaInitData: WidgetSchemaType = { /* ... */ };
```

Form components import all three. Don't re-derive defaults inline; use `WidgetSchemaInitData`.

### 9.2 — `useForm` setup

```tsx
const form = useForm<WidgetSchemaType>({
    resolver: zodResolver(WidgetSchema),
    defaultValues: WidgetSchemaInitData
});
```

- `shouldUnregister: true` is added when the form contains conditionally-rendered fields (e.g. fields hidden by tab/switch state, or by a discriminated-union type field).
- For multi-step or modal-driven flows, hydrate on data load:
    ```tsx
    useEffect(() => {
        if (data) form.reset(data, { keepDefaultValues: false });
    }, [data, form]);
    ```

### 9.3 — Field rendering: always `<Controller>` + shadcn `Field` primitives

Never use `register` directly. Every field is a `<Controller>` rendering shadcn's `Field*` primitives:

```tsx
<Controller
    name="email"
    control={form.control}
    render={({ field, fieldState }) => (
        <Field data-invalid={fieldState.invalid}>
            <FieldLabel htmlFor="widget-email">{t("email")}</FieldLabel>
            <InputGroup>
                <InputGroupAddon>
                    <Mail />
                </InputGroupAddon>
                <InputGroupInput
                    id="widget-email"
                    aria-invalid={fieldState.invalid}
                    placeholder={t("placeholder-email-example")}
                    disabled={loading}
                    {...field}
                />
            </InputGroup>
            <FieldDescription>{t("widget-email-description")}</FieldDescription>
            {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
        </Field>
    )}
/>
```

Recurring conventions:

- `name` matches the schema path (dotted paths for nested objects: `name="location.url"`, `name="target.flag"`).
- `id` on the input is `<feature>-<field>` (kebab-case): `widget-email`, `widget-name`, `widget-identification`.
- `aria-invalid={fieldState.invalid}` and `data-invalid={fieldState.invalid}` are both set — the first on the input, the second on the wrapping `<Field>`.
- Every field has a `FieldLabel` + `FieldDescription` (translated) + an `{fieldState.invalid && <FieldError errors={[fieldState.error]} />}` line. `FieldError` accepts an array because Zod can produce multiple issues.
- `disabled={loading}` is applied to every interactive control while a mutation is pending.

When the underlying input is uncontrolled or inside a popover/select, **omit** the spread-`...field` and instead wire `value` / `onValueChange` (or `onChange`) explicitly. This applies to `Select`, `DropdownMenu + Command`, `Popover + Calendar`, `Switch`, etc.

### 9.4 — Layout primitives

- The form body is wrapped in `<FieldGroup>`. Fieldsets use `<FieldSet>` + `<FieldLegend>` + `<FieldDescription>`.
- For toggle-style fields, use `orientation="horizontal"` on `<Field>` and place a `<FieldContent>` (label/description) next to the control.
- For input + description on the same row use `orientation="responsive"`.

### 9.5 — Submit handling

```tsx
function onSubmit(values: WidgetSchemaType) {
    const finalized = prepareWidgetData(values); // optional, defined in <Feature>.helpers.ts
    createWidget(finalized);                     // returned by useCreateWidget()
}

<form onSubmit={form.handleSubmit(onSubmit)}>...</form>;
```

- The submit button is **at the end of the `<FieldGroup>`**, full-width by default, with `type="submit"` and:
    - Create form: `disabled={loading || !form.formState.isValid}`
    - Update form: `disabled={loading || !form.formState.isValid || !form.formState.isDirty}`
- The button label is `t("submit")` (or a feature-specific equivalent like `t("contact-send")`).

### 9.6 — Reset on dialog close

Dialog-hosted forms reset on `onCloseAutoFocus`, not `onOpenChange`:

```tsx
function resetState() {
    form.reset(WidgetSchemaInitData);
}

<DialogContent onCloseAutoFocus={resetState}>
```

Update dialogs do not call `resetState` because the next data load re-hydrates the form via the `useEffect` shown above.

### 9.7 — Dynamic Zod schemas (when forms are metadata-driven)

When a form is built dynamically from a `Record<string, FormInputSchemaType>` (e.g. user-defined custom fields):

- Use `ZodSchemaCreator(formFields)` to build the schema.
- Use `ZodSchemaDefaultValues(formFields)` to build defaults.
- Render fields with the project's `FormFieldsDynamicField` component from `src/components/form-fields/`.
- Extend the dynamic schema with `.extend({ ... })` only for known additional fields (e.g. `picture`).

Skip this section entirely when the project has no dynamic-form feature — don't introduce the abstraction speculatively.

### 9.8 — File / image fields

- Use the project's shared image picker (commonly `<ProfilePic image={...} setImage={...} />`) from `src/components/general/`.
- The Zod field is a `z.custom` accepting `File`, `string` (existing URL), or `undefined`:
    ```ts
    picture: z.custom(
        (value) => value === undefined || value instanceof File || typeof value === "string",
        { error: "invalid-image" }
    );
    ```
- The API layer uploads `File` instances to a presigned URL inside the `.api.ts` file — the form just hands over the `File`.

### 9.9 — Forms — don't do

- Don't use `register` instead of `<Controller>`. Every field uses `<Controller>`.
- Don't use plain `<label>`, `<p>`, or `<span>` for field labels/descriptions/errors. Use the shadcn `Field*` primitives.
- Don't mix `react-hook-form`'s `<Form>` provider components from shadcn — use `<Controller>` directly.
- Don't translate validation messages inline; the message is already a translation key, and the toast / `<FieldError>` rendering is responsible for `t()`-wrapping when needed.
- Don't reset the form on `onOpenChange` — that fires before the close animation and causes flicker. Use `onCloseAutoFocus`.

---

## 10 — Mutation lifecycle (toast feedback)

Every mutation hook MUST supply `onSuccess`, `onError`, and `onSettled`:

```ts
onSuccess: (response) => {
    toast.success(t(response || "<feature-success-i18n-key>"), { duration: 5000 });   // duration optional
    queryClient.invalidateQueries({ queryKey: ["..."] });                              // every related key
    successCallBack?.();                                                               // close dialog / redirect
},
onError: (error) => {
    toast.error(t(error.message || "something-went-wrong-please-try-again-later"));
},
onSettled: () => {
    reset();                                                                           // mutation.reset, not form
}
```

Notes:

- Server messages already come through `error.message` (normalized in `API.ts`) and are themselves i18n keys — that's why `t(error.message)` works.
- `something-went-wrong-please-try-again-later` is the canonical fallback. Don't invent a new fallback key.
- Add new success-key strings to **both** `en.json` and `ar.json` in the same change.

---

## 11 — Route-level data fetching pattern

Pages consume hooks and short-circuit in this order:

```tsx
const { widget, loading: loadingWidget } = useWidgetById(scopeId, widgetId);
const { stats, loading: loadingStats } = useGetWidgetStats(scopeId);

if (!scopeId) return <NoScopeSelected />;
if (loadingWidget || loadingStats) return <WidgetsSkeleton />;
if (isError) return <GenericError />;
```

- Empty / no-selection states first.
- Loading second — render a feature-specific skeleton from `components/skeleton/`.
- Error third — render an empty-state from `components/empty-states/`.
- Then the real content.

When forking a hook's `loading` for two simultaneous requests, alias each (`loading: loadingWidget`, `loading: loadingStats`).

---

## 12 — Data tables and table-powered collections (TanStack Table)

Every list view that needs filtering, sorting, selection, or export uses **`@tanstack/react-table`**. Literal data tables follow the full table decomposition. Card/grid collections can use TanStack Table headlessly for filtering and row modeling while rendering cards instead of `<Table>` primitives.

### 12.1 — File decomposition per table

For a feature `widgets`, expect under `src/components/widgets/`:

```
WidgetsTable.tsx                # composer: wires hooks, table instance, header row, body, pagination
WidgetsTableMain.tsx            # the <Table>/<TableHeader>/<TableBody> primitive wrapper
WidgetsTableColumnDefs.tsx      # ColumnDef<...>[] + co-located translated header / cell mini-components
WidgetsTableActionCell.tsx      # row-actions menu (view, edit, delete, etc.)
WidgetsTableSearch.tsx          # search input bound to a column filter
WidgetsTableFilter.tsx          # filter dropdown for one or more columns
WidgetsTableVisibility.tsx      # column visibility toggle
WidgetsTableExport.tsx          # CSV / JSON export
WidgetsTablePagination.tsx      # page-size + paging controls
WidgetsTableDelete.tsx          # bulk-delete (only mounted when rows are selected)
WidgetsTableResetSelection.tsx  # clear-selection button
```

If a feature doesn't need every file (no export, no filter, no bulk-delete), **omit those files**. Don't create empty stubs.

Card-backed collections may add a `<Feature>Card.tsx` and omit table-only pieces such as action cells, visibility, pagination, selection, delete, reset, and export controls when the UI does not expose those capabilities. Access grants currently use `AccessTable`, `AccessTableMain`, `AccessTableColumnDefs`, `AccessTableSearch`, and `AccessCard`.

### 12.2 — `WidgetsTable.tsx` skeleton

```tsx
function WidgetsTable({ scopeId }: WidgetsTableProps) {
    const { widgets, loading } = useGetAllWidgets(scopeId);

    const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);
    const [columnVisibility, setColumnVisibility] = useState<VisibilityState>({});
    const [rowSelection, setRowSelection] = useState<RowSelectionState>({});

    const columns = useMemo(() => {
        if (!widgets || !widgets.length) return [];
        return WidgetsTableColumnDefs(scopeId);
    }, [widgets, scopeId]);

    const table = useReactTable({
        data: widgets || [],
        columns,
        getCoreRowModel: getCoreRowModel(),
        getPaginationRowModel: getPaginationRowModel(),
        getFilteredRowModel: getFilteredRowModel(),
        onColumnFiltersChange: setColumnFilters,
        onColumnVisibilityChange: setColumnVisibility,
        onRowSelectionChange: setRowSelection,
        initialState: { pagination: { pageIndex: 0, pageSize: 20 } },
        state: { columnFilters, columnVisibility, rowSelection },
        autoResetPageIndex: false,
        getRowId: (row) => row._id as string
    });

    if (loading) return <TableSkeleton />;

    const isRowSelected = Object.keys(rowSelection).length > 0;

    return (
        <div className="space-y-4">
            <div className="flex flex-col gap-4 md:flex-row md:justify-between">
                <WidgetsTableSearch table={table} />
                <ButtonGroup className="flex w-full *:flex-1 md:w-auto">
                    <WidgetsTableFilter table={table} />
                    <WidgetsTableVisibility table={table} />
                    <WidgetsTableExport table={table} />
                    {isRowSelected && (
                        <Fragment>
                            <WidgetsTableDelete scopeId={scopeId} table={table} />
                            <WidgetsTableResetSelection table={table} />
                        </Fragment>
                    )}
                    <CreateWidgetDialog scopeId={scopeId} />
                </ButtonGroup>
            </div>
            <div className="overflow-hidden rounded-md border">
                <WidgetsTableMain table={table} />
            </div>
            <WidgetsTablePagination table={table} />
        </div>
    );
}
```

Hard rules:

- `pageSize: 20` is the standard default.
- `autoResetPageIndex: false` is the standard — preserve page on filter changes.
- `getRowId` must use the entity's stable id field, e.g. `row._id as string` for database entities or `row.clerkId` for access grants.
- The bulk-action buttons (`Delete`, `ResetSelection`) are mounted **only when** `rowSelection` is non-empty, wrapped in a `<Fragment>`.
- The `<Create...>` dialog/button is the **last** child of the `<ButtonGroup>`.

For headless card/grid collections:

- Keep `columnFilters` and the row models needed for the UX; omit `columnVisibility`, `rowSelection`, and pagination state when there are no controls for them.
- `ColumnDef[]` can be accessor-only when columns exist only for searching/filtering.
- Render `table.getFilteredRowModel().rows` into cards in `<Feature>TableMain`.
- Use a card/grid skeleton that matches the real layout instead of `<TableSkeleton />`.
- Empty state can be a centered feature-specific key such as `t("no-access-found")`.

### 12.3 — Column definitions (`WidgetsTableColumnDefs.tsx`)

- Export a function that returns the `ColumnDef[]` so id-scoped values (scope ids, parent ids) close over correctly.
- The first column is a `select` checkbox column (`enableSorting: false`, `enableHiding: false`) when the feature uses row selection.
- The last column is the action cell, appended via `columns.push(FormatActionsHeaderAndCell(...))`, when row actions render inside a table.
- For translated headers, define a small in-file component (e.g. `TranslatedHeader`) instead of calling `useTranslation()` inside the cell render — `header: () => <TranslatedHeader header="name" />`.
- Cell renderers are kept short. Wrap inline content in `<Badge>`, `<Avatar>`, or simple `<div>` with utility classes (`w-max max-w-56 truncate ...`).

### 12.4 — Empty body

`WidgetsTableMain.tsx` must render a `t("no-results")` row when `table.getRowModel().rows?.length === 0`:

```tsx
{
    isEmpty && (
        <TableRow>
            <TableCell colSpan={table.getAllColumns().length} className="h-24 text-center">
                {t("no-results")}
            </TableCell>
        </TableRow>
    );
}
```

For card/grid collection mains, render a centered empty message with the feature key that matches the UI, e.g. `t("no-access-found")`.

### 12.5 — CSV / JSON export

- Live in `<Feature>.helpers.ts`, three functions: `Structure<Feature>ToExport(rows)`, `<Feature>CSVExporter(headers, data)`, `<Feature>JSONExporter(data)`. Names are PascalCase by precedent.
- The CSV exporter uses **PapaParse** with a UTF-8 BOM (`"﻿"`) prepended — required for non-ASCII columns (Arabic, Chinese, accented Latin, etc.) to render correctly in Excel:
    ```ts
    const blob = new Blob(["﻿" + csv], { type: "text/csv;charset=utf-8;" });
    ```
- Filenames use `${feature}_${Date.now()}.csv` / `.json`.
- The export button in the table is a `<DropdownMenu>` with `csv` / `json` items.

### 12.6 — Bulk delete

- The bulk-delete component takes `{ scopeId, ..., table }` props.
- It calls `useDelete<Feature>` with ids from the entity's stable id field, e.g. `<feature>Ids: table.getFilteredSelectedRowModel().flatRows.map((r) => r.original._id)` for database entities.
- After success it should reset selection: `table.toggleAllRowsSelected(false)` (or simply rely on cache invalidation; mirror local precedent).

### 12.7 — Tables — don't do

- Don't compose literal data tables outside this decomposition for the sake of brevity. Even a small table follows the pattern, while card/grid collections use the headless subset described above.
- Don't wire pagination with custom hooks when `getPaginationRowModel()` already covers it.
- Don't bypass loading skeletons. Use `<TableSkeleton />` for literal table views and a matching feature/card skeleton for card or grid collections.
- Don't add server-side filtering/pagination unless the dataset is large enough to require it. Default to client-side.

---

## 13 — Loading, error, and empty states

Every feature must render the same three-state UX. Components live in dedicated folders and use the shadcn primitives — never custom-built fallbacks.

### 13.1 — Render order at the route / section level

Pages and large sections short-circuit in this exact order:

```tsx
if (!scopeId) return <NoScopeSelected />;             // 1. empty / no-selection state
if (loadingX || loadingY) return <FeatureSkeleton />; // 2. loading state
if (isError) return <GenericError />;                 // 3. error state
return ( /* real content */ );                        // 4. happy path
```

Don't combine these into ternaries inside JSX. Use early returns.

### 13.2 — Empty / no-data states (`src/components/empty-states/`)

- One `PascalCase.tsx` file per state. File naming: `No<Thing>Found.tsx`, `No<Thing>Selected.tsx`, `GenericError.tsx`, `NotFound.tsx`, `ErrorComponent.tsx`.
- Built on the shadcn `<Empty>` family of primitives:
    ```tsx
    <Empty>
        <EmptyHeader>
            <EmptyMedia variant="icon">
                <Icon />
            </EmptyMedia>
            <EmptyTitle>{t("no-<thing>-selected")}</EmptyTitle>
            <EmptyDescription>{t("no-<thing>-selected-description")}</EmptyDescription>
        </EmptyHeader>
        <EmptyContent>
            <EmptyDescription>
                {t("need-help-q")} <Link to="/">{t("contact-support")}</Link>
            </EmptyDescription>
        </EmptyContent>
    </Empty>
    ```
- Page-level empty states wrap the `<Empty>` in a centered flex container: `<div className="flex flex-1 items-center justify-center">...</div>`.
- The error/destructive variant tints the icon: `<EmptyMedia className="bg-destructive/20 text-destructive" variant="icon">`.
- Translation keys for empty/error copy follow the conventions in `conventions.md §4` (`no-<thing>-found`, `no-<thing>-selected`, `something-went-wrong`, `unexpected-error-description`, `need-help-q`, `contact-support`).

### 13.3 — Error fallbacks

- `ErrorComponent.tsx` is the **router-level default** (`defaultErrorComponent` in `main.tsx`) — full-screen, with a "Reload" `<Button>` calling `window.location.reload()`.
- `GenericError.tsx` is the **section-level** error used inside route components when a hook returns `isError` — same primitives, no full-screen wrapper.
- Don't show raw error messages from `error.message` in empty-states — these are i18n keys, but they're already routed to the toast layer by the mutation hooks. The empty-state shows generic copy and a "contact support" link.

### 13.4 — Loading states (`src/components/skeleton/`)

- One `PascalCase.tsx` skeleton per page or section, named `<Feature>Skeleton.tsx` (plus a generic `TableSkeleton.tsx`, `PlaceholderSkeleton.tsx` etc).
- Built from the shadcn `<Skeleton />` primitive arranged to **match the real layout** (headings, stat grids, tables) so swapping in real content doesn't shift the page.
- Use `<TableSkeleton />` inside literal table views while `loading` is true. For `useReactTable`-powered card/grid collections, use a feature skeleton or local skeleton layout that matches the real cards.
- For inline mid-content loading (e.g. card action pending or button submitting), use the existing shadcn `<Spinner />` from `@/components/ui/spinner` or the established local button spinner pattern.

### 13.5 — Spinners

- `<SpinnerPage />` from `src/components/general/` is the full-page spinner used by `_app.tsx`, `_auth.tsx`, and `_base.tsx` while Clerk's `useAuth().isLoaded` is false.
- For other "tiny synchronous loading" cases, prefer a skeleton over a spinner unless the surrounding action-cell/card pattern already uses `<Spinner />`.

### 13.6 — Toasts (sonner)

- Mounted once at root via `<Toaster richColors />` inside `__root.tsx`. Don't mount additional `Toaster` instances.
- Used **only** inside mutation `onSuccess` / `onError` handlers; not inside route components, not inside queries.
- Always pass an i18n key wrapped in `t()`:
    - Success: `toast.success(t(response || "<feature-success-key>"), { duration: 5000 })`.
    - Error: `toast.error(t(error.message || "something-went-wrong-please-try-again-later"))`.

### 13.7 — Loading/error/empty — don't do

- Don't roll a custom `<Spinner />` or `<Empty />` — use the existing primitives.
- Don't put loading/error logic inside the components rendered by happy-path content — the route gate handles it.
- Don't render different skeletons for "first load" vs "re-fetch". `placeholderData: keepPreviousData` already keeps last data on screen during refetches.
- Don't show toasts on read errors (`useQuery`); only on mutation errors. Read errors render `<GenericError />` instead.

---

## 14 — State management

There are exactly three places state lives in this stack. Pick the right one for what you're storing.

### 14.1 — Server state → TanStack Query

All data that comes from the backend is owned by **TanStack Query**. Never mirror it into local state.

- Reads: `useQuery` inside a hook under `src/hooks/<feature>/use<Resource>...ts`. Components consume the hook.
- Writes: `useMutation` inside a hook in the same folder. Use `useIdempotentMutation` for creates only when the service function and backend endpoint support `Idempotency-Key`. See `§5 — Create flow` above.
- Defaults are set globally in `src/main.tsx`:
    ```ts
    new QueryClient({
        defaultOptions: {
            queries: {
                staleTime: 3 * 60 * 1000,
                refetchInterval: 5 * 60 * 1000
            }
        }
    });
    ```
    **Don't override these per-hook unless you have a specific reason** (e.g. an inherently real-time chart). Per-hook overrides used in this stack: `placeholderData: keepPreviousData` for lists, `retry: false` everywhere, `select: (data) => data?.reverse()` for reverse-chronological lists.
- Query keys live in arrays starting with the kebab-case resource (`["widgets"]`, `["widget-children", parentId, childId]`). Mutations invalidate **every** related key — list, detail, and any aggregated stats key.

### 14.2 — Cross-feature client state → React contexts

Use a **context** in `src/contexts/<thing>-provider.tsx` only when state needs to be shared across multiple unrelated features. Typical cases:

- A theme provider for light/dark mode (persisted to `localStorage`).
- A "currently selected scope" provider (e.g. organization id, workspace id, project id) — typically with toggle-off semantics: re-selecting the same id clears it.

Provider conventions:

- Default-state-throwing hook (`if (context === undefined) throw new Error("Context usage prohibited in the current placement")`).
- Persistence to `localStorage` happens **inside** the provider, not the consumer.
- Two named exports per file: `<Thing>Provider` and `use<Thing>` (or `useSelected<Thing>` etc).

Don't reach for a global state library (Redux, Zustand, Jotai). The patterns above cover everything.

### 14.3 — Component-local state → `useState` / `useReducer`

- Anything UI-only (open/closed, current tab, table column visibility, selection, filter strings) lives in the component that owns it.
- For TanStack Table state, manage `columnFilters`, `columnVisibility`, and `rowSelection` with `useState` and pass them into `useReactTable({ state: { ... } })`. See `§12 — Data tables`.
- Use `useMemo` to derive expensive structures (column definitions, lookup maps). Only depend on values that actually change.

### 14.4 — Where NOT to put state

- **Don't store derived state**. Compute it during render or with `useMemo`.
- **Don't mirror Query data** into `useState`. If you need to transform the data, use `select` on the query, or compute it during render.
- **Don't put feature-local state into a context.** A `Dialog`'s open/close state belongs in the component that owns the trigger.
- **Don't hand-roll fetch logic with `useEffect` + `setState`.** Always go through `useQuery` / `useMutation`.

### 14.5 — `localStorage` usage

- Keys typically used in this stack: a theme key (e.g. `vite-ui-theme`), a scope-selection key, plus `i18nextLng` (managed by `i18next-browser-languagedetector`).
- When persisting a new piece of state, prefer hanging it off a context provider (the established pattern). Don't sprinkle direct `localStorage` access across components.
- Never store auth tokens, sensitive API responses, or anything that could grow unbounded.

### 14.6 — Optimistic updates

- This stack does **not** use optimistic mutations by default. Every mutation invalidates and refetches on success.
- Don't introduce optimistic updates without explicit approval — they couple component code to mutation hooks and complicate rollback.
