# 20. Mutations: Server Actions and Route Handlers

## 20.1 Server Actions (Server Functions)

**What it is:** An `async` function marked `'use server'` that runs **on the server** but can be called from a form or a button in the browser. Next.js creates the HTTP endpoint for you. No API route, no `fetch`, fully typed.

**Why it matters:** writing data (create a sale, receive stock, update a price) becomes one function. Forms even work before JavaScript loads (progressive enhancement).

```ts
// app/dashboard/stock/actions.ts
'use server';

import { z } from 'zod';
import { updateTag } from 'next/cache';
import { redirect } from 'next/navigation';
import { batchSchema } from '@/schemas/batch';
import { requireRole } from '@/lib/auth';

export type FormState = {
  ok: boolean;
  message?: string;
  errors?: Partial<Record<keyof z.infer<typeof batchSchema>, string[]>>;
};

export async function receiveStock(prevState: FormState, formData: FormData): Promise<FormState> {
  // 1. AUTHORISE — a Server Action is a public endpoint; always check who is calling
  const user = await requireRole(['pharmacist', 'admin']);

  // 2. VALIDATE — never trust formData
  const parsed = batchSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { ok: false, errors: z.flattenError(parsed.error).fieldErrors };
  }

  // 3. MUTATE
  try {
    await db.batch.create({ data: { ...parsed.data, receivedById: user.id } });
  } catch (e) {
    if (isUniqueViolation(e)) return { ok: false, errors: { batchNo: ['Batch number already exists'] } };
    return { ok: false, message: 'Could not save the batch. Try again.' };
  }

  // 4. REVALIDATE cached data that changed
  updateTag('medicines');
  updateTag(`stock-${parsed.data.medicineId}`);

  return { ok: true, message: 'Stock received.' };
  // or: redirect('/dashboard/stock');   (redirect throws, so call it outside try/catch)
}
```

## 20.2 Using a Server Action in a form

```tsx
// app/dashboard/stock/receive-form.tsx
'use client';
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';
import { receiveStock, type FormState } from './actions';

const initialState: FormState = { ok: false };

export function ReceiveStockForm({ medicines }: { medicines: Medicine[] }) {
  const [state, formAction, isPending] = useActionState(receiveStock, initialState);

  return (
    <form action={formAction} className="grid max-w-md gap-4">
      <select name="medicineId" required>
        {medicines.map((m) => <option key={m.id} value={m.id}>{m.name}</option>)}
      </select>
      <FieldError errors={state.errors?.medicineId} />

      <input name="batchNo" placeholder="AMX-2026-01" required />
      <FieldError errors={state.errors?.batchNo} />

      <input name="quantity" type="number" min={1} required />
      <FieldError errors={state.errors?.quantity} />

      <input name="costPrice" type="number" min={0} required />
      <input name="expiresAt" type="date" required />
      <FieldError errors={state.errors?.expiresAt} />

      {state.message && (
        <p role="status" className={state.ok ? 'text-green-700' : 'text-red-600'}>{state.message}</p>
      )}
      <SubmitButton />
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();     // must be INSIDE the <form>
  return <button disabled={pending}>{pending ? 'Saving…' : 'Receive stock'}</button>;
}

function FieldError({ errors }: { errors?: string[] }) {
  return errors?.length ? <p className="text-sm text-red-600">{errors[0]}</p> : null;
}
```

**Without a form** — call the action like a normal async function from an event handler, often inside `startTransition`:

```tsx
'use client';
import { useTransition } from 'react';
import { voidSale } from './actions';

export function VoidSaleButton({ saleId }: { saleId: number }) {
  const [isPending, startTransition] = useTransition();
  return (
    <button disabled={isPending} onClick={() => startTransition(async () => { await voidSale(saleId); })}>
      {isPending ? 'Voiding…' : 'Void'}
    </button>
  );
}
```

**Passing extra arguments** with `bind`: `<form action={updatePrice.bind(null, medicine.id)}>` — the action receives `(id, formData)`.

## 20.3 Optimistic UI

**What it is:** Show the expected result immediately, before the server confirms; React rolls back automatically if the action fails.

```tsx
'use client';
import { useOptimistic } from 'react';

export function StockAdjuster({ medicineId, stock }: { medicineId: number; stock: number }) {
  const [optimisticStock, addOptimistic] = useOptimistic(stock, (current, delta: number) => current + delta);

  async function adjust(delta: number) {
    addOptimistic(delta);                 // UI updates instantly
    await adjustStock(medicineId, delta); // server action; real value arrives after revalidation
  }

  return (
    <form action={async () => adjust(-1)}>
      <span>{optimisticStock} units</span>
      <button>Remove 1 (damaged)</button>
    </form>
  );
}
```

## 20.4 Server Action security checklist

- Every Server Action is a **public HTTP endpoint**. Check authentication and authorisation inside **each** action, not only in the page.
- Validate every input with Zod.
- Never trust IDs or prices sent from the client: re-read the price from the database when creating a sale.
- Return safe error messages; log details on the server.
- Next.js protects actions against CSRF (POST only, origin checks) and gives them unguessable IDs, but that does not replace auth checks.

## 20.5 Route Handlers (API endpoints)

**What it is:** A `route.ts` file inside `app/` creates an HTTP endpoint using the standard Web `Request`/`Response` objects. Use it for things that are not form submissions from your own UI: webhooks (M-Pesa callbacks), a mobile app, third-party integrations, file downloads.

```ts
// app/api/medicines/route.ts → GET/POST /api/medicines
import { NextRequest } from 'next/server';

export async function GET(request: NextRequest) {
  const search = request.nextUrl.searchParams.get('search') ?? '';
  const medicines = await db.medicine.findMany({
    where: { name: { contains: search, mode: 'insensitive' } },
    take: 20,
  });
  return Response.json({ data: medicines });
}

export async function POST(request: NextRequest) {
  const user = await getApiUser(request);                       // e.g. verify a Bearer token
  if (!user) return Response.json({ message: 'Unauthenticated' }, { status: 401 });

  const parsed = NewMedicineSchema.safeParse(await request.json());
  if (!parsed.success) return Response.json({ errors: z.flattenError(parsed.error).fieldErrors }, { status: 422 });

  const medicine = await db.medicine.create({ data: parsed.data });
  return Response.json({ data: medicine }, { status: 201 });
}
```

```ts
// app/api/medicines/[id]/route.ts → /api/medicines/7
export async function GET(_req: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const medicine = await db.medicine.findUnique({ where: { id: Number(id) } });
  return medicine ? Response.json({ data: medicine }) : Response.json({ message: 'Not found' }, { status: 404 });
}
```

```ts
// app/api/mpesa/callback/route.ts — payment webhook
export async function POST(request: Request) {
  const body = await request.json();
  const { CheckoutRequestID, ResultCode } = body.Body.stkCallback;
  await db.payment.update({
    where: { checkoutRequestId: CheckoutRequestID },
    data: { status: ResultCode === 0 ? 'paid' : 'failed' },
  });
  return Response.json({ ResultCode: 0, ResultDesc: 'Accepted' });   // always acknowledge quickly
}
```

**Supported methods:** `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`.

## 20.6 Headers and cookies

```ts
import { cookies, headers } from 'next/headers';

// Read (Server Components, Actions, Route Handlers)
const cookieStore = await cookies();
const branch = cookieStore.get('branch')?.value ?? 'dodoma';
const userAgent = (await headers()).get('user-agent');

// Write (Server Actions and Route Handlers only)
cookieStore.set('branch', 'dar', { httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: 60 * 60 * 24 * 30 });
cookieStore.delete('branch');

// Set response headers in a Route Handler
return new Response(csv, { headers: { 'Content-Type': 'text/csv', 'Content-Disposition': 'attachment; filename="sales.csv"' } });
```

Security headers for every page go in `next.config.ts` (`async headers()`) or in `proxy.ts` (section 26).

## 20.7 Server Action vs Route Handler: which one?

| Situation | Use |
| --- | --- |
| A form or button in your own Next.js UI changes data | **Server Action** |
| A mobile app, Postman or another server calls you | **Route Handler** |
| A webhook (M-Pesa, Stripe, Laravel) | **Route Handler** |
| Reading data for a page | **Server Component** (neither) |
| Streaming a file download or CSV export | **Route Handler** |
