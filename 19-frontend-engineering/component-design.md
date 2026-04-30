# 🧩 কম্পোনেন্ট ডিজাইন (Component Design)

## ভূমিকা

ফ্রন্টএন্ডে **কম্পোনেন্ট** = UI-এর একক যেগুলো compose হয়ে পেজ তৈরি করে। ভালো component = পুনঃব্যবহারযোগ্য, predictable, accessible, এবং composable। খারাপ component = একটা PR-এ ১০টা prop যোগ হয়, ২ মাস পর কেউ touch করতে চায় না। এই ফাইলে আমরা দেখব **Atomic Design**, **composition patterns** (compound components, render props, hooks, slots), **controlled vs uncontrolled**, **headless components**, **prop drilling solutions**, এবং **accessibility-first** approach।

```
   ভালো component-এর ৫ গুণ
   ─────────────────────────
   1. Single Responsibility  → একটাই কাজ ভালোভাবে
   2. Composable             → ছোট ছোট মিলে বড় UI
   3. Accessible             → keyboard + screen reader
   4. Type-safe              → TS দিয়ে contract
   5. Predictable            → একই input → একই output
```

---

## 🔬 Atomic Design (Brad Frost)

UI-কে chemistry-র মতো ভাবুন: atom → molecule → organism → template → page।

```
┌─────────────────────────────────────────────────────────────┐
│                 Atomic Design Hierarchy                     │
├─────────────────────────────────────────────────────────────┤
│ ATOMS       Button, Input, Label, Icon, Heading             │
│             ⚛️  ছোটতম, কোনো dependency নেই                  │
│                                                             │
│ MOLECULES   SearchBar = Input + Button                      │
│             📌 ২-৪ atom মিলে functional unit                │
│                                                             │
│ ORGANISMS   Header = Logo + Nav + SearchBar + UserMenu      │
│             🧬 একাধিক molecule + atom, business meaningful  │
│                                                             │
│ TEMPLATES   ProductPageTemplate (layout, no real data)      │
│             🖼️  page-level skeleton                         │
│                                                             │
│ PAGES       /product/iphone-15 (real data inserted)        │
│             📄 routable, data-bound                         │
└─────────────────────────────────────────────────────────────┘
```

### বাস্তব folder structure

```
src/components/
├── atoms/
│   ├── Button/
│   ├── Input/
│   ├── Icon/
│   └── Spinner/
├── molecules/
│   ├── SearchBar/
│   ├── PriceTag/
│   ├── RatingStars/
│   └── FormField/      // Label + Input + ErrorText
├── organisms/
│   ├── ProductCard/
│   ├── Header/
│   ├── CartDrawer/
│   └── CheckoutForm/
├── templates/
│   └── ProductPageTemplate/
└── pages/
    └── ProductPage/    // template + data fetching
```

⚠️ Strict atomic অনুসরণ বাধ্যতামূলক নয় — অনেক টিম শুধু `ui/` (atoms+molecules) এবং `features/` (organisms+pages) ব্যবহার করে।

---

## 🧬 Composition Patterns

### ১. Children prop — সরলতম composition

```jsx
function Card({ children }) {
  return <article className="card">{children}</article>;
}

<Card>
  <h2>iPhone 15</h2>
  <p>৳ ১,২০,০০০</p>
  <button>কার্টে যোগ</button>
</Card>
```

### ২. Compound Components

একটি parent + multiple specialized children। Parent context provide করে; children configuration ছাড়াই কাজ করে।

```jsx
// Tabs.tsx
import { createContext, useContext, useState } from 'react';

const TabsCtx = createContext(null);

export function Tabs({ defaultValue, children }) {
  const [active, setActive] = useState(defaultValue);
  return (
    <TabsCtx.Provider value={{ active, setActive }}>
      <div className="tabs">{children}</div>
    </TabsCtx.Provider>
  );
}

Tabs.List = function List({ children }) {
  return <div role="tablist" className="tabs-list">{children}</div>;
};

Tabs.Trigger = function Trigger({ value, children }) {
  const { active, setActive } = useContext(TabsCtx);
  const isActive = active === value;
  return (
    <button
      role="tab"
      aria-selected={isActive}
      tabIndex={isActive ? 0 : -1}
      onClick={() => setActive(value)}
      className={`tab ${isActive ? 'active' : ''}`}
    >
      {children}
    </button>
  );
};

Tabs.Panel = function Panel({ value, children }) {
  const { active } = useContext(TabsCtx);
  if (active !== value) return null;
  return <div role="tabpanel">{children}</div>;
};

// ব্যবহার — declarative, structure দেখা যায়
<Tabs defaultValue="info">
  <Tabs.List>
    <Tabs.Trigger value="info">তথ্য</Tabs.Trigger>
    <Tabs.Trigger value="reviews">রিভিউ</Tabs.Trigger>
    <Tabs.Trigger value="qa">প্রশ্ন</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="info">প্রোডাক্ট বিবরণ...</Tabs.Panel>
  <Tabs.Panel value="reviews">৪.৫ ⭐ ...</Tabs.Panel>
  <Tabs.Panel value="qa">প্রশ্নোত্তর...</Tabs.Panel>
</Tabs>
```

✅ Flexible structure, no prop explosion, semantic HTML।

### ৩. Render Props

Child হিসেবে function pass করুন; parent state দেয়, child ঠিক করে কীভাবে render হবে।

```jsx
function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>
      {children(pos)}
    </div>
  );
}

<MouseTracker>
  {({ x, y }) => <p>Position: {x}, {y}</p>}
</MouseTracker>
```

আজকাল hooks বেশি — কিন্তু `<DataLoader render={...}/>` মতো library API এখনো এই pattern ব্যবহার করে।

### ৪. Custom Hooks

State + behavior reuse করার আধুনিক pattern।

```javascript
// useDisclosure.ts — modal/drawer/popover এর জন্য
export function useDisclosure(initial = false) {
  const [isOpen, setOpen] = useState(initial);
  return {
    isOpen,
    open: useCallback(() => setOpen(true), []),
    close: useCallback(() => setOpen(false), []),
    toggle: useCallback(() => setOpen((v) => !v), []),
  };
}

// ব্যবহার
function Header() {
  const cart = useDisclosure();
  const menu = useDisclosure();
  return (
    <>
      <button onClick={cart.toggle}>🛒</button>
      <button onClick={menu.toggle}>☰</button>
      <CartDrawer open={cart.isOpen} onClose={cart.close} />
      <MobileMenu open={menu.isOpen} onClose={menu.close} />
    </>
  );
}
```

### ৫. Slots Pattern

Multiple "named children" — Vue-এ native, React-এ prop দিয়ে।

```jsx
function PageLayout({ header, sidebar, children, footer }) {
  return (
    <div className="layout">
      <header className="layout__header">{header}</header>
      <aside className="layout__sidebar">{sidebar}</aside>
      <main className="layout__main">{children}</main>
      <footer className="layout__footer">{footer}</footer>
    </div>
  );
}

<PageLayout
  header={<Header />}
  sidebar={<CategoryNav />}
  footer={<Footer />}
>
  <ProductGrid />
</PageLayout>
```

### ৬. Higher-Order Components (HOC) — পুরোনো

```jsx
function withLogger(Wrapped) {
  return function Logged(props) {
    useEffect(() => console.log('mount', Wrapped.name), []);
    return <Wrapped {...props} />;
  };
}

const ButtonWithLog = withLogger(Button);
```

আজকাল hooks-এ replace হয়ে গেছে; HOC শুধু legacy বা library API তে।

---

## 🎮 Controlled vs Uncontrolled

### Controlled — parent state-এ value

```jsx
function PriceFilter() {
  const [price, setPrice] = useState(0);
  return (
    <input
      type="number"
      value={price}                            // ← single source of truth
      onChange={(e) => setPrice(+e.target.value)}
    />
  );
}
```

✅ Predictable, validation possible, easy testing।
❌ Every keystroke re-renders parent।

### Uncontrolled — DOM-এ value

```jsx
function ContactForm() {
  const inputRef = useRef(null);
  return (
    <form onSubmit={() => alert(inputRef.current.value)}>
      <input ref={inputRef} defaultValue="" />
      <button>Submit</button>
    </form>
  );
}
```

✅ Performant (no re-render), simpler for one-shot forms।
❌ Hard to derive UI state, validate on-change।

### Hybrid: React Hook Form

Best of both — uncontrolled inputs + state observation:

```jsx
import { useForm } from 'react-hook-form';

function CheckoutForm() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  return (
    <form onSubmit={handleSubmit((data) => placeOrder(data))}>
      <input
        {...register('phone', {
          required: 'মোবাইল নাম্বার দিন',
          pattern: { value: /^01[3-9]\d{8}$/, message: 'সঠিক BD নাম্বার দিন' },
        })}
      />
      {errors.phone && <span>{errors.phone.message}</span>}

      <input
        {...register('amount', { required: true, min: 10, valueAsNumber: true })}
      />

      <button type="submit">অর্ডার করুন</button>
    </form>
  );
}
```

ইনপুট uncontrolled → কোনো re-render নেই; শুধু error/submit-এ component update।

---

## 🎩 Headless Components

UI **logic** আলাদা, **markup/style** ব্যবহারকারী লেখেন। এটা আজকের library design-এ dominant pattern (Radix UI, Headless UI, React Aria, TanStack)।

```jsx
// useCombobox.ts — headless logic
export function useCombobox({ items, onSelect }) {
  const [query, setQuery] = useState('');
  const [highlighted, setHighlighted] = useState(0);
  const filtered = items.filter((i) =>
    i.label.toLowerCase().includes(query.toLowerCase())
  );

  function onKeyDown(e) {
    if (e.key === 'ArrowDown') setHighlighted((h) => Math.min(h + 1, filtered.length - 1));
    if (e.key === 'ArrowUp')   setHighlighted((h) => Math.max(h - 1, 0));
    if (e.key === 'Enter')     onSelect(filtered[highlighted]);
  }

  return {
    inputProps: {
      value: query,
      onChange: (e) => setQuery(e.target.value),
      onKeyDown,
      role: 'combobox',
      'aria-expanded': filtered.length > 0,
    },
    items: filtered.map((it, i) => ({
      ...it,
      itemProps: {
        role: 'option',
        'aria-selected': i === highlighted,
        onMouseEnter: () => setHighlighted(i),
        onClick: () => onSelect(it),
      },
    })),
  };
}

// User defines markup
function CitySearch() {
  const { inputProps, items } = useCombobox({
    items: cities,
    onSelect: (c) => console.log(c),
  });

  return (
    <div className="combobox">
      <input {...inputProps} className="my-input" />
      <ul>
        {items.map((it) => (
          <li key={it.id} {...it.itemProps} className="my-option">
            {it.label}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

✅ Total style freedom, accessibility built-in, swappable markup।
✅ Real examples: **Radix UI**, **Headless UI** (Tailwind), **React Aria** (Adobe), **TanStack Table**।

---

## 🪜 Prop Drilling — সমস্যা ও সমাধান

```jsx
// ❌ Prop drilling — user ৪ level deep পাঠানো
<App>
  <Layout user={user}>
    <Header user={user}>
      <Nav user={user}>
        <Avatar user={user} />
```

**সমাধান:**

| পদ্ধতি            | কখন ব্যবহার                              |
|--------------------|------------------------------------------|
| Composition (children) | Layout-type problem (slot inversion) |
| Context           | Theme, locale, current user, auth        |
| Zustand/Jotai     | Frequently changing global state         |
| Component split   | Container/presentational, smaller tree   |

**Composition fix:**
```jsx
// Child কে Header build-এ দাও, Header-কে user pass করতে হবে না
<Layout header={<Header><Avatar user={user} /></Header>}>
  ...
</Layout>
```

---

## ♿ Accessibility-first Components

প্রতিটি interactive component-এ ৪টি জিনিস দরকার:

1. **Semantic HTML** — `<button>`, `<a>`, `<nav>`, `<dialog>` (div+role নয়)
2. **Keyboard navigation** — Tab, Enter, Esc, Arrow keys
3. **ARIA attributes** — যেখানে semantic যথেষ্ট নয়
4. **Focus management** — modal খুললে focus inside, বন্ধ হলে trigger-এ ফিরে

### সঠিক Modal/Dialog উদাহরণ

```jsx
import { useEffect, useRef } from 'react';

function Modal({ open, onClose, title, children }) {
  const dialogRef = useRef(null);
  const previouslyFocused = useRef(null);

  useEffect(() => {
    if (open) {
      previouslyFocused.current = document.activeElement;
      dialogRef.current?.focus();
      // body scroll lock
      document.body.style.overflow = 'hidden';
    } else {
      previouslyFocused.current?.focus();
      document.body.style.overflow = '';
    }
  }, [open]);

  useEffect(() => {
    if (!open) return;
    const onKey = (e) => e.key === 'Escape' && onClose();
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [open, onClose]);

  if (!open) return null;

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        tabIndex={-1}
        className="modal"
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id="modal-title">{title}</h2>
        {children}
        <button onClick={onClose} aria-label="বন্ধ করুন">✕</button>
      </div>
    </div>
  );
}
```

বাস্তবে `<dialog>` element + `react-aria` / `radix-ui` ব্যবহার করুন — focus trap, scroll lock, portal — সব built-in।

---

## 🍔 Real-world: Foodpanda Restaurant Card

Foodpanda-র restaurant card একটা চমৎকার organism। ভাঙার চেষ্টা করি:

```
   ┌────────────────────────────────────────┐
   │ [IMAGE 16:9]                    [♥]   │  ← FavoriteButton (atom)
   │                                        │
   │ Pizza Hut                       4.6⭐  │  ← Heading + RatingStars
   │ ৳ ৳  •  Pizza, Italian                 │  ← PriceLevel + CuisineTags
   │                                        │
   │ 🚴 ৩০ মিনিট  •  ৳ ৩০ ডেলিভারি          │  ← DeliveryInfo (molecule)
   │                                        │
   │ [📌 Free delivery]  [⚡ Top choice]    │  ← BadgeList (molecule)
   └────────────────────────────────────────┘
```

```jsx
// Atomic breakdown
// Atoms:
function Image({ src, alt, aspectRatio = '16/9', loading = 'lazy' }) { /* ... */ }
function Heading({ level = 3, children }) { /* ... */ }
function Badge({ icon, children, variant }) { /* ... */ }
function FavoriteButton({ active, onToggle }) { /* ... */ }

// Molecules:
function RatingStars({ value, count }) { /* ... */ }
function DeliveryInfo({ minutes, fee, free }) {
  return (
    <div className="delivery-info">
      <span>🚴 {minutes} মিনিট</span>
      <span>•</span>
      <span>{free ? 'ফ্রি ডেলিভারি' : `৳ ${fee} ডেলিভারি`}</span>
    </div>
  );
}

// Organism:
type RestaurantCardProps = {
  id: string;
  name: string;
  imageUrl: string;
  rating: number;
  reviewCount: number;
  cuisines: string[];
  priceLevel: 1 | 2 | 3;
  deliveryTime: number;
  deliveryFee: number;
  freeDelivery: boolean;
  badges: Array<{ icon: string; label: string }>;
  isFavorite: boolean;
  onClick: () => void;
  onFavoriteToggle: () => void;
};

export function RestaurantCard(p: RestaurantCardProps) {
  return (
    <article
      className="restaurant-card"
      onClick={p.onClick}
      role="button"
      tabIndex={0}
      onKeyDown={(e) => e.key === 'Enter' && p.onClick()}
      aria-label={`${p.name}, ${p.rating} stars, ${p.deliveryTime} মিনিট`}
    >
      <div className="restaurant-card__media">
        <Image src={p.imageUrl} alt={p.name} loading="lazy" />
        <FavoriteButton
          active={p.isFavorite}
          onToggle={(e) => { e.stopPropagation(); p.onFavoriteToggle(); }}
        />
      </div>

      <div className="restaurant-card__body">
        <div className="restaurant-card__title-row">
          <Heading level={3}>{p.name}</Heading>
          <RatingStars value={p.rating} count={p.reviewCount} />
        </div>

        <p className="restaurant-card__meta">
          {'৳'.repeat(p.priceLevel)} • {p.cuisines.join(', ')}
        </p>

        <DeliveryInfo
          minutes={p.deliveryTime}
          fee={p.deliveryFee}
          free={p.freeDelivery}
        />

        {p.badges.length > 0 && (
          <ul className="restaurant-card__badges">
            {p.badges.map((b) => (
              <li key={b.label}>
                <Badge icon={b.icon}>{b.label}</Badge>
              </li>
            ))}
          </ul>
        )}
      </div>
    </article>
  );
}
```

**শেখার পয়েন্ট:**
- Card একটা semantic `<article>`
- Click + keyboard support
- Image lazy load + alt text
- Favorite button event propagation আটকায় (card click trigger না হয়)
- Sub-components (atoms/molecules) আলাদা testable
- Type-safe props — required vs optional clear

---

## ⚠️ সাধারণ Pitfall

| ভুল                           | সমস্যা                          | সমাধান                          |
|--------------------------------|----------------------------------|---------------------------------|
| ১৫+ prop একটাই component       | Configuration explosion          | Compound components             |
| Prop drilling ৫+ level         | Refactor hell                    | Composition / Context           |
| `div` দিয়ে button             | Keyboard + screen reader fail   | `<button>` ব্যবহার              |
| `useState` server data রাখা    | Stale, manual refetch           | React Query                     |
| Event handler inline lambda    | Re-render performance           | `useCallback` only when needed   |
| Memoization everywhere          | Premature optimization           | শুধু measured slow component-এ |
| Default export anonymous       | Hard to debug                    | Named export                    |
| Component file ৫০০+ লাইন      | Untouchable                     | Split into atoms                |
| Side effects render-এ          | Bug, extra fetch                 | `useEffect` / event handlers    |
| Modal না trap focus           | Accessibility fail              | Focus trap library / manual     |

---

## ✅ চেকলিস্ট — Production Component

- [ ] একটাই দায়িত্ব (single responsibility)
- [ ] Props interface clear, TypeScript-এ documented
- [ ] Required vs optional prop স্পষ্ট
- [ ] Semantic HTML element
- [ ] Keyboard navigation কাজ করে
- [ ] ARIA attributes যেখানে দরকার
- [ ] Focus management (modal/drawer)
- [ ] Loading + error + empty state
- [ ] Storybook story আছে
- [ ] Unit test (logic) + Visual test (snapshot)
- [ ] i18n-ready (text hardcoded নয়)
- [ ] Theme-aware (CSS variable / theme prop)

---

## 📝 সারসংক্ষেপ

ভালো component design = **single responsibility + composition + accessibility + type safety**। **Atomic Design** structure দেয় (atom→molecule→organism→template→page)। **Composition patterns** (children, compound components, render props, hooks, slots, headless) prop explosion এবং prop drilling-এর সমাধান। **Controlled vs uncontrolled** trade-off বুঝে fit choose করুন; ফর্মে React Hook Form best of both। **Headless components** (Radix, Aria) modern library design philosophy। প্রতিটি interactive component-এ keyboard + ARIA + focus management লাগবে। বাংলাদেশী context-এ Foodpanda restaurant card, Daraz product card, bKash transaction row — সব এই pattern অনুসরণ করে।

> 💡 মনে রাখুন: **"Component যত ছোট ও focused, reuse তত সহজ। Compose করুন, configure নয়।"**
