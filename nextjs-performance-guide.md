# Next.js Performance Optimization Guide

Complete practical guide to Next.js 15+ performance optimization, Lighthouse scoring, and production-ready techniques.

## Table of Contents
1. [Next.js Built-in Optimizations](#nextjs-built-in-optimizations)
2. [Image and Asset Optimization](#image-and-asset-optimization)
3. [Rendering Strategies](#rendering-strategies)
4. [Code Splitting & Bundle Optimization](#code-splitting--bundle-optimization)
5. [Core Web Vitals](#core-web-vitals)
6. [Font and CSS Optimization](#font-and-css-optimization)
7. [API Routes Performance](#api-routes-performance)
8. [Lighthouse CI Setup](#lighthouse-ci-setup)
9. [Performance Monitoring](#performance-monitoring)
10. [Production Optimizations](#production-optimizations)

## Next.js Built-in Optimizations

### Automatic Code Splitting
```jsx
// Next.js automatically splits code at the page level
// pages/dashboard.js
export default function Dashboard() {
  return <div>Dashboard content</div>;
}

// Dynamic imports for components
import dynamic from 'next/dynamic';

const DynamicComponent = dynamic(() => import('../components/HeavyComponent'), {
  loading: () => <p>Loading...</p>,
  ssr: false // Disable SSR for client-only components
});

function MyPage() {
  return (
    <div>
      <h1>My Page</h1>
      <DynamicComponent />
    </div>
  );
}
```

### Image Optimization with next/image
```jsx
import Image from 'next/image';

// Basic optimized image
function ProfileImage() {
  return (
    <Image
      src="/profile.jpg"
      alt="Profile"
      width={300}
      height={300}
      priority // Load above-the-fold images eagerly
    />
  );
}

// Responsive images
function ResponsiveImage() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero"
      fill
      style={{ objectFit: 'cover' }}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    />
  );
}

// External images with domains config
// next.config.js
module.exports = {
  images: {
    domains: ['example.com', 'cdn.example.com'],
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384]
  }
};
```

### Font Optimization
```jsx
// app/layout.js (App Router)
import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap', // Prevents invisible text during font swap
});

const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  display: 'swap',
});

export default function RootLayout({ children }) {
  return (
    <html lang="en" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}

// Custom fonts
import localFont from 'next/font/local';

const customFont = localFont({
  src: './custom-font.woff2',
  display: 'swap',
  fallback: ['Arial', 'sans-serif']
});
```

## Image and Asset Optimization

### Advanced Image Strategies
```jsx
// Lazy loading with intersection observer
import { useState, useRef, useEffect } from 'react';
import Image from 'next/image';

function LazyImage({ src, alt, ...props }) {
  const [isLoaded, setIsLoaded] = useState(false);
  const [isInView, setIsInView] = useState(false);
  const imgRef = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsInView(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, []);

  return (
    <div ref={imgRef}>
      {isInView && (
        <Image
          src={src}
          alt={alt}
          onLoad={() => setIsLoaded(true)}
          className={isLoaded ? 'opacity-100' : 'opacity-0'}
          {...props}
        />
      )}
    </div>
  );
}

// Image placeholder with blur
import placeholderImage from '../public/placeholder.jpg';

function BlurImage() {
  return (
    <Image
      src="/high-res-image.jpg"
      alt="Content"
      width={800}
      height={600}
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQ..." // Base64 blur
      // or
      placeholder="blur"
      blurDataURL={placeholderImage}
    />
  );
}
```

### Static Asset Optimization
```javascript
// next.config.js
module.exports = {
  compress: true, // Enable gzip compression

  webpack(config) {
    // Optimize SVGs
    config.module.rules.push({
      test: /\.svg$/,
      use: ['@svgr/webpack']
    });

    // Bundle analyzer (dev only)
    if (process.env.ANALYZE) {
      const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
      config.plugins.push(
        new BundleAnalyzerPlugin({
          analyzerMode: 'server',
          openAnalyzer: true
        })
      );
    }

    return config;
  },

  // Static file compression
  experimental: {
    optimizeCss: true,
    optimizePackageImports: ['lucide-react', '@radix-ui/react-icons']
  }
};
```

## Rendering Strategies

### Server-Side Rendering (SSR)
```jsx
// app/posts/page.js (App Router)
async function PostsPage() {
  // This runs on the server for each request
  const posts = await fetch('https://api.example.com/posts', {
    cache: 'no-store' // Always fetch fresh data
  }).then(res => res.json());

  return (
    <div>
      <h1>Latest Posts</h1>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </div>
  );
}

// pages/posts.js (Pages Router)
export async function getServerSideProps(context) {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());

  return {
    props: {
      posts,
    },
  };
}

function PostsPage({ posts }) {
  return (
    <div>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
        </article>
      ))}
    </div>
  );
}
```

### Static Site Generation (SSG)
```jsx
// app/blog/[slug]/page.js (App Router)
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());

  return posts.map((post) => ({
    slug: post.slug,
  }));
}

async function BlogPost({ params }) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`, {
    cache: 'force-cache' // Static generation
  }).then(res => res.json());

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}

// pages/blog/[slug].js (Pages Router)
export async function getStaticPaths() {
  const posts = await fetch('https://api.example.com/posts').then(res => res.json());

  const paths = posts.map((post) => ({
    params: { slug: post.slug },
  }));

  return { paths, fallback: 'blocking' };
}

export async function getStaticProps({ params }) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`).then(res => res.json());

  return {
    props: { post },
    revalidate: 3600, // Revalidate every hour
  };
}
```

### Incremental Static Regeneration (ISR)
```jsx
// app/products/page.js (App Router)
async function ProductsPage() {
  const products = await fetch('https://api.example.com/products', {
    next: { revalidate: 3600 } // Revalidate every hour
  }).then(res => res.json());

  return (
    <div>
      {products.map(product => (
        <div key={product.id}>
          <h2>{product.name}</h2>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  );
}

// pages/products.js (Pages Router)
export async function getStaticProps() {
  const products = await fetch('https://api.example.com/products').then(res => res.json());

  return {
    props: { products },
    revalidate: 3600, // ISR: regenerate at most once per hour
  };
}
```

## Code Splitting & Bundle Optimization

### Dynamic Imports
```jsx
import { useState } from 'react';
import dynamic from 'next/dynamic';

// Component-level splitting
const Chart = dynamic(() => import('react-chartjs-2'), {
  loading: () => <div>Loading chart...</div>,
  ssr: false
});

const Modal = dynamic(() => import('../components/Modal'));

// Library splitting
function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  const [chartLib, setChartLib] = useState(null);

  const loadChart = async () => {
    const { Chart } = await import('chart.js/auto');
    setChartLib(Chart);
    setShowChart(true);
  };

  return (
    <div>
      <button onClick={loadChart}>Load Chart</button>
      {showChart && chartLib && <Chart />}
    </div>
  );
}

// Conditional component loading
const AdminPanel = dynamic(
  () => import('../components/AdminPanel'),
  {
    loading: () => <div>Loading admin panel...</div>
  }
);

function App({ user }) {
  return (
    <div>
      {user.isAdmin && <AdminPanel />}
    </div>
  );
}
```

### Bundle Analysis
```json
// package.json
{
  "scripts": {
    "analyze": "ANALYZE=true next build",
    "build": "next build",
    "build:analyze": "npm run build && npm run analyze"
  }
}
```

```javascript
// Bundle optimization configuration
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: [
      'lodash-es',
      '@mui/material',
      '@mui/icons-material',
      'date-fns',
      'ramda'
    ]
  },

  webpack(config, { isServer }) {
    // Optimize bundle splitting
    if (!isServer) {
      config.optimization.splitChunks = {
        ...config.optimization.splitChunks,
        cacheGroups: {
          ...config.optimization.splitChunks.cacheGroups,
          commons: {
            name: 'commons',
            chunks: 'all',
            minChunks: 2,
            maxSize: 244000,
          },
          vendor: {
            test: /[\/]node_modules[\/]/,
            name: 'vendors',
            chunks: 'all',
            maxSize: 244000,
          }
        }
      };
    }

    return config;
  }
};
```

## Core Web Vitals

### Largest Contentful Paint (LCP)
```jsx
// Optimize above-the-fold content
import Image from 'next/image';

function HeroSection() {
  return (
    <section>
      {/* Priority loading for hero images */}
      <Image
        src="/hero.jpg"
        alt="Hero"
        width={1200}
        height={600}
        priority
        style={{ width: '100%', height: 'auto' }}
      />
      <h1>Welcome to our site</h1>
    </section>
  );
}

// Preload critical resources
// app/layout.js
export default function RootLayout({ children }) {
  return (
    <html>
      <head>
        <link rel="preload" href="/hero.jpg" as="image" />
        <link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossOrigin="" />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

### First Input Delay (FID) / Interaction to Next Paint (INP)
```jsx
import { useTransition, startTransition } from 'react';

function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value) => {
    setQuery(value); // Urgent update

    startTransition(() => {
      // Non-urgent expensive operation
      const filtered = performHeavySearch(value);
      setResults(filtered);
    });
  };

  return (
    <div>
      <input 
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      {isPending && <div>Searching...</div>}
      <SearchResults results={results} />
    </div>
  );
}

// Web Workers for heavy computations
// utils/worker.js
export function createWorker() {
  return new Worker(new URL('./heavy-computation.worker.js', import.meta.url));
}

// heavy-computation.worker.js
self.onmessage = function(e) {
  const { data, operation } = e.data;

  let result;
  switch (operation) {
    case 'sort':
      result = data.sort((a, b) => a.value - b.value);
      break;
    case 'filter':
      result = data.filter(item => item.active);
      break;
  }

  self.postMessage(result);
};
```

### Cumulative Layout Shift (CLS)
```jsx
// Reserve space for dynamic content
function ArticleCard({ article }) {
  return (
    <div className="article-card">
      {/* Fixed aspect ratio prevents layout shift */}
      <div style={{ aspectRatio: '16/9', position: 'relative' }}>
        <Image
          src={article.image}
          alt={article.title}
          fill
          style={{ objectFit: 'cover' }}
        />
      </div>
      <h2>{article.title}</h2>
      <p>{article.excerpt}</p>
    </div>
  );
}

// Skeleton loaders
function SkeletonLoader() {
  return (
    <div className="animate-pulse">
      <div className="h-48 bg-gray-300 rounded mb-4"></div>
      <div className="h-4 bg-gray-300 rounded mb-2"></div>
      <div className="h-4 bg-gray-300 rounded w-3/4"></div>
    </div>
  );
}

function ArticleList() {
  const [articles, setArticles] = useState(null);

  useEffect(() => {
    fetchArticles().then(setArticles);
  }, []);

  if (!articles) {
    return (
      <div>
        {[...Array(6)].map((_, i) => (
          <SkeletonLoader key={i} />
        ))}
      </div>
    );
  }

  return (
    <div>
      {articles.map(article => (
        <ArticleCard key={article.id} article={article} />
      ))}
    </div>
  );
}
```

## Font and CSS Optimization

### Font Loading Strategies
```jsx
// app/layout.js
import { Inter, Playfair_Display } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  preload: true,
});

const playfair = Playfair_Display({
  subsets: ['latin'],
  display: 'swap',
  weight: ['400', '700'],
});

// Font with fallback system
const roboto = Inter({
  subsets: ['latin'],
  display: 'swap',
  fallback: ['system-ui', 'arial'],
});
```

### CSS Optimization
```javascript
// next.config.js
module.exports = {
  experimental: {
    optimizeCss: true, // Enable CSS optimization
  },

  webpack(config) {
    // Critical CSS extraction
    if (!config.isServer) {
      config.optimization.splitChunks.cacheGroups.styles = {
        name: 'styles',
        test: /\.(css|scss)$/,
        chunks: 'all',
        enforce: true,
      };
    }

    return config;
  }
};
```

```css
/* Critical CSS - inline in head */
.hero { 
  height: 100vh; 
  display: flex; 
  align-items: center; 
}

/* Non-critical CSS - load async */
@media (max-width: 768px) {
  .hero { height: 50vh; }
}
```

## API Routes Performance

### Optimized API Routes
```javascript
// pages/api/posts.js or app/api/posts/route.js
import { NextResponse } from 'next/server';

// Enable edge runtime for better performance
export const runtime = 'edge';

export async function GET(request) {
  try {
    // Add caching headers
    const posts = await fetchPosts();

    return NextResponse.json(posts, {
      status: 200,
      headers: {
        'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
        'CDN-Cache-Control': 'public, s-maxage=3600',
      },
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch posts' },
      { status: 500 }
    );
  }
}

// Database connection pooling
// lib/db.js
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20, // Maximum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

export { pool };

// API route with connection pooling
export async function GET() {
  const client = await pool.connect();

  try {
    const result = await client.query('SELECT * FROM posts LIMIT 10');
    return NextResponse.json(result.rows);
  } finally {
    client.release();
  }
}
```

### Middleware Performance
```javascript
// middleware.js
import { NextResponse } from 'next/server';

export function middleware(request) {
  // Add security headers
  const response = NextResponse.next();

  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-XSS-Protection', '1; mode=block');

  return response;
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
};
```

## Lighthouse CI Setup

### GitHub Actions Configuration
```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build Next.js app
        run: npm run build

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          configPath: './lighthouserc.json'
          uploadArtifacts: true
          temporaryPublicStorage: true

      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: lighthouse-results
          path: .lighthouseci
```

### Lighthouse Configuration
```json
// lighthouserc.json
{
  "ci": {
    "collect": {
      "url": [
        "http://localhost:3000",
        "http://localhost:3000/about",
        "http://localhost:3000/blog"
      ],
      "startServerCommand": "npm start",
      "numberOfRuns": 3,
      "settings": {
        "preset": "desktop",
        "chromeFlags": "--no-sandbox --disable-dev-shm-usage",
        "formFactor": "desktop",
        "throttling": {
          "rttMs": 40,
          "throughputKbps": 10240,
          "cpuSlowdownMultiplier": 1
        }
      }
    },
    "assert": {
      "preset": "lighthouse:recommended",
      "assertions": {
        "categories:performance": ["error", {"minScore": 0.9}],
        "categories:accessibility": ["error", {"minScore": 0.9}],
        "categories:best-practices": ["error", {"minScore": 0.9}],
        "categories:seo": ["error", {"minScore": 0.9}],
        "first-contentful-paint": ["warn", {"maxNumericValue": 2000}],
        "largest-contentful-paint": ["error", {"maxNumericValue": 2500}],
        "cumulative-layout-shift": ["error", {"maxNumericValue": 0.1}],
        "total-blocking-time": ["error", {"maxNumericValue": 300}]
      }
    },
    "upload": {
      "target": "temporary-public-storage"
    }
  }
}
```

## Performance Monitoring

### Web Vitals Tracking
```javascript
// lib/vitals.js
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function sendToAnalytics({ name, value, id }) {
  // Send to Google Analytics
  if (typeof gtag !== 'undefined') {
    gtag('event', name, {
      event_category: 'Web Vitals',
      event_label: id,
      value: Math.round(name === 'CLS' ? value * 1000 : value),
      non_interaction: true,
    });
  }

  // Send to custom endpoint
  fetch('/api/vitals', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name, value, id, url: window.location.href }),
  });
}

export function reportWebVitals() {
  getCLS(sendToAnalytics);
  getFID(sendToAnalytics);
  getFCP(sendToAnalytics);
  getLCP(sendToAnalytics);
  getTTFB(sendToAnalytics);
}

// pages/_app.js
import { reportWebVitals } from '../lib/vitals';

export function reportWebVitals(metric) {
  reportWebVitals(metric);
}

// app/layout.js (App Router)
'use client';
import { useReportWebVitals } from 'next/web-vitals';

export default function Layout({ children }) {
  useReportWebVitals((metric) => {
    console.log(metric);
    // Send to analytics
  });

  return children;
}
```

### Performance API Usage
```javascript
// lib/performance.js
export function measurePageLoad() {
  if (typeof window !== 'undefined' && 'performance' in window) {
    const navigation = performance.getEntriesByType('navigation')[0];

    return {
      domContentLoaded: navigation.domContentLoadedEventEnd - navigation.domContentLoadedEventStart,
      loadComplete: navigation.loadEventEnd - navigation.loadEventStart,
      pageLoad: navigation.loadEventEnd - navigation.fetchStart,
      ttfb: navigation.responseStart - navigation.fetchStart,
    };
  }
  return null;
}

// Custom performance observer
export function observeLongTasks() {
  if ('PerformanceObserver' in window) {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > 50) {
          console.warn('Long task detected:', entry);
          // Send to monitoring service
        }
      }
    });

    observer.observe({ entryTypes: ['longtask'] });
  }
}
```

## Production Optimizations

### Advanced Next.js Configuration
```javascript
// next.config.js
const nextConfig = {
  // Enable SWC minification
  swcMinify: true,

  // Compression
  compress: true,

  // Image optimization
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    minimumCacheTTL: 31536000, // 1 year
  },

  // Headers for security and performance
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-XSS-Protection',
            value: '1; mode=block',
          },
        ],
      },
      {
        source: '/api/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=300, s-maxage=300',
          },
        ],
      },
    ];
  },

  // Redirects for SEO
  async redirects() {
    return [
      {
        source: '/old-page',
        destination: '/new-page',
        permanent: true,
      },
    ];
  },

  // Experimental features
  experimental: {
    optimizeCss: true,
    optimizePackageImports: ['lucide-react', '@radix-ui/react-icons'],
    turbo: {
      rules: {
        '*.svg': {
          loaders: ['@svgr/webpack'],
          as: '*.js',
        },
      },
    },
  },

  // Webpack optimizations
  webpack(config, { isServer, dev }) {
    // Production optimizations
    if (!dev && !isServer) {
      config.optimization.splitChunks.cacheGroups.commons = {
        name: 'commons',
        chunks: 'all',
        minChunks: 2,
        maxSize: 244000,
      };
    }

    return config;
  },
};

module.exports = nextConfig;
```

### CDN and Caching Strategy
```javascript
// next.config.js
module.exports = {
  assetPrefix: process.env.NODE_ENV === 'production' ? 'https://cdn.example.com' : '',

  async headers() {
    return [
      {
        source: '/api/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=300, stale-while-revalidate=600',
          },
        ],
      },
      {
        source: '/_next/static/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
    ];
  },
};
```

## Performance Checklists

### Development Checklist
- [ ] Use Next.js Image component for all images
- [ ] Implement proper font loading with next/font
- [ ] Add dynamic imports for heavy components
- [ ] Configure proper rendering strategy (SSG/SSR/ISR)
- [ ] Optimize bundle with tree shaking
- [ ] Use React 18 concurrent features
- [ ] Implement proper loading states
- [ ] Add skeleton screens for better perceived performance

### Testing Checklist
- [ ] Achieve 90+ Lighthouse scores on all core pages
- [ ] Test Core Web Vitals with real user data
- [ ] Analyze bundle size with webpack-bundle-analyzer
- [ ] Test on slow networks and devices
- [ ] Validate image optimization and lazy loading
- [ ] Check font loading and FOIT/FOUT
- [ ] Verify API route performance
- [ ] Test caching strategies

### Production Checklist
- [ ] Set up Lighthouse CI for continuous monitoring
- [ ] Configure CDN for static assets
- [ ] Implement proper cache headers
- [ ] Set up Web Vitals monitoring
- [ ] Enable compression (Brotli/Gzip)
- [ ] Configure security headers
- [ ] Set up error monitoring (Sentry, LogRocket)
- [ ] Monitor bundle size in CI/CD

## Performance Targets

### Bundle Sizes (Next.js 15)
- First Load JS: < 200kb (gzipped)
- Route JS: < 50kb per route (gzipped)  
- CSS: < 50kb total (gzipped)
- Images: WebP/AVIF with proper sizing

### Core Web Vitals Targets
- LCP: < 2.5s (mobile), < 1.8s (desktop)
- FID/INP: < 100ms
- CLS: < 0.1
- TTFB: < 600ms
- Speed Index: < 3.4s

### Build Performance (Next.js 15)
- Initial Build: ~65% faster than v14
- Hot Module Replacement: < 50ms
- Cold Start: < 120ms
- Memory Usage: ~30% less than v14

## Tools and Resources

### Essential Tools
- Next.js DevTools
- Lighthouse CI
- webpack-bundle-analyzer  
- next-bundle-analyzer
- web-vitals
- @next/bundle-analyzer

### Monitoring Services
- Vercel Analytics
- Google PageSpeed Insights
- WebPageTest
- GTmetrix
- Pingdom
- New Relic Browser

### Performance Libraries
- next/image (built-in)
- next/font (built-in)  
- next/dynamic (built-in)
- react-window (virtualization)
- intersection-observer (lazy loading)

## Migration Guide (Next.js 14 → 15)

### Key Changes
```bash
# Update to Next.js 15
npm install next@15 react@latest react-dom@latest

# Update TypeScript config
npm install -D @types/react@latest @types/react-dom@latest
```

### Performance Improvements in v15
- 65% faster build times with Turbopack
- 67% faster Hot Module Replacement
- 30% reduced memory usage
- 60% faster server component hydration
- Enhanced image optimization
- Better tree shaking

---

## Final Notes

**Remember: Measure → Optimize → Verify**

1. **Start with Next.js defaults** - They're already optimized
2. **Use built-in components** - Image, Font, Dynamic imports
3. **Choose the right rendering strategy** - SSG > ISR > SSR > CSR  
4. **Monitor continuously** - Set up Lighthouse CI and Web Vitals tracking
5. **Test on real devices** - Performance varies significantly across devices

Next.js provides excellent defaults, but these optimizations will get you to 95+ Lighthouse scores consistently.

---

*Complete Next.js 15+ Performance Optimization Guide - Save as nextjs-performance-guide-2025.md*
