# 🗂️ স্টেট ম্যানেজমেন্ট (State Management)

## ভূমিকা

"State" মানে যেকোনো ডেটা যা সময়ের সাথে পরিবর্তিত হয় এবং UI-তে প্রতিফলিত হতে হবে। ফ্রন্টএন্ডে state ম্যানেজমেন্টের জটিলতা এই সাধারণ প্রশ্ন থেকে আসে: "এই ডেটা কোথায় রাখব?" — কম্পোনেন্টে? Context-এ? Redux store-এ? URL-এ? Server-এ? প্রতিটির trade-off আছে। ভুল জায়গায় state রাখলে: prop drilling, অপ্রয়োজনীয় re-render, stale data, race conditions, আর debugging-এ দিন কেটে যাবে।

```
   প্রশ্ন: "এই state কোথায় থাকবে?"
   ────────────────────────────────
   শুধু এক component-এ?      → useState (local)
   পেজের URL-এ থাকা উচিত?    → URL state (router)
   একাধিক component শেয়ার?   → Context / Zustand
   সার্ভার থেকে আসা ডেটা?     → React Query / SWR
   ফর্ম ইনপুট?               → React Hook Form / Formik
   পুরো অ্যাপ-ব্যাপী?         → Redux / Zustand / Jotai
```

---

## 📊 স্টেটের ক্যাটেগরি — চারটি মূল প্রকার

```
┌──────────────────────────────────────────────────────────────┐
│                    State Categories                          │
├─────────────┬────────────────────────────────────────────────┤
│  Server     │ API থেকে আসা ডেটা: products, user profile,    │
│  State      │ orders। Caching, refetch, stale time লাগে।    │
│             │ → React Query, SWR, Apollo, RTK Query          │
├─────────────┼────────────────────────────────────────────────┤
│  Client     │ UI-only state: modal open, theme, sidebar,    │
│  State      │ selected tab। Server জানে না, ক্লায়েন্টে থাকে। │
│             │ → useState, Zustand, Jotai, Redux              │
├─────────────┼────────────────────────────────────────────────┤
│  URL        │ ?page=2&filter=mobile&sort=price.              │
│  State      │ Shareable, bookmark-able, browser back support │
│             │ → React Router, Next.js useSearchParams        │
├─────────────┼────────────────────────────────────────────────┤
│  Form       │ Input values, validation errors, dirty state, │
│  State      │ submit status।                                 │
│             │ → React Hook Form, Formik, native HTML form    │
└─────────────┴────────────────────────────────────────────────┘
```

**মূলনীতি**: প্রতিটি state-কে যথাসম্ভব **"নিচের" স্তরে** রাখুন। Modal open/close যদি শুধু এক component-এ লাগে, তাহলে Redux-এ পাঠানোর প্রয়োজন নেই।

---

## 🏛️ Patterns — Flux থেকে Atom

### ১. Flux (Facebook, 2014)

Flux একটা **আর্কিটেকচারাল প্যাটার্ন**, library নয়। মূল ধারণা: **unidirectional data flow**।

```
   ┌──────┐  Action   ┌────────────┐  Update  ┌───────┐  Notify  ┌──────┐
   │ View ├──────────▶│ Dispatcher ├─────────▶│ Store ├─────────▶│ View │
   └──────┘           └────────────┘          └───────┘          └──────┘
          ◀──────────────────────────── re-render ──────────────────────
```

কেন ভালো: ডেটা সবসময় এক দিকে যায় → debugging সহজ, predictable।

### ২. Redux (Flux-এর evolved form)

Redux একটি single store, pure reducer functions, এবং immutable updates ব্যবহার করে।

```javascript
// Redux Toolkit (RTK) দিয়ে আধুনিক উদাহরণ
import { createSlice, configureStore } from '@reduxjs/toolkit';

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [], total: 0 },
  reducers: {
    addItem(state, action) {
      // RTK Immer ব্যবহার করে → mutation lookalike, কিন্তু immutable
      state.items.push(action.payload);
      state.total += action.payload.price;
    },
    removeItem(state, action) {
      const idx = state.items.findIndex(i => i.id === action.payload);
      if (idx >= 0) {
        state.total -= state.items[idx].price;
        state.items.splice(idx, 1);
      }
    },
  },
});

export const { addItem, removeItem } = cartSlice.actions;
export const store = configureStore({
  reducer: { cart: cartSlice.reducer },
});
```

```javascript
// React-এ ব্যবহার
import { useSelector, useDispatch } from 'react-redux';

function CartIcon() {
  const count = useSelector((s) => s.cart.items.length);
  const dispatch = useDispatch();
  return <button onClick={() => dispatch(addItem({ id: 1, price: 500 }))}>
    🛒 {count}
  </button>;
}
```

**Redux কখন বাছবেন:**
- বড় টিম, predictable patterns দরকার
- DevTools (time-travel debugging) দরকার
- অনেক complex business logic একই jay shared

### ৩. Zustand — minimal, hook-based

```javascript
import { create } from 'zustand';

const useCart = create((set, get) => ({
  items: [],
  addItem: (item) => set((s) => ({ items: [...s.items, item] })),
  removeItem: (id) => set((s) => ({
    items: s.items.filter((i) => i.id !== id),
  })),
  total: () => get().items.reduce((sum, i) => sum + i.price, 0),
}));

// component-এ
function CartIcon() {
  const count = useCart((s) => s.items.length);
  return <span>🛒 {count}</span>;
}
```

কোনো Provider নেই, boilerplate নেই — তবু global state। Selector-based subscription → শুধু count বদলালে এই component re-render হয়।

### ৪. Jotai — atom-based

```javascript
import { atom, useAtom } from 'jotai';

const itemsAtom = atom([]);
const totalAtom = atom((get) =>
  get(itemsAtom).reduce((s, i) => s + i.price, 0)
);

function Cart() {
  const [items, setItems] = useAtom(itemsAtom);
  const [total] = useAtom(totalAtom);   // derived, auto-update
  return <div>Total: ৳{total}</div>;
}
```

প্রতিটি atom independent — fine-grained reactivity, Recoil-এর সরল উত্তরসূরী।

### ৫. Recoil (Facebook experimental)

`atom` + `selector` (derived state) একই concept, কিন্তু Provider লাগে এবং Suspense integration আছে।

### ৬. React Context

বিল্ট-ইন, কিন্তু **state management library নয়** — শুধু dependency injection। যেকোনো context value বদলালে সব consumer re-render হয় (selector নেই)। তাই **rarely-changing data** (theme, locale, current user) জন্য ভালো।

```jsx
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={theme}>
      <Layout />
    </ThemeContext.Provider>
  );
}
```

### ৭. MobX — observable + auto-tracking

```javascript
import { makeAutoObservable } from 'mobx';
import { observer } from 'mobx-react-lite';

class CartStore {
  items = [];
  constructor() { makeAutoObservable(this); }
  add(item) { this.items.push(item); }
  get total() { return this.items.reduce((s, i) => s + i.price, 0); }
}

const cart = new CartStore();

const CartView = observer(() => <div>৳{cart.total}</div>);
```

OOP-ফ্রেন্ডলি, magical reactivity। Backend-from-Java/C# devs পছন্দ করেন।

---

## 📋 Comparison Table

| Library     | Boilerplate | Bundle (gz) | DevTools | Selector | Async    | Best For |
|-------------|------------|-------------|----------|----------|----------|----------|
| Redux Toolkit | Medium   | ~12 KB      | ✅ Best  | ✅       | Thunk/Saga/RTK Query | Enterprise, large teams |
| Zustand     | Low        | ~1 KB       | ✅ Yes   | ✅       | Plain async | Most apps, simple-medium |
| Jotai       | Low        | ~3 KB       | ✅ Yes   | atomic   | Plain async | Fine-grained, derived state |
| Recoil      | Medium     | ~14 KB      | Limited  | atomic   | Suspense | Experimental, FB stack |
| MobX        | Low (OOP)  | ~16 KB      | ✅ Yes   | auto     | Plain async | OOP teams |
| Context     | Lowest     | 0 (built-in)| ❌       | ❌       | manual   | Theme, locale, user |
| React Query | Low        | ~13 KB      | ✅ Best  | per-query| native   | Server state |
| SWR         | Lowest     | ~4 KB       | Limited  | per-key  | native   | Server state, simple |

---

## 🌍 Local vs Global — কখন কোনটা?

```
   Local (useState/useReducer)
   ──────────────────────────
   ✅ একটাই component বা ছোট subtree
   ✅ ephemeral data: input value, toggle
   ✅ form draft (যতক্ষণ submit হয়নি)

   Lifted state (parent useState + props)
   ──────────────────────────────────────
   ✅ siblings-এর মধ্যে শেয়ার
   ⚠️ ৩+ level deep হলে → Context বা store

   Global (Zustand/Redux)
   ──────────────────────
   ✅ একাধিক page/route জুড়ে দরকার
   ✅ cart, auth, theme, notifications
   ✅ optimistic update + rollback
   ❌ প্রতিটা component-এর private UI state নয়!

   Server state (React Query/SWR)
   ──────────────────────────────
   ✅ যেকোনো ডেটা যা server থেকে আসে
   ✅ caching, background refetch, dedup
```

---

## ⚡ Optimistic Updates

ইউজার action করার সাথে সাথে UI আপডেট দেখানো (server confirm-এর আগেই)। ব্যর্থ হলে rollback।

```javascript
// React Query দিয়ে — bKash "Send Money" এর মতো
import { useMutation, useQueryClient } from '@tanstack/react-query';

function useSendMoney() {
  const qc = useQueryClient();

  return useMutation({
    mutationFn: (data) => fetch('/api/send-money', {
      method: 'POST', body: JSON.stringify(data),
    }).then(r => r.json()),

    // 1) optimistic update
    onMutate: async (newTx) => {
      await qc.cancelQueries({ queryKey: ['transactions'] });
      const prev = qc.getQueryData(['transactions']);
      qc.setQueryData(['transactions'], (old) => [
        { ...newTx, id: 'temp-' + Date.now(), status: 'pending' },
        ...(old ?? []),
      ]);
      return { prev };
    },

    // 2) error → rollback
    onError: (_err, _vars, ctx) => {
      qc.setQueryData(['transactions'], ctx.prev);
      toast.error('পাঠানো ব্যর্থ হয়েছে');
    },

    // 3) success বা error — server-truth দিয়ে refetch
    onSettled: () => {
      qc.invalidateQueries({ queryKey: ['transactions'] });
    },
  });
}
```

ইউজারের কাছে অ্যাপ "তাত্ক্ষণিক" মনে হয় — bKash, Pathao-তে অপরিহার্য।

---

## 🗃️ Normalization — Entity Pattern

বড় list + nested ডেটায় duplicate এবং stale data এড়াতে normalize করুন।

```javascript
// ❌ Nested (duplicate user data)
const orders = [
  { id: 1, user: { id: 10, name: 'Karim' }, items: [...] },
  { id: 2, user: { id: 10, name: 'Karim' }, items: [...] },  // duplicate!
];

// ✅ Normalized (Redux Toolkit / normalizr)
const state = {
  users: { 10: { id: 10, name: 'Karim' } },
  orders: {
    1: { id: 1, userId: 10, itemIds: [101, 102] },
    2: { id: 2, userId: 10, itemIds: [103] },
  },
  items: { 101: {...}, 102: {...}, 103: {...} },
};
```

**RTK-এর `createEntityAdapter`** এই pattern সরাসরি দেয়:

```javascript
import { createEntityAdapter, createSlice } from '@reduxjs/toolkit';

const ordersAdapter = createEntityAdapter({
  selectId: (o) => o.id,
  sortComparer: (a, b) => b.createdAt - a.createdAt,
});

const ordersSlice = createSlice({
  name: 'orders',
  initialState: ordersAdapter.getInitialState({ loading: false }),
  reducers: {
    addOrder: ordersAdapter.addOne,
    upsertOrders: ordersAdapter.upsertMany,
    removeOrder: ordersAdapter.removeOne,
  },
});

// auto-generated selectors
export const { selectAll, selectById } = ordersAdapter.getSelectors(
  (s) => s.orders
);
```

---

## 🔄 Server State: React Query / TanStack Query

**সবচেয়ে গুরুত্বপূর্ণ shift** গত ৫ বছরে: server state-কে Redux-এ না রেখে dedicated tool ব্যবহার করা।

```javascript
import { useQuery } from '@tanstack/react-query';

function ProductList({ category }) {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ['products', category],
    queryFn: () => fetch(`/api/products?cat=${category}`).then(r => r.json()),
    staleTime: 60_000,           // ১ মিনিট পর্যন্ত fresh
    gcTime: 5 * 60_000,          // ৫ মিনিট পর cache clear
    refetchOnWindowFocus: true,  // tab switch করে আসলে auto refresh
    retry: 3,                    // ব্যর্থ হলে ৩ বার চেষ্টা
  });

  if (isLoading) return <Skeleton />;
  if (error) return <Error onRetry={refetch} />;
  return <Grid items={data} />;
}
```

**যা স্বয়ংক্রিয় হয়:**
- Caching + dedup (একই query একবার fetch)
- Background refetch (stale-while-revalidate)
- Window focus / network reconnect refetch
- Pagination, infinite scroll
- Optimistic updates
- Request cancellation

**SWR** (Vercel-এর) প্রায় একই, আরো ছোট bundle।

---

## 🎯 React-এ re-render অপ্টিমাইজেশন

Global store থেকে re-render কন্ট্রোল করার মূল কৌশল:

```javascript
// ❌ পুরো store subscribe → cart-এর যেকোনো change → re-render
const cart = useCart();

// ✅ শুধু count subscribe → শুধু count change-এ re-render
const count = useCart((s) => s.items.length);

// ✅ একাধিক value lazily — shallow compare
import { shallow } from 'zustand/shallow';
const { count, total } = useCart(
  (s) => ({ count: s.items.length, total: s.total() }),
  shallow
);
```

Redux-এ `useSelector(selectFn)` একই কাজ করে। Reselect দিয়ে memoized selector।

---

## 🇧🇩 বাস্তব উদাহরণ — bKash Transaction List

bKash অ্যাপের "লেনদেন" পেজ একটা চমৎকার case study:

```javascript
// stores/transactions.ts (Zustand + React Query mix)
import { create } from 'zustand';
import { useInfiniteQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// ---- Client UI state (Zustand) ----
export const useTxFilters = create((set) => ({
  type: 'all',                 // all | send | receive | bill
  dateRange: 'last30',
  setType: (type) => set({ type }),
  setDateRange: (dateRange) => set({ dateRange }),
}));

// ---- Server state (React Query, infinite scroll) ----
export function useTransactions() {
  const { type, dateRange } = useTxFilters();

  return useInfiniteQuery({
    queryKey: ['transactions', type, dateRange],
    queryFn: ({ pageParam = 0 }) =>
      fetch(`/api/tx?type=${type}&range=${dateRange}&page=${pageParam}`)
        .then(r => r.json()),
    getNextPageParam: (last) => last.nextPage ?? undefined,
    staleTime: 30_000,
  });
}

// ---- Optimistic Send Money ----
export function useSendMoney() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (payload) => fetch('/api/send', {
      method: 'POST', body: JSON.stringify(payload),
    }).then(r => {
      if (!r.ok) throw new Error('Failed');
      return r.json();
    }),
    onMutate: async (newTx) => {
      // সব transaction queries pause + optimistic add
      await qc.cancelQueries({ queryKey: ['transactions'] });
      const snapshots = qc.getQueriesData({ queryKey: ['transactions'] });
      qc.setQueriesData({ queryKey: ['transactions'] }, (old) => {
        if (!old) return old;
        const optimistic = {
          id: `temp-${Date.now()}`,
          status: 'pending',
          amount: -newTx.amount,
          counterparty: newTx.to,
          ts: Date.now(),
        };
        return {
          ...old,
          pages: [
            { ...old.pages[0], items: [optimistic, ...old.pages[0].items] },
            ...old.pages.slice(1),
          ],
        };
      });
      return { snapshots };
    },
    onError: (_e, _v, ctx) => {
      ctx.snapshots.forEach(([key, data]) => qc.setQueryData(key, data));
    },
    onSettled: () => qc.invalidateQueries({ queryKey: ['transactions'] }),
  });
}
```

**কেন এই split:**
- `useTxFilters` (filter UI) → Zustand: কয়েক ms-এ access, persist করতে চাইলে localStorage middleware।
- transactions list → React Query: caching, refetch, infinite scroll, optimistic — সবই free।
- ভুল হবে যদি transactions list-কে Redux-এ রাখা হত — তখন caching/staleness manually লিখতে হত।

---

## 🏛️ Architecture Pattern: Compose Multiple Approaches

আধুনিক বড় অ্যাপে একটাই tool ব্যবহার করা rare:

```
   ┌──────────────────────────────────────────────────┐
   │              Modern App State Stack              │
   ├──────────────────────────────────────────────────┤
   │  URL state          → React Router / Next.js     │
   │  Server state       → React Query / SWR          │
   │  Global UI state    → Zustand / Jotai            │
   │  Form state         → React Hook Form            │
   │  Local UI state     → useState / useReducer      │
   │  Theme / locale     → React Context              │
   │  Auth / session     → Context + persisted store  │
   └──────────────────────────────────────────────────┘
```

Daraz / Foodpanda মতো অ্যাপে এই layered approach সাধারণ।

---

## ⚠️ সাধারণ Pitfall

| ভুল | পরিণতি | সমাধান |
|-----|--------|---------|
| Server data Redux-এ store | Stale data, manual refetch logic | React Query/SWR |
| Context-এ frequently changing value | সব consumer re-render | Zustand/Jotai/split context |
| Global store-এ form state | অপ্রয়োজনীয় re-render | React Hook Form (uncontrolled) |
| Selector ছাড়া পুরো store সাবস্ক্রাইব | প্রতিটা পরিবর্তনে re-render | `useSelector(s => s.x)` |
| Mutating state directly (Redux/Zustand) | Render না হওয়া | Immer/spread |
| `useEffect` দিয়ে data fetch | Race condition, cleanup ভুল | React Query |
| Filter URL-এ না, state-এ | Reload করলে হারায়, share করা যায় না | URL params |

---

## ✅ চেকলিস্ট

- [ ] প্রতিটি state-কে আপনি ৪ ক্যাটেগরিতে map করেছেন?
- [ ] Server state React Query / SWR-এ?
- [ ] Selector ব্যবহার করে re-render কমানো হয়েছে?
- [ ] Critical mutation-এ optimistic update আছে?
- [ ] Global store সঠিকভাবে normalize করা?
- [ ] Form state form library-তে, global store-এ নয়?
- [ ] URL-এ থাকা উচিত এমন state (filter, pagination) URL-এ আছে?

---

## 📝 সারসংক্ষেপ

State ম্যানেজমেন্ট = "ডেটা কোথায় রাখব" সিদ্ধান্ত। চারটি মূল ক্যাটেগরি — server, client, URL, form — প্রতিটির জন্য আলাদা best tool আছে। **সবচেয়ে বড় win**: server state-কে Redux-এ না রেখে React Query/SWR-এ রাখা — caching, dedup, background refetch সব free হয়। ক্লায়েন্ট UI state-এর জন্য Zustand/Jotai modern minimal পছন্দ; Redux Toolkit এখনো বড় টিমে valid। Context = DI tool, state library নয়। Optimistic update + normalization বড় অ্যাপে অপরিহার্য। বাংলাদেশী bKash/Pathao-র মতো অ্যাপে network unreliable — তাই caching + optimistic UI ব্যবহারকারী experience-এর core।

> 💡 মনে রাখুন: **"Server state caching tool, client state store, form state form library — তিনটে আলাদা।"**
