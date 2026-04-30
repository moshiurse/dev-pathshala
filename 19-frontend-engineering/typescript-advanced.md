# 🦾 TypeScript অ্যাডভান্সড (TypeScript Advanced)

## ভূমিকা

TypeScript জুনিয়র লেভেলে = "JS + types"। সিনিয়র লেভেলে = "compile-time program যা runtime program-এর contract enforce করে"। জেনেরিক, conditional types, mapped types, template literal types, `infer`, branded types — এগুলো শুধু "ফ্যান্সি ফিচার" নয়, এগুলো বড় codebase-এ runtime bug প্রতিরোধের অস্ত্র। API response মিস-typed → production crash; ID confusion (userId বনাম orderId দুইটাই string) → wrong record update; এই type-level discipline ছাড়া বড় ফ্রন্টএন্ড টিকবে না।

```
   TypeScript-এর দুই জগৎ
   ────────────────────
        runtime (JS)              compile time (TS only)
        ─────────────             ──────────────────────
        values, functions   ←→    types, interfaces
        let, const                type, interface
        if/else                   conditional types
        for                       mapped types
        +, -, ===                 &, |, extends
```

---

## 🧬 Generics — Type-level functions

```typescript
// সরল generic
function identity<T>(value: T): T { return value; }
const x = identity(42);          // T inferred as number
const s = identity('hello');     // T inferred as string

// Generic constraint — T কে object-like হতে হবে
function pluck<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const out = {} as Pick<T, K>;
  for (const k of keys) out[k] = obj[k];
  return out;
}

const user = { id: 1, name: 'Karim', email: 'k@x.com', password: 'secret' };
const safe = pluck(user, ['id', 'name']);
// safe: { id: number; name: string }
```

### Generic React component

```tsx
type SelectProps<T> = {
  options: T[];
  getLabel: (item: T) => string;
  getValue: (item: T) => string;
  value: T | null;
  onChange: (value: T) => void;
};

function Select<T>({ options, getLabel, getValue, value, onChange }: SelectProps<T>) {
  return (
    <select
      value={value ? getValue(value) : ''}
      onChange={(e) => {
        const found = options.find((o) => getValue(o) === e.target.value);
        if (found) onChange(found);
      }}
    >
      {options.map((opt) => (
        <option key={getValue(opt)} value={getValue(opt)}>
          {getLabel(opt)}
        </option>
      ))}
    </select>
  );
}

// ব্যবহার
<Select
  options={cities}                 // City[]
  getLabel={(c) => c.bn}
  getValue={(c) => c.id}
  value={selected}
  onChange={setSelected}
/>
```

`<T,>` (trailing comma) JSX-এ generic বোঝাতে দরকার।

---

## ❓ Conditional Types

```typescript
type IsString<T> = T extends string ? true : false;
type A = IsString<'hi'>;     // true
type B = IsString<42>;       // false

// Distributive over union
type ToArray<T> = T extends any ? T[] : never;
type C = ToArray<string | number>;   // string[] | number[]
```

### বাস্তব use: API response unwrap

```typescript
type ApiResponse<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; message: string };

type DataOf<R> = R extends { status: 'success'; data: infer D } ? D : never;

type UserResp = ApiResponse<{ id: number; name: string }>;
type User = DataOf<UserResp>;   // { id: number; name: string }
```

---

## 🗺️ Mapped Types

```typescript
// Object-এর প্রতি key-কে transform করা
type Optional<T> = { [K in keyof T]?: T[K] };
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };

// সব value-কে string বানাও
type Stringify<T> = { [K in keyof T]: string };

// Filter: শুধু string property গুলো
type StringKeys<T> = { [K in keyof T]: T[K] extends string ? K : never }[keyof T];

interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}
type SK = StringKeys<User>;   // 'name' | 'email'
```

### Key remapping (TS 4.1+)

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<{ name: string; age: number }>;
// {
//   getName: () => string;
//   getAge:  () => number;
// }
```

---

## 🔤 Template Literal Types

```typescript
type Lang = 'bn' | 'en';
type Page = 'home' | 'product' | 'cart';
type Route = `/${Lang}/${Page}`;
// '/bn/home' | '/bn/product' | '/bn/cart' | '/en/home' | ...

// Event name strict
type EventName<T extends string> = `on${Capitalize<T>}`;
type Click = EventName<'click'>;   // 'onClick'

// Path utility
type CssVar<K extends string> = `--${K}`;
type C = CssVar<'brand-primary'>;   // '--brand-primary'
```

### বাস্তব: Type-safe nested keys

```typescript
type Path<T, Prefix extends string = ''> = {
  [K in keyof T]: T[K] extends object
    ? `${Prefix}${string & K}` | Path<T[K], `${Prefix}${string & K}.`>
    : `${Prefix}${string & K}`;
}[keyof T];

const config = {
  brand: { primary: '#f57', secondary: '#0f1' },
  font: { size: { sm: 12, md: 14 } },
};

type ConfigPath = Path<typeof config>;
// 'brand' | 'brand.primary' | 'brand.secondary' | 'font' | 'font.size' | 'font.size.sm' | 'font.size.md'
```

---

## 🔍 `infer` keyword

Conditional type-এর ভেতরে type extract।

```typescript
// Function-এর return type বের করা
type ReturnType<F> = F extends (...args: any[]) => infer R ? R : never;
type R = ReturnType<() => string>;   // string

// Promise unwrap
type Awaited<P> = P extends Promise<infer T> ? T : P;
type A = Awaited<Promise<number>>;   // number

// First element of tuple
type First<T extends any[]> = T extends [infer F, ...any[]] ? F : never;
type F = First<[string, number, boolean]>;   // string

// Function-এর first parameter
type FirstParam<F> = F extends (a: infer A, ...rest: any[]) => any ? A : never;
```

---

## 🛠️ Built-in Utility Types

| Utility            | কী করে                                     |
|--------------------|--------------------------------------------|
| `Partial<T>`       | সব property optional                      |
| `Required<T>`      | সব property required                      |
| `Readonly<T>`      | সব property readonly                      |
| `Pick<T, K>`       | K keys-গুলো রাখো                            |
| `Omit<T, K>`       | K keys-গুলো বাদ দাও                        |
| `Record<K, V>`     | `{ [k in K]: V }` শর্টকাট                   |
| `Exclude<T, U>`    | T থেকে U বাদ                              |
| `Extract<T, U>`    | T থেকে U নাও                              |
| `NonNullable<T>`   | `null` ও `undefined` বাদ                   |
| `ReturnType<F>`    | function return type                       |
| `Parameters<F>`    | tuple of params                            |
| `Awaited<T>`       | Promise unwrap                             |
| `InstanceType<C>`  | class instance type                        |
| `ConstructorParameters<C>` | constructor params tuple           |

```typescript
interface Product {
  id: string;
  name: string;
  price: number;
  description?: string;
}

type ProductInput   = Omit<Product, 'id'>;          // create-এর সময়
type ProductUpdate  = Partial<Omit<Product, 'id'>>; // update-এ সব optional
type ProductSummary = Pick<Product, 'id' | 'name' | 'price'>;
type PriceMap       = Record<string, number>;       // { [productId]: price }
```

---

## 🎭 Discriminated Unions

Tag field দিয়ে union narrow করা — TS-এর সবচেয়ে শক্তিশালী প্যাটার্ন।

```typescript
type LoadingState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: Product[] }
  | { status: 'error'; error: string };

function render(state: LoadingState) {
  switch (state.status) {
    case 'idle':    return 'অপেক্ষা করুন';
    case 'loading': return 'লোড হচ্ছে...';
    case 'success': return state.data.map(/* ... */);   // .data accessible
    case 'error':   return state.error;                 // .error accessible
    default: {
      const _exhaustive: never = state;   // exhaustiveness check!
      throw new Error(_exhaustive);
    }
  }
}
```

`never` দিয়ে exhaustiveness — নতুন variant যোগ করলে compile error।

### Real example: Form action

```typescript
type FormAction =
  | { type: 'set_field'; field: string; value: unknown }
  | { type: 'reset' }
  | { type: 'submit_start' }
  | { type: 'submit_success'; result: SubmitResult }
  | { type: 'submit_error'; error: ApiError };

function reducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'set_field': return { ...state, values: { ...state.values, [action.field]: action.value } };
    case 'reset':     return initialState;
    case 'submit_start':   return { ...state, submitting: true };
    case 'submit_success': return { ...state, submitting: false, result: action.result };
    case 'submit_error':   return { ...state, submitting: false, error: action.error };
  }
}
```

---

## 🛂 Type Guards & Narrowing

```typescript
// 1) typeof guard
function double(x: string | number) {
  if (typeof x === 'number') return x * 2;       // x: number
  return x.repeat(2);                            // x: string
}

// 2) instanceof
if (err instanceof ApiError) { /* err.statusCode */ }

// 3) `in` operator
type Bird = { fly: () => void; name: string };
type Fish = { swim: () => void; name: string };
function move(animal: Bird | Fish) {
  if ('fly' in animal) animal.fly();
  else animal.swim();
}

// 4) Custom type predicate (`is`)
function isProduct(x: unknown): x is Product {
  return typeof x === 'object' && x !== null
    && 'id' in x && typeof (x as any).id === 'string'
    && 'price' in x && typeof (x as any).price === 'number';
}

const data: unknown = await fetch('/p/1').then(r => r.json());
if (isProduct(data)) {
  console.log(data.price);   // safe
}

// 5) assertion function (`asserts`)
function assertDefined<T>(v: T | undefined, msg = 'Expected defined'): asserts v is T {
  if (v === undefined) throw new Error(msg);
}

const user = users.find(u => u.id === id);
assertDefined(user, 'User not found');
console.log(user.name);   // user: User (narrowed from User | undefined)
```

### Schema-based runtime validation: **Zod**

```typescript
import { z } from 'zod';

const ProductSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  price: z.number().positive(),
  description: z.string().optional(),
  createdAt: z.coerce.date(),
});

type Product = z.infer<typeof ProductSchema>;   // single source of truth!

// Runtime parse + compile-time type
const data: unknown = await fetch('/api/p/1').then(r => r.json());
const product = ProductSchema.parse(data);     // throws if invalid
//    ^ Product
```

বড় টিমে এটা game changer — API contract drift ধরা পড়ে।

---

## 🏷️ Branded Types — Nominal typing

TypeScript structural typing ব্যবহার করে — দুই string আলাদা না। কিন্তু `userId` ও `orderId` দুইটাই string হলে confusion হয়। **Branded types** দিয়ে nominal typing simulate।

```typescript
type Brand<K, T> = K & { readonly __brand: T };

type UserId  = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;
type Money   = Brand<number, 'Money'>;   // taka

function createUserId(s: string): UserId {
  if (!/^u_[a-z0-9]+$/.test(s)) throw new Error('Invalid');
  return s as UserId;
}

function getUser(id: UserId) { /* ... */ }
function cancelOrder(id: OrderId) { /* ... */ }

const u = createUserId('u_abc');
const o = 'o_xyz' as OrderId;

getUser(u);          // ✅
getUser(o);          // ❌ Type error: OrderId not assignable to UserId
getUser('random');   // ❌ string not assignable to UserId
```

bKash-এ taka unit confusion (paisa vs taka mix-up) — branded `Paisa` ও `Taka` type দিয়ে compile-time এ ধরা যায়।

---

## 🔒 Const Assertions

```typescript
// Without `as const`
const config = { theme: 'dark', size: 'lg' };
// type: { theme: string; size: string }

// With `as const`
const config2 = { theme: 'dark', size: 'lg' } as const;
// type: { readonly theme: 'dark'; readonly size: 'lg' }

// Tuple
const point = [10, 20] as const;
// type: readonly [10, 20]

// Enum-like from object
const ROUTES = {
  HOME: '/',
  CART: '/cart',
  PROFILE: '/profile',
} as const;

type Route = typeof ROUTES[keyof typeof ROUTES];
// '/' | '/cart' | '/profile'
```

---

## 🔧 Module Augmentation

Third-party type extend করা।

```typescript
// styled-components theme augmentation
import 'styled-components';

declare module 'styled-components' {
  export interface DefaultTheme {
    colors: {
      brand: string;
      text: string;
      surface: string;
    };
    spacing: (n: number) => string;
  }
}

// এখন: const Btn = styled.button`color: ${(p) => p.theme.colors.brand};`  ✅ typed
```

```typescript
// Window-এ global property
declare global {
  interface Window {
    bkash: {
      create: (config: BkashConfig) => BkashInstance;
    };
    dataLayer: Array<Record<string, unknown>>;
  }
}

window.dataLayer.push({ event: 'purchase', value: 999 });   // ✅
```

---

## 🛡️ tsconfig Strict Options

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",

    /* Strict flags — সব on করুন */
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "useUnknownInCatchVariables": true,
    "alwaysStrict": true,

    /* Additional safety */
    "noUncheckedIndexedAccess": true,    // arr[0] -> T | undefined
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,

    /* Other */
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,             // Vite/esbuild compat
    "verbatimModuleSyntax": true,
    "resolveJsonModule": true
  }
}
```

**`noUncheckedIndexedAccess`** দিলে:
```typescript
const arr = [1, 2, 3];
const x = arr[0];   // type: number | undefined  (আগে ছিল number)
if (x !== undefined) console.log(x.toFixed(2));
```

বাস্তব bug ধরে — production crash কম।

---

## 🌐 Real Example: API Response Typing (Daraz product API)

```typescript
import { z } from 'zod';

// 1) Schema
const ProductSchema = z.object({
  id: z.string(),
  sku: z.string(),
  title: z.object({ bn: z.string(), en: z.string() }),
  price: z.object({
    original: z.number(),
    discounted: z.number().nullable(),
    currency: z.literal('BDT'),
  }),
  rating: z.object({
    average: z.number().min(0).max(5),
    count: z.number().int().nonnegative(),
  }),
  stock: z.discriminatedUnion('status', [
    z.object({ status: z.literal('in_stock'), quantity: z.number() }),
    z.object({ status: z.literal('low_stock'), quantity: z.number() }),
    z.object({ status: z.literal('out_of_stock'), restockDate: z.string().optional() }),
  ]),
  images: z.array(z.string().url()).min(1),
  attributes: z.record(z.string(), z.string()).optional(),
});

const ProductListResponseSchema = z.object({
  items: z.array(ProductSchema),
  page: z.number(),
  pageSize: z.number(),
  total: z.number(),
});

// 2) Inferred types
export type Product = z.infer<typeof ProductSchema>;
export type ProductListResponse = z.infer<typeof ProductListResponseSchema>;

// 3) Type-safe fetcher
async function fetchProducts(category: string, page = 1): Promise<ProductListResponse> {
  const res = await fetch(`/api/products?cat=${category}&page=${page}`);
  if (!res.ok) throw new ApiError(res.status, await res.text());
  const json = await res.json();
  return ProductListResponseSchema.parse(json);   // runtime validate + typed
}

// 4) Discriminated stock narrowing in component
function StockBadge({ stock }: { stock: Product['stock'] }) {
  switch (stock.status) {
    case 'in_stock':
      return <span className="badge-green">স্টকে আছে ({stock.quantity})</span>;
    case 'low_stock':
      return <span className="badge-yellow">মাত্র {stock.quantity}টি বাকি!</span>;
    case 'out_of_stock':
      return <span className="badge-red">
        স্টকে নেই {stock.restockDate ? `(${stock.restockDate})` : ''}
      </span>;
  }
}
```

---

## 🇧🇩 বাংলাদেশী কনটেক্সট

- **bKash**: `Paisa` ও `Taka` branded types — paisa-taka mix bug রোধ
- **Pathao**: `RideId` ও `DriverId` ও `UserId` সব আলাদা branded
- **Daraz**: i18n key strict typing — `t('cart.title')` typo compile error
- **Foodpanda**: phone number `BdMobile = Brand<string, 'BdMobile'>` যা শুধু `01[3-9]\d{8}` regex pass করে validate হয়
- **NID**: `NidNumber` branded — ১০/১৭ digit validate

---

## ⚠️ সাধারণ Pitfall

| ভুল                                  | পরিণতি                          | সমাধান                       |
|---------------------------------------|----------------------------------|------------------------------|
| `any` ব্যবহার                         | Type safety গায়েব               | `unknown` + narrowing        |
| `as Type` cast (assertion)            | Runtime mismatch                | Zod / type predicate         |
| `Function` type                       | Too loose                        | নির্দিষ্ট signature           |
| `Object` / `{}`                       | মানে "non-null" — confusing     | `Record<string, unknown>`    |
| Optional + default mix                | Confusing API                   | একটা bechun                  |
| Enum (numeric)                        | Bundle bloat, weird              | `as const` object বা string union |
| Deep generic stack                    | Slow compile                    | Helper alias দিয়ে ভাঙুন      |
| `noUncheckedIndexedAccess` off        | `arr[i]` undefined miss         | Strict mode on               |
| `// @ts-ignore`                       | Hidden bug                       | `@ts-expect-error` (auto-removes when fixed) |

---

## ✅ চেকলিস্ট

- [ ] `tsconfig.json` strict mode সব flag on
- [ ] `noUncheckedIndexedAccess` enabled
- [ ] কোনো `any` বা `as` cast নেই (justified ছাড়া)
- [ ] API response Zod/io-ts দিয়ে validate
- [ ] Discriminated union exhaustiveness check (`never`)
- [ ] Branded types: `UserId`, `OrderId`, `Money` ইত্যাদি
- [ ] Generic component-এ proper constraint
- [ ] Utility types (`Pick`, `Omit`, `Partial`) সঠিক জায়গায়
- [ ] Module augmentation third-party-র জন্য
- [ ] `Awaited<T>`, `ReturnType<F>` দিয়ে DRY

---

## 📝 সারসংক্ষেপ

Advanced TypeScript = compile-time safety net যা runtime bug কমায়। **Generics** type-level reusability দেয়; **conditional + mapped + template literal types** complex transformation; **`infer`** types unwrap; **utility types** (Partial, Pick, Omit, Record, ReturnType) প্রতিদিনের কাজ; **discriminated unions + exhaustiveness check** state machine modeling-এর গোল্ড স্ট্যান্ডার্ড; **type guards** + `unknown` narrow করা; **branded types** nominal typing; **const assertions** literal lock; **module augmentation** library extend; **strict tsconfig** ছাড়া বড় কোডবেস টিকবে না। API boundary-তে **Zod** runtime + compile time এক করে — সবচেয়ে বড় win বাস্তব প্রজেক্টে।

> 💡 মনে রাখুন: **"`any` মানে surrender, `unknown` মানে discipline। বড় কোডবেসে discipline চাই।"**
