# 🔌 প্রোভাইডার প্যাটার্ন (Provider Pattern)

> **ক্যাটাগরি:** React Design Pattern
> **উৎস:** patterns.dev/react/provider-pattern

---

## সংজ্ঞা

**প্রোভাইডার প্যাটার্ন** React Context API ব্যবহার করে ডাটা **একাধিক চাইল্ড কমপোনেন্টের** কাছে সরাসরি পৌঁছে দেয় — প্রতিটি লেভেলে props পাস না করে (Prop Drilling ছাড়াই)।

```
   Prop Drilling (❌ সমস্যা)
   ══════════════════════════════════════════════════

   App (theme="dark")
     ↓ props.theme
     Layout
       ↓ props.theme
       Sidebar
         ↓ props.theme
           MenuItem ← এখানে দরকার, কিন্তু ৪ লেভেল নিচে!


   Provider Pattern (✅ সমাধান)
   ══════════════════════════════════════════════════

   ThemeProvider (theme="dark")
   ┌───────────────────────────────────┐
   │  App                              │
   │    Layout                         │
   │      Sidebar                      │
   │        MenuItem ← সরাসরি context │
   └───────────────────────────────────┘
```

---

## 🏠 বাস্তব উদাহরণ (Bangladesh Context)

> Daraz-এ `UserProvider` দিয়ে লগড-ইন ইউজারের তথ্য সব কমপোনেন্টে পাঠানো হয়।
> bKash-এ `ThemeProvider` দিয়ে ডার্ক/লাইট মোড, `LocaleProvider` দিয়ে বাংলা/English নিয়ন্ত্রণ।

---

## ✅ মৌলিক উদাহরণ — Theme Provider

```jsx
import { createContext, useContext, useState } from 'react';

// ১. Context তৈরি করুন
const ThemeContext = createContext();

// ২. Custom hook — সহজে ব্যবহারের জন্য
export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be used within ThemeProvider');
  return context;
}

// ৩. Provider কমপোনেন্ট
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');

  function toggleTheme() {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  }

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// ৪. ব্যবহার — যেকোনো নেস্টেড কমপোনেন্টে
function Header() {
  const { theme, toggleTheme } = useTheme();
  return (
    <header className={`header header--${theme}`}>
      <button onClick={toggleTheme}>
        {theme === 'light' ? '🌙 ডার্ক' : '☀️ লাইট'}
      </button>
    </header>
  );
}

// ৫. App-এ wrap করুন
function App() {
  return (
    <ThemeProvider>
      <Header />
      <MainContent />
      <Footer />
    </ThemeProvider>
  );
}
```

---

## ✅ bKash-স্টাইল User Auth Provider

```jsx
import { createContext, useContext, useState, useCallback } from 'react';

const AuthContext = createContext(null);

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}

export function AuthProvider({ children }) {
  const [user, setUser]       = useState(null);
  const [loading, setLoading] = useState(false);

  const login = useCallback(async (phone, pin) => {
    setLoading(true);
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        body: JSON.stringify({ phone, pin }),
      });
      const data = await response.json();
      setUser(data.user);
      return data;
    } finally {
      setLoading(false);
    }
  }, []);

  const logout = useCallback(() => {
    setUser(null);
    localStorage.removeItem('token');
  }, []);

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// যেকোনো কমপোনেন্টে
function TransactionButton() {
  const { user, logout } = useAuth();

  return user ? (
    <div>
      <span>স্বাগতম, {user.name}</span>
      <button onClick={logout}>লগআউট</button>
    </div>
  ) : (
    <LoginForm />
  );
}
```

---

## ✅ Multiple Providers — কম্পোজিশন

```jsx
// প্রতিটি concern আলাদা Provider-এ
function AppProviders({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LocaleProvider locale="bn">
          <CartProvider>
            {children}
          </CartProvider>
        </LocaleProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

function App() {
  return (
    <AppProviders>
      <Router>
        <AppRoutes />
      </Router>
    </AppProviders>
  );
}
```

---

## ✅ পারফরম্যান্স অপ্টিমাইজেশন — useMemo

```jsx
// Context value পরিবর্তন হলে সব consumer রি-রেন্ডার হয়
// useMemo দিয়ে অপ্রয়োজনীয় রি-রেন্ডার রোধ করুন

export function CartProvider({ children }) {
  const [items, setItems]   = useState([]);
  const [total, setTotal]   = useState(0);

  const addItem = useCallback((item) => {
    setItems(prev => [...prev, item]);
    setTotal(prev => prev + item.price);
  }, []);

  // ✅ value মেমোাইজ করুন
  const value = useMemo(
    () => ({ items, total, addItem }),
    [items, total, addItem]
  );

  return (
    <CartContext.Provider value={value}>
      {children}
    </CartContext.Provider>
  );
}
```

---

## ❌ সমস্যা vs ✅ সমাধান

```jsx
// ❌ Prop Drilling — প্রতিটি লেভেলে theme পাস করা
<App theme="dark">
  <Layout theme="dark">
    <Sidebar theme="dark">
      <MenuItem theme="dark" /> {/* এখানেই দরকার ছিল */}
    </Sidebar>
  </Layout>
</App>

// ✅ Provider Pattern
<ThemeProvider>
  <App>
    <Layout>
      <Sidebar>
        <MenuItem /> {/* useTheme() দিয়ে সরাসরি পায় */}
      </Sidebar>
    </Layout>
  </App>
</ThemeProvider>
```

---

## 🔄 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

| প্যাটার্ন | সম্পর্ক |
|----------|---------|
| Observer | Context পরিবর্তন হলে সব consumer (observer) আপডেট হয় |
| Singleton | Context একটি Singleton-এর মতো — একটাই ইনস্ট্যান্স |
| Compound Component | Compound Component-এ Context শেয়ার করা হয় |
| HOC | HOC দিয়ে Context ইনজেক্ট করা হতো (hooks-এর আগে) |

---

## ✅ সুবিধা (Pros)

- **Prop drilling দূর** — গভীর নেস্টেড কমপোনেন্টে সহজে ডাটা
- **কেন্দ্রীভূত স্টেট** — এক জায়গায় ম্যানেজমেন্ট
- **পুনঃব্যবহারযোগ্য** — যেকোনো কমপোনেন্ট consume করতে পারে

## ❌ অসুবিধা (Cons)

- **ওভারব্যবহার** — সব কিছু context-এ না রাখাই ভালো
- **রি-রেন্ডার** — value পরিবর্তে সব consumer রি-রেন্ডার হয়
- **টেস্টিং** — মক provider লাগে

---

## 📝 কখন ব্যবহার করবেন

- ✅ Theme, Locale, Auth — গ্লোবাল স্টেটের জন্য
- ✅ ৩+ লেভেল গভীর prop drilling এড়াতে
- ❌ Frequently update হওয়া ডাটা (প্রতি সেকেন্ডে বদলায়) — Redux/Zustand ভালো
- ❌ শুধু ২ লেভেলের জন্য — সরাসরি props যথেষ্ট
