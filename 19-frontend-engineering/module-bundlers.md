# 📦 মডিউল বান্ডলার (Module Bundlers)

## ভূমিকা

আধুনিক ফ্রন্টএন্ডে আপনি লেখেন ES modules, TypeScript, JSX, Sass, PostCSS। কিন্তু ব্রাউজার শুধু (এখনো) plain JS, CSS, HTML বোঝে। **Module Bundler** এই gap পূরণ করে — আপনার শত শত source file নিয়ে dependency graph বানায়, transform করে, optimize করে, এবং deploy-ready bundle তৈরি করে। Webpack, Vite, Rollup, esbuild, Turbopack, Parcel — প্রতিটির আলাদা philosophy।

```
   src/                              dist/
   ├── App.tsx                       ├── index.html
   ├── components/         ━━━━▶    ├── assets/
   │   ├── Header.tsx     bundler   │   ├── main.[hash].js
   │   └── Card.tsx                 │   ├── vendor.[hash].js
   ├── styles.scss                  │   └── main.[hash].css
   └── api.ts                       └── ...
```

---

## 🏗️ Bundler কী করে — ৫টি মূল কাজ

```
   ┌──────────────────────────────────────────────────────┐
   │              Module Bundler Pipeline                 │
   ├──────────────────────────────────────────────────────┤
   │ 1. ENTRY        → entry file(s) থেকে শুরু             │
   │ 2. RESOLVE      → import path → file path             │
   │ 3. LOAD/PARSE   → file content → AST                  │
   │ 4. TRANSFORM    → TS→JS, JSX→JS, SCSS→CSS, Babel       │
   │ 5. OUTPUT       → graph → optimized chunks/assets     │
   └──────────────────────────────────────────────────────┘
```

প্রতিটি bundler এই ৫ ধাপ করে, কিন্তু **কখন এবং কীভাবে** করে — এতেই পার্থক্য।

---

## ⚔️ Bundler-দের তুলনা

| Bundler    | ভাষা        | Dev mode      | Prod build     | HMR     | Best for             |
|------------|-------------|---------------|----------------|---------|----------------------|
| **Webpack** | JS         | Bundle (slow) | Mature, configurable | Good | Legacy, complex apps |
| **Vite**   | JS+Rust(esbuild+rollup) | Native ESM (fast) | Rollup-based | Excellent | Modern apps (default choice) |
| **Rollup** | JS         | N/A (build only) | Excellent for libs | N/A | Libraries, npm packages |
| **esbuild**| Go         | Fastest       | Basic optimizations | Limited | Speed-critical, simple |
| **Turbopack** | Rust    | Fast (incremental) | Beta | Excellent | Next.js (future) |
| **Parcel** | JS+Rust    | Zero-config   | Good           | Good    | Quick prototypes |
| **SWC**    | Rust       | Compiler only | (use in others)| -       | Babel replacement |
| **Bun**    | Zig        | Built-in bundler | Fast | Good | Bun runtime apps |

```
   Build Speed (relative — small project)
   ───────────────────────────────────────
   esbuild   ▏ 0.1s       ⚡⚡⚡⚡⚡
   Vite      ▏ 0.3s       ⚡⚡⚡⚡
   Turbopack ▏ 0.5s       ⚡⚡⚡
   Parcel    ▏ 1s         ⚡⚡
   Rollup    ▏ 2s         ⚡⚡
   Webpack   ▏ 5-30s      ⚡
```

---

## 🌳 Tree Shaking

**Tree shaking** মানে dead code elimination — যা import হয়নি, তা bundle-এ আসবে না। এর জন্য **ES Modules (`import/export`)** চাই, কারণ static analysis সম্ভব।

```javascript
// utils.js
export function add(a, b) { return a + b; }
export function multiply(a, b) { return a * b; }
export function divide(a, b) { return a / b; }   // ব্যবহার হচ্ছে না

// app.js
import { add } from './utils';
console.log(add(2, 3));
```

✅ Bundle-এ শুধু `add` যাবে, `multiply` ও `divide` বাদ।

❌ **Tree shaking ভাঙে যা:**
```javascript
// CommonJS (require) static নয়
const utils = require('./utils');

// Side effects
import './polyfill';   // entire file রাখতে হবে

// Dynamic
import(path);
```

**`package.json` এ side-effects flag**:
```json
{
  "name": "my-lib",
  "sideEffects": false      // অথবা ["*.css", "src/polyfill.js"]
}
```

এতে bundler আগ্রাসীভাবে dead code remove করতে পারে।

---

## ✂️ Code Splitting

পুরো অ্যাপের কোড একটাই বিশাল bundle হলে first load slow। Code splitting মানে: প্রয়োজন অনুযায়ী আলাদা chunk।

### টাইপ ১: Route-based splitting

```javascript
// React Router + dynamic import
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Home    = lazy(() => import('./pages/Home'));
const Cart    = lazy(() => import('./pages/Cart'));
const Profile = lazy(() => import('./pages/Profile'));

export function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/cart" element={<Cart />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}
```

ফলাফল: প্রতিটি route আলাদা chunk। ইউজার `/cart` না গেলে cart কোড download হবে না।

### টাইপ ২: Component-level splitting

```javascript
// Heavy chart library শুধু modal খুললে load হবে
const ChartModal = lazy(() => import('./ChartModal'));

function Dashboard() {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button onClick={() => setOpen(true)}>📊 Show Chart</button>
      {open && (
        <Suspense fallback={<Skeleton />}>
          <ChartModal onClose={() => setOpen(false)} />
        </Suspense>
      )}
    </>
  );
}
```

### টাইপ ৩: Vendor splitting

`react`, `lodash`, `date-fns` মতো dependency-গুলো আলাদা vendor chunk-এ — কারণ এরা কম change হয়, browser cache long-term ধরে রাখে।

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
  },
};
```

---

## ⚡ Dynamic Import + Lazy Loading

```javascript
// সব dynamic import → আলাদা chunk
button.addEventListener('click', async () => {
  const { exportToPDF } = await import('./pdfExport');
  exportToPDF(data);
});

// Webpack magic comments
const Module = await import(
  /* webpackChunkName: "report" */
  /* webpackPrefetch: true */     // idle time-এ prefetch
  './Report'
);
```

**`prefetch` vs `preload`:**
- `prefetch` — ব্রাউজার idle হলে ডাউনলোড। "ভবিষ্যতে লাগতে পারে।"
- `preload` — তখনই ডাউনলোড। "এখনই দরকার, কিন্তু parser আবিষ্কার করার আগেই।"

---

## ⚙️ Webpack Config — Production-ready

```javascript
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CompressionPlugin = require('compression-webpack-plugin');

module.exports = (env, argv) => {
  const isProd = argv.mode === 'production';
  return {
    entry: './src/index.tsx',
    output: {
      path: path.resolve(__dirname, 'dist'),
      filename: isProd ? 'js/[name].[contenthash:8].js' : 'js/[name].js',
      chunkFilename: 'js/[name].[contenthash:8].chunk.js',
      publicPath: '/',
      clean: true,
    },
    resolve: {
      extensions: ['.tsx', '.ts', '.jsx', '.js'],
      alias: {
        '@': path.resolve(__dirname, 'src'),
      },
    },
    module: {
      rules: [
        {
          test: /\.[jt]sx?$/,
          exclude: /node_modules/,
          use: { loader: 'swc-loader' },   // Babel-এর ১০x দ্রুত
        },
        {
          test: /\.scss$/,
          use: [
            isProd ? MiniCssExtractPlugin.loader : 'style-loader',
            'css-loader',
            'postcss-loader',
            'sass-loader',
          ],
        },
        {
          test: /\.(png|jpg|svg|webp|avif)$/,
          type: 'asset',
          parser: { dataUrlCondition: { maxSize: 8 * 1024 } },
        },
      ],
    },
    optimization: {
      splitChunks: { chunks: 'all' },
      runtimeChunk: 'single',
      moduleIds: 'deterministic',     // long-term caching
    },
    plugins: [
      new HtmlWebpackPlugin({ template: './public/index.html' }),
      isProd && new MiniCssExtractPlugin({
        filename: 'css/[name].[contenthash:8].css',
      }),
      isProd && new CompressionPlugin({
        algorithm: 'brotliCompress',
        test: /\.(js|css|html|svg)$/,
      }),
    ].filter(Boolean),
    devtool: isProd ? 'source-map' : 'eval-cheap-module-source-map',
    devServer: {
      hot: true,
      port: 3000,
      historyApiFallback: true,
    },
  };
};
```

---

## ⚡ Vite Config

Vite-এর philosophy: **dev mode এ bundle করো না, native ESM ব্যবহার করো**। ব্রাউজার সরাসরি import resolve করে → instant cold start। শুধু production-এ Rollup bundle।

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react-swc';
import { visualizer } from 'rollup-plugin-visualizer';
import path from 'path';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true, gzipSize: true }),
  ],
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src') },
  },
  build: {
    target: 'es2020',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          react: ['react', 'react-dom', 'react-router-dom'],
          query: ['@tanstack/react-query'],
          charts: ['recharts', 'd3'],
        },
      },
    },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': 'http://localhost:8000',
    },
  },
});
```

```
   Vite Dev Mode (native ESM)
   ───────────────────────────
   Browser  ──GET── /src/App.tsx ──▶ Vite server
                                       │
                                       ▼
                                   on-demand transform
                                   (esbuild for TS/JSX)
                                       │
                                       ▼
                              return as ESM
   
   ✅ কোনো bundle নেই → cold start ১০০ms
   ✅ HMR file-level → শুধু changed module reload
```

---

## 🔥 HMR (Hot Module Replacement)

কোড বদলালে full page reload না করে শুধু changed module swap। React state preserved থাকে।

```javascript
// Vite এ React Fast Refresh built-in
// Custom module-এ HMR API:
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    console.log('Reloaded:', newModule);
  });
  import.meta.hot.dispose(() => {
    // cleanup before swap
  });
}
```

**HMR vs Live Reload:**
- Live reload = পুরো page reload (state হারায়)
- HMR = শুধু changed module swap (state থাকে) ✅

---

## 📊 Source Maps

Production-এ minified code → debug করা অসম্ভব। Source map browser-কে original source দেখায়।

| Mode                              | Build speed | Quality  | Production safe? |
|-----------------------------------|-------------|----------|-------------------|
| `eval`                            | ⚡⚡⚡⚡⚡  | ⭐       | ❌ (dev only)    |
| `eval-cheap-module-source-map`    | ⚡⚡⚡⚡    | ⭐⭐⭐    | ❌ (dev)         |
| `cheap-module-source-map`         | ⚡⚡⚡      | ⭐⭐⭐    | ⚠️                |
| `source-map`                      | ⚡         | ⭐⭐⭐⭐⭐ | ✅ (separate file) |
| `hidden-source-map`               | ⚡         | ⭐⭐⭐⭐⭐ | ✅ (Sentry only)  |

**Production tip**: `hidden-source-map` upload করুন Sentry/Datadog-এ, কিন্তু serve করবেন না — original code expose হবে।

---

## 📈 Bundle Analysis

```bash
# Webpack
npm i -D webpack-bundle-analyzer

# Vite
npm i -D rollup-plugin-visualizer
```

আউটপুট: একটা treemap যেখানে প্রতিটি module-এর size visible।

```
   Bundle Treemap (visualizer)
   ────────────────────────────
   ┌────────────────────────────────────┐
   │  vendor.js (180 KB gz)             │
   │  ┌────────────┬───────────────┐    │
   │  │ react-dom  │   moment.js   │    │ ← ⚠️ ৭০KB!  date-fns দিয়ে replace করো
   │  │   45 KB    │     70 KB     │    │
   │  ├────────────┼───────────────┤    │
   │  │ lodash 50KB│ rest...       │    │ ← lodash-es + tree shake
   │  └────────────┴───────────────┘    │
   └────────────────────────────────────┘
```

**Common findings:**
- `moment.js` (300KB unminified) → `dayjs` বা `date-fns` (~10KB)
- `lodash` full → `lodash-es` per-function import
- পুরো icon library import → per-icon import
- Polyfills যা modern browser-এ লাগে না

---

## 🌐 Module Federation (Webpack 5)

Multiple deployed apps runtime-এ component শেয়ার করে — Micro-frontend-এর foundation।

```javascript
// host (shell) webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  // ...
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        cart:    'cart@https://cart.daraz.com.bd/remoteEntry.js',
        catalog: 'catalog@https://catalog.daraz.com.bd/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true },
      },
    }),
  ],
};

// remote (cart team) webpack.config.js
new ModuleFederationPlugin({
  name: 'cart',
  filename: 'remoteEntry.js',
  exposes: {
    './CartButton': './src/CartButton',
    './CartPage':   './src/pages/Cart',
  },
  shared: { react: { singleton: true } },
});

// host-এ ব্যবহার
const CartButton = lazy(() => import('cart/CartButton'));
```

---

## 🇧🇩 বাংলাদেশী কনটেক্সট

### Daraz BD chunk strategy

মেগা সেল-এর সময় হোমপেজ + product page critical। তাই:
- `home.chunk.js` + `product.chunk.js` — preload
- `checkout.chunk.js` — prefetch (cart-এ ক্লিক করার পর)
- `seller-portal.chunk.js` — শুধু seller route-এ load

ফলে 3G ইউজারের জন্য initial bundle ~150KB gz, পুরো অ্যাপ ~600KB।

### Foodpanda PWA + bundler

Vite + Workbox plugin দিয়ে service worker generate। Asset hashing → long-term cache। User offline-এ গেলেও menu cached থাকে।

### Pathao map page

ম্যাপ লাইব্রেরি (Mapbox/Google Maps SDK) ~200KB। শুধু booking route-এ async load — হোমপেজ ফাঁকা থাকে। `<link rel="prefetch">` দিয়ে hover-এ prefetch।

### bKash internal portal

Webpack 5 + Module Federation দিয়ে separate teams (transactions, KYC, settlements) আলাদা deploy। Shell অ্যাপ runtime-এ compose করে।

---

## ⚠️ সাধারণ Pitfalls

| ভুল                              | সমস্যা                          | সমাধান                          |
|-----------------------------------|----------------------------------|----------------------------------|
| `import _ from 'lodash'`          | পুরো লাইব্রেরি bundle-এ        | `import debounce from 'lodash/debounce'` বা `lodash-es` |
| `moment.js` ব্যবহার               | 300KB+, locale-heavy            | `dayjs`, `date-fns`              |
| Polyfill সব browser-এ           | Modern browser-এ অপ্রয়োজনীয়   | `browserslist` + `@babel/preset-env` `useBuiltIns: 'usage'` |
| Single huge bundle                | First load slow                  | Route-based splitting            |
| Vendor chunk সবসময় change       | Cache invalidate                 | `splitChunks` + content hash    |
| Source map prod-এ public         | Source code leak                | `hidden-source-map` + Sentry    |
| CommonJS-only dependency          | Tree shaking fail                | ESM version বা manual exclude   |
| HMR না কাজ করা                   | Anonymous default export        | Named export ব্যবহার            |
| Image সরাসরি import (large)      | Bundle-এ embed                   | `asset/resource` বা CDN         |

---

## ✅ চেকলিস্ট — Production Bundle

- [ ] Tree shaking enabled (ESM, `sideEffects: false`)
- [ ] Route-based code splitting
- [ ] Vendor chunk আলাদা, content-hash naming
- [ ] CSS extracted (MiniCssExtractPlugin)
- [ ] Compression: gzip + brotli
- [ ] Image optimized (`asset/resource`, modern formats)
- [ ] Bundle analyzer report check করেছেন
- [ ] `moment.js`/full `lodash`/`jquery` নেই
- [ ] Source map: `hidden-source-map` + monitoring tool-এ upload
- [ ] Long-term caching: `[contenthash]` filenames
- [ ] Polyfills only-as-needed (browserslist)
- [ ] Lighthouse audit, Performance score 90+

---

## 📝 সারসংক্ষেপ

Module bundler আপনার source code-কে ব্রাউজার-ready bundle বানায় — resolve, transform, optimize, output। **Webpack** mature কিন্তু slow, **Vite** modern ও বেশিরভাগ অ্যাপের default choice (native ESM dev + Rollup prod), **esbuild/SWC** ultra-fast compiler, **Rollup** library-র জন্য, **Turbopack** Rust-powered Webpack successor (Next.js)। **Tree shaking** dead code কমায় (ESM চাই), **code splitting** + **lazy loading** initial bundle ছোট রাখে, **HMR** dev experience ভালো করে। Production-এ **bundle analysis**, **source map strategy**, **long-term caching** অপরিহার্য। Module Federation দিয়ে micro-frontend। বাংলাদেশী 3G ইউজারের জন্য initial bundle <200KB gz রাখা target।

> 💡 মনে রাখুন: **"Bundle যত ছোট, ইউজার তত খুশি। যা লাগবে না, তা পাঠাবেন না।"**
